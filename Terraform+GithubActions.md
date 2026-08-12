# Terraform + CI/CD Integration — GitHub Actions + Terraform

GitHub Actions is a very natural fit for Terraform since your code likely already lives in GitHub — you get triggers, secrets management, and PR integration all built into the same platform, no separate CI server to run.

## Why GitHub Actions for Terraform?
Aspect	Benefit
Native to GitHub	No separate server to maintain (unlike Jenkins)
YAML-based workflows	Version-controlled alongside your .tf code, in .github/workflows/
Built-in secrets store	GitHub Secrets for cloud credentials, no separate credentials plugin
PR integration	Official hashicorp/setup-terraform action can auto-comment plan output directly on PRs
Environments feature	Built-in manual approval gates via GitHub Environments
Marketplace actions	Official HashiCorp actions maintained directly by Terraform's creators

## High-Level Architecture

    Push / Pull Request to GitHub repo
            │
            ▼
    GitHub Actions workflow triggered
            │
            ├─► terraform fmt -check
            ├─► terraform init
            ├─► terraform validate
            ├─► terraform plan   → posted as PR comment
            ├─► (Environment approval gate, for prod)
            └─► terraform apply  (only on merge to main)
            │
            ▼
    Remote backend (S3 + DynamoDB, or Terraform Cloud)

### Step 1: Prerequisites

1. GitHub repository with your .tf files

2. Remote state backend already configured (S3 + DynamoDB, Azure Storage, GCS, or Terraform Cloud) — same reasoning as Jenkins: GitHub Actions runners are ephemeral, so state cannot live locally

3. Cloud credentials stored as GitHub Secrets

4. Workflow files live in .github/workflows/*.yml

### Step 2: Store Credentials as GitHub Secrets

Go to your repo → Settings → Secrets and variables → Actions → New repository secret

Add:
    * AWS_ACCESS_KEY_ID
    * AWS_SECRET_ACCESS_KEY
(or better, see the OIDC section below — no long-lived keys at all)

These are encrypted at rest, masked in logs automatically, and only injected into the workflow run as environment variables.

### Step 3: Basic Workflow File

**Create .github/workflows/terraform.yml:**

```
yaml
name: 'Terraform CI/CD'

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: ap-south-1

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Init
        run: terraform init -input=false

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan -input=false -out=tfplan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -input=false tfplan
```

### What each step does
Step	Purpose
actions/checkout@v4	Pulls your repo's code into the runner
hashicorp/setup-terraform@v3	Official action — installs the specified Terraform CLI version
fmt -check	Fails if code isn't properly formatted
init	Downloads providers, connects to remote backend
validate	Catches syntax errors
plan	Computes and saves the plan
apply (conditional)	Only runs on a push to main (not on PRs) — keeps PRs plan-only

**Key GitHub Actions concept:** if: github.ref == 'refs/heads/main' && github.event_name == 'push' — this condition ensures apply never runs on a pull request, only after code is actually merged to main.

### Step 4: Posting Plan Output as a PR Comment

This is one of GitHub Actions' most valuable Terraform patterns — reviewers see the infrastructure impact directly in the PR, without digging through logs.

```
yaml
name: 'Terraform PR Plan'

on:
  pull_request:
    branches: [main]

env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  AWS_DEFAULT_REGION: ap-south-1

permissions:
  contents: read
  pull-requests: write   # needed to post PR comments

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - name: Terraform Init
        run: terraform init -input=false

      - name: Terraform Plan
        id: plan
        run: terraform plan -input=false -no-color -out=tfplan
        continue-on-error: true

      - name: Show Plan Output
        id: show
        run: terraform show -no-color tfplan > plan_output.txt

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan_output.txt', 'utf8');
            const maxLength = 65000;
            const truncated = plan.length > maxLength
              ? plan.substring(0, maxLength) + "\n... (truncated)"
              : plan;

            const output = `#### Terraform Plan 📖
            <details><summary>Show Plan</summary>

            \`\`\`terraform
            ${truncated}
            \`\`\`

            </details>`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

      - name: Fail if Plan Failed
        if: steps.plan.outcome == 'failure'
        run: exit 1
```

**Result:** every PR that touches .tf files automatically gets a bot comment showing exactly what infrastructure will change — reviewers approve/reject with full visibility, just like reviewing code diffs.

### Step 5: Manual Approval Gate Using GitHub Environments

GitHub has a native feature called Environments that supports required reviewers — this is GitHub Actions' equivalent of Jenkins' input step.

**Setup (in GitHub UI):**

1. Repo → Settings → Environments → New environment → name it production

2. Under Deployment protection rules, enable Required reviewers and add specific people/teams who must approve

3. (Optional) Restrict which branches can deploy to this environment

**Using it in the workflow:**
```
yaml
jobs:
  terraform-apply:
    runs-on: ubuntu-latest
    environment: production   # ← this triggers the approval gate

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - name: Terraform Init
        run: terraform init -input=false

      - name: Terraform Plan
        run: terraform plan -input=false -out=tfplan

      - name: Terraform Apply
        run: terraform apply -input=false tfplan
```

When this job runs, GitHub Actions pauses and shows "Waiting for approval" in the Actions UI. A designated reviewer must click Approve before the apply step executes — functionally identical to Jenkins' manual gate, but built into GitHub natively with no plugin needed.

### Step 6: OIDC — Passwordless AWS Authentication (Best Practice)

Storing long-lived AWS_ACCESS_KEY_ID/AWS_SECRET_ACCESS_KEY as GitHub Secrets works, but it's not ideal — those keys don't expire and are a standing risk if ever leaked. The modern best practice is OpenID Connect (OIDC): GitHub Actions authenticates to AWS using short-lived, auto-rotating tokens, with no stored secrets at all.

**Setup on the AWS side:**

1. Create an IAM OIDC identity provider trusting token.actions.githubusercontent.com

2. Create an IAM role with a trust policy scoped to your specific GitHub repo/branch

3. Attach only the permissions Terraform actually needs (least privilege)

**Workflow using OIDC:**

```
yaml
name: 'Terraform with OIDC'

on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - run: terraform init -input=false
      - run: terraform plan -input=false -out=tfplan
      - run: terraform apply -input=false tfplan
```

No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY anywhere — GitHub requests a short-lived token from AWS at runtime, scoped precisely to this workflow run, and it expires automatically. This is now the recommended approach for cloud-connected GitHub Actions pipelines.

### Step 7: Multi-Environment Workflow (Dev / Staging / Prod)

```yaml
name: 'Terraform Multi-Env'

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.5

      - name: Terraform Init
        run: terraform init -input=false -backend-config=${{ github.event.inputs.environment }}-backend.hcl

      - name: Terraform Plan
        run: terraform plan -input=false -var-file=${{ github.event.inputs.environment }}.tfvars -out=tfplan

      - name: Terraform Apply
        run: terraform apply -input=false tfplan
```

workflow_dispatch with a choice input adds a manual trigger button in the GitHub Actions UI, where you pick dev/staging/prod from a dropdown before running — and because environment: is set dynamically from that input, GitHub automatically applies the right approval rules (e.g., prod environment requiring reviewers, dev not requiring any).

### Step 8: Adding Security Scanning (tfsec / Checkov)

```yaml
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: .
```
or

```yaml
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
```

These scan your .tf files for common misconfigurations (open security groups, unencrypted storage, overly-permissive IAM policies) before anything gets applied — catching security issues at PR time rather than after deployment.

### Step 9: Full Production-Grade Workflow (Combining Everything)

```yaml
name: 'Terraform CI/CD'

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

env:
  TF_VERSION: 1.7.5

jobs:
  plan:
    name: 'Plan'
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Format Check
        run: terraform fmt -check

      - name: Init
        run: terraform init -input=false

      - name: Validate
        run: terraform validate

      - name: Security Scan
        uses: aquasecurity/tfsec-action@v1.0.3

      - name: Plan
        run: terraform plan -input=false -no-color -out=tfplan | tee plan_output.txt

      - name: Comment Plan on PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan_output.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `#### Terraform Plan 📖\n<details><summary>Show Plan</summary>\n\n\`\`\`\n${plan}\n\`\`\`\n</details>`
            });

  apply:
    name: 'Apply'
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform-role
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Init
        run: terraform init -input=false

      - name: Apply
        run: terraform apply -input=false -auto-approve
```

**Flow this produces:**

1. Open a PR → plan job runs → plan posted as PR comment → team reviews

2. Merge PR to main → apply job runs → pauses for production environment approval (if reviewers are configured) → applies

## Jenkins vs GitHub Actions — Quick Comparison
|Aspect	|Jenkins	|GitHub Actions|
|--------|---------|--------------|
|Hosting    |	Self-hosted server to maintain	|   Fully managed by GitHub |
|Config format  |	Groovy (Jenkinsfile)    |	YAML|
|Secrets	|   Credentials plugin	|   Native GitHub Secrets|
|Approval gates |	input step	|   GitHub Environments + required reviewers|
|PR comments	|   Needs custom scripting/plugin   |	actions/github-script — straightforward|
|OIDC cloud auth |	Possible but more setup |  	First-class support (aws-actions/configure-aws-credentials)|
|Best for	|   Teams with existing Jenkins infra, complex custom pipelines |	Teams already on GitHub, wanting less infra to maintain|

## Best Practices Recap (GitHub Actions specific)

1. Use OIDC instead of static AWS keys — no long-lived secrets to leak or rotate.

2. Split plan (on PR) and apply (on merge to main) into separate jobs/triggers — never auto-apply from a PR branch.

3. Use GitHub Environments with required reviewers for production — don't rely on -auto-approve without a human gate.

4. Always comment plan output on PRs — this is GitHub Actions' single biggest ergonomic win over many other CI tools for Terraform.

5. Pin the Terraform version via hashicorp/setup-terraform and commit .terraform.lock.hcl.

6. Add a security scan step (tfsec/checkov) before apply.

7. Use remote state with locking (S3 + DynamoDB, or Terraform Cloud) — GitHub-hosted runners are ephemeral and stateless between runs.

8. Scope IAM roles tightly per environment — don't let the dev pipeline's role have prod permissions.