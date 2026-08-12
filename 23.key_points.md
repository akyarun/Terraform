# 1. Terraform State Deep Dive

## State locking (recap + what happens under the hood)

You already know locking prevents concurrent writes (S3 native locking or DynamoDB, from earlier). What's worth adding here is the lock lifecycle and failure modes:

    * A lock is acquired before any state-reading operation that could write back — plan, apply, destroy, state mv/rm, import, even refresh. plan -lock=false exists but should be treated as a debugging escape hatch, not routine use.

    * If a process crashes mid-operation (killed CI job, network partition, laptop sleep), the lock is orphaned — nobody else can proceed until it's cleared.

    * terraform force-unlock <LOCK_ID> clears it manually. This is dangerous if used carelessly: if the original process is actually still running (not dead, just slow), force-unlocking creates exactly the concurrent-write race locking exists to prevent. Always confirm the original operation is truly dead (check CI job status, ask the teammate) before force-unlocking.

### State corruption handling

State corruption typically looks like: terraform plan throwing JSON parse errors, resources listed in state that don't match terraform show output, or a state file that's truncated/empty after a failed write.

**Common causes:**

    * A process killed mid-write (before the backend finished persisting the new state)

    * Manual hand-editing of the state file gone wrong

    * Two concurrent writers slipping past a broken/misconfigured lock

    * A backend outage during an in-flight apply

**Diagnosis:**

```
    terraform validate          # checks config syntax, not state
    terraform state list        # if this fails outright, state is likely malformed JSON
    terraform show -json | jq . # will error clearly on invalid JSON
```

**Handling it, in order of preference:**

1. **Pull the last-known-good version from backend versioning.** This is the reason S3 bucket versioning is non-negotiable (from your remote-state question) — list object versions, restore the previous good one.

```
   aws s3api list-object-versions --bucket mycompany-terraform-state --prefix prod/networking/terraform.tfstate
   aws s3api get-object --bucket mycompany-terraform-state --key prod/networking/terraform.tfstate --version-id <VERSION_ID> terraform.tfstate.recovered
```

2. **Use the local backup file.** Terraform automatically writes terraform.tfstate.backup (the state as it was before the last successful operation) even with a remote backend during certain operations — check for this before assuming total loss.

3. **Manual JSON surgery, as an absolute last resort.** The state file's JSON schema is documented but not meant for hand-editing; a single misplaced bracket or wrong resource_type string can make things worse. If you must, edit a copy, validate with terraform show -json , and never edit the file that's actively the source of truth for the backend.

### State recovery

Beyond restoring from a backup, two other recovery patterns matter:

**Recovering a single resource's tracking (not the whole file)** — if one resource's state entry got corrupted or accidentally removed but the resource is still healthy:

```
    terraform state rm aws_instance.web        # stop tracking (doesn't destroy real infra)
    terraform import aws_instance.web i-0abc123def456789   # re-attach it
```

This rm + import combo is the standard fix-a-single-resource pattern — the import command connects here directly to Topic 4 below.

**Recovering from total state loss** — if there's genuinely no backup/version available: terraform import every resource, one at a time, rebuilding state from scratch against real infrastructure. Painful, but it's why versioned backends matter enough to be treated as non-negotiable rather than optional.

**Backend strategies**

Beyond the S3 + native-locking pattern already covered, the broader landscape of backend choices:

|Backend	|   Locking	    |   Best for    |
|-------------|---------------|----------------|
|s3	    |Native (use_lockfile) or DynamoDB  |	AWS-centric teams, full control |
|azurerm    |	Native (blob lease) |	Azure-centric teams |
|gcs	    |Native	|   GCP-centric teams   |
|remote (Terraform Cloud/Enterprise)    |	Built-in, plus run queueing |	Teams wanting managed state + policy + UI, not just storage |
|local  |	None	   | Solo experimentation only — never for team/production use  |

* Terraform Cloud's "remote" backend deserves a mention since it's qualitatively different from the others: state storage is bundled with a full run pipeline (plan → policy check → approval → apply), so "backend" there also means "where runs execute," not just "where state lives." This is a meaningfully different operating model from self-managed S3 + local apply.

* Migrating between backends — change the backend block and re-run init; Terraform detects the change and offers to copy existing state into the new backend:

```
terraform init -migrate-state
```

Always back up the state manually before doing this on anything production — -migrate-state is generally safe but this is exactly the kind of operation you want a rollback path for.

## 2. Security in Terraform

**Secrets management**

Never put secrets directly in .tf files or .tfvars committed to version control — this is the single most common Terraform security mistake. Instead:

```yaml
variable "db_password" {
  type      = string
  sensitive = true
}
```
```
export TF_VAR_db_password="$(aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text)"
terraform apply
```

Or read the secret directly at plan/apply time via a data source instead of an environment variable at all:

```yaml
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)["password"]
}
```

This pattern (Secrets Manager, SSM Parameter Store, or the HashiCorp Vault provider) means the secret's actual value never lives in your .tf files or shell history — only a reference to where it's stored does.

* Critical caveat: secrets still end up in the state file, in plaintext, regardless of which method feeds them in. State encryption at rest (SSE on the S3 bucket) and tight IAM access to the state bucket are therefore just as important a secrets-protection boundary as how the secret entered your config.

**Sensitive variables**

sensitive = true on a variable or output block redacts the value from CLI output (plan/apply show (sensitive value) instead of the real value) and from logs — but it does not encrypt it in state, and does not stop a person with state show access or state-file read access from seeing it in plaintext.

```yaml
variable "api_key" {
  type      = string
  sensitive = true
}

output "connection_string" {
  value     = "postgres://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}"
  sensitive = true
}
```

If a sensitive value is embedded in a larger non-sensitive expression (like the connection string above interpolating a sensitive password), Terraform automatically propagates the sensitivity to the whole output — you don't need to mark every intermediate piece. nonsensitive() exists to explicitly strip the marking when you're certain a specific value is safe to display, but should be used sparingly and deliberately.

**IAM least privilege**

* Terraform typically needs broad permissions to do its job (creating/modifying/destroying whatever resource types your config touches), which makes it an attractive target if the credentials leak. Practical mitigations:

* Separate plan and apply permissions where the platform supports it (Terraform Cloud/Enterprise natively support this: a "plan" run can use read-only credentials, only the "apply" step uses write credentials) — limits blast radius of a compromised plan-only pipeline.

* Scope IAM policies to only the resource types/actions your config actually manages, not *:*. Tools like iamlive or reviewing CloudTrail after a representative apply can help derive the minimal actual permission set.

* Use short-lived credentials via OIDC federation from CI (GitHub Actions/GitLab CI assuming an IAM role via OIDC) instead of long-lived static access keys stored as CI secrets — this was mentioned in the authentication section of your provider question, and is directly a least-privilege/blast-radius control.

* Permission boundaries — an IAM concept, not Terraform-specific, but commonly paired: attach a boundary policy to the role Terraform assumes so that even if the role's attached policy is misconfigured too broadly, the boundary caps what it can actually do.

* Separate roles per environment — the Terraform role that manages dev shouldn't be able to touch prod resources at all, ideally enforced by using genuinely separate AWS accounts per environment (tying back to the workspace/directory-per-environment discussion).

**tfsec scanning (and the broader static-analysis category)**

tfsec statically analyzes .tf files for security misconfigurations before anything is applied — public S3 buckets, overly permissive security groups, unencrypted resources, missing MFA requirements, etc.

```bash
tfsec .
```

    Result #1 HIGH Block public access to S3 buckets
    aws_s3_bucket.app_assets[0:0]
    1 │ resource "aws_s3_bucket" "app_assets" {
    Public access is not blocked. Consider aws_s3_bucket_public_access_block.

Note: HashiCorp archived tfsec in 2025, folding its rule engine into Trivy (the same maintainers, Aqua Security), so current guidance is to run trivy config . for the same class of checks — worth verifying against Trivy's current docs since tooling in this space moves fast. Other tools in the same category worth knowing:

* Checkov — broader multi-cloud/multi-IaC static analysis (Terraform, CloudFormation, Kubernetes manifests).

* TFLint — more general linting (unused variables, provider-specific best practices, deprecated syntax) rather than strictly security-focused.

* Terraform Cloud/Enterprise Sentinel or OPA policies — policy-as-code enforced at plan time in the platform itself, blocking non-compliant applies rather than just warning.

**CI integration pattern:**

```yaml
# example GitHub Actions step
- name: Security scan
  run: tfsec . --minimum-severity HIGH
```

Fail the pipeline on high/critical findings; treat lower-severity findings as warnings that don't block merge but get tracked.

## 3. Terraform Drift Detection

**What infrastructure drift is**

Drift is when the real state of infrastructure diverges from what's recorded in Terraform's state file — someone manually resized an instance in the console, a Lambda's environment variable got hotfixed directly, an autoscaler changed a value Terraform also manages. Terraform's model assumes it's the sole source of truth for anything it manages; drift breaks that assumption silently until the next plan.