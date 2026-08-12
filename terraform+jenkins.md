# Terraform + CI/CD Integration — Jenkins + Terraform

## Why put Terraform in a CI/CD pipeline at all?

|Manual terraform apply from laptop	     |   Terraform via Jenkins pipeline                |
|----------------------------------------|-------------------------------------------------|
|Depends on whoever's laptop, their local state/creds |	Consistent environment, every time  |
|No approval step enforced	    |   Can require manual approval before apply|
|No audit trail of who ran what	| Jenkins build logs record every plan/apply|
|Credentials often sit in ~/.aws/credentials    |	Credentials injected securely, scoped per pipeline|
|Easy to forget plan before apply   |	Pipeline enforces plan → review → apply order|
|No consistent versioning of Terraform/provider |	Pipeline pins exact tool versions|


## High-Level Architecture

    Developer pushes code / opens PR
            │
            ▼
    Git repo (GitHub/GitLab/Bitbucket)
            │  (webhook triggers)
            ▼
        Jenkins
            │
            ├─► terraform init
            ├─► terraform validate
            ├─► terraform plan   → saved as artifact, posted to PR/Slack
            ├─► (Manual approval gate, for prod)
            └─► terraform apply  → real infrastructure updated
            │
            ▼
    Remote backend (S3 + DynamoDB, or Terraform Cloud)
    stores/locks state

### Step 1: Prerequisites

1. Jenkins server with:
* Terraform binary installed (or installed dynamically per build)
* Git plugin, Pipeline plugin (usually default)
* Credentials plugin for storing cloud secrets securely

2. Cloud credentials (e.g., AWS IAM user/role) with least-privilege permissions for the resources Terraform will manage

3. Remote state backend already set up (e.g., S3 bucket + DynamoDB lock table) — critical for CI/CD, since each Jenkins run is a fresh environment with no local state memory

4. Git repository containing your .tf files

### Step 2: Set Up Remote State (Required for CI/CD)

Since Jenkins agents are usually ephemeral (fresh workspace each run, sometimes even fresh containers), you cannot rely on local terraform.tfstate — it would be lost between runs. Remote state is mandatory in practice.

```
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

This ensures every Jenkins build reads/writes the same shared state, with locking to prevent two builds from applying simultaneously.

### Step 3: Store Cloud Credentials Securely in Jenkins

Never hardcode AWS keys in your Jenkinsfile or .tf files. Use Jenkins' built-in Credentials Manager:

1. Go to Manage Jenkins → Credentials → Add Credentials
2. Choose "Secret text" (or "AWS Credentials" if the plugin is installed)
3. Add:
    * ID: aws-access-key-id
    * ID: aws-secret-access-key

You'll reference these by ID inside the Jenkinsfile — Jenkins injects them as environment variables at runtime, and masks them in logs.

### Step 4: Write the Jenkinsfile

This is the core of the integration — a declarative pipeline defining the stages.

```
groovy
pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'ap-south-1'
        TF_IN_AUTOMATION      = 'true'   // tells Terraform it's running non-interactively
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init -input=false'
            }
        }

        stage('Terraform Format Check') {
            steps {
                sh 'terraform fmt -check'
            }
        }

        stage('Terraform Validate') {
            steps {
                sh 'terraform validate'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh 'terraform plan -input=false -out=tfplan'
            }
        }

        stage('Approval') {
            when {
                branch 'main'   // only require approval for prod-bound branch
            }
            steps {
                input message: 'Apply this Terraform plan to production?', ok: 'Apply'
            }
        }

        stage('Terraform Apply') {
            when {
                branch 'main'
            }
            steps {
                sh 'terraform apply -input=false tfplan'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'tfplan', allowEmptyArchive: true
        }
        failure {
            echo 'Pipeline failed — check the plan/apply logs above.'
        }
    }
}
```

#### What each stage does
Stage	Purpose
Checkout	Pulls your .tf code from Git
Terraform Init	Downloads providers, connects to remote backend
Format Check	Fails the build if code isn't formatted (terraform fmt) — enforces style consistency
Validate	Catches syntax errors before wasting time on a plan
Plan	Computes and saves the exact plan to tfplan — this is what gets reviewed
Approval	Manual gate — a human clicks "Apply" in the Jenkins UI before anything touches real infra (critical for prod safety)
Apply	Applies the exact previously reviewed plan file — no surprises between plan and apply

TF_IN_AUTOMATION=true is a Terraform-recognized env var that slightly adjusts CLI output to be more automation-friendly (removes some "next steps" hints meant for interactive humans).

### Step 5: The Manual Approval Gate (Deep Dive)

The input step is one of the most important safety mechanisms when running Terraform via Jenkins for production infrastructure:

```
groovy
stage('Approval') {
    steps {
        input message: 'Apply this Terraform plan to production?', ok: 'Apply'
    }
}
```

When the pipeline reaches this stage, it pauses and Jenkins shows a button in the UI. A human reviews the terraform plan output from the previous stage (visible in the console log or posted elsewhere — see Slack integration below) and clicks Apply to proceed, or aborts the build.

This mirrors the same "plan first, apply after explicit confirmation" safety net Terraform gives you locally — just enforced at the team/pipeline level instead of relying on an individual typing yes.

**Auto-apply for non-prod, gated apply for prod** is a very common pattern:

```
groovy
stage('Terraform Apply') {
    steps {
        script {
            if (env.BRANCH_NAME == 'main') {
                input message: 'Apply to PRODUCTION?'
            }
            sh 'terraform apply -input=false tfplan'
        }
    }
}
```

### Step 6: Handling Pull Requests (Plan-Only on PRs)

A very common workflow: run plan on every PR (so reviewers see the infra impact before merging), but only run apply after merge to main.

```
groovy
pipeline {
    agent any
    stages {
        stage('Terraform Plan') {
            steps {
                sh 'terraform init -input=false'
                sh 'terraform plan -input=false -no-color -out=tfplan | tee plan_output.txt'
            }
        }

        stage('Post Plan to PR') {
            when {
                changeRequest()   // only runs when triggered by a PR
            }
            steps {
                script {
                    def planOutput = readFile('plan_output.txt')
                    // Post planOutput as a PR comment via GitHub API / plugin
                }
            }
        }

        stage('Terraform Apply') {
            when {
                branch 'main'    // only runs on merge to main, not on PRs
            }
            steps {
                sh 'terraform apply -input=false tfplan'
            }
        }
    }
}
```

This way, every teammate can see exactly what infrastructure changes a PR will cause directly in the PR review, before it's ever merged or applied — infrastructure changes get the same review rigor as application code.

## Step 7: Multi-Environment Pipelines (Dev / Staging / Prod)

Combine this with what you learned about .tfvars — a common pattern is parameterizing the Jenkins pipeline itself:

```
groovy
pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    }

    stages {
        stage('Terraform Init') {
            steps {
                sh 'terraform init -input=false'
            }
        }

        stage('Terraform Plan') {
            steps {
                sh "terraform plan -input=false -var-file=${params.ENVIRONMENT}.tfvars -out=tfplan"
            }
        }

        stage('Approval') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: "Apply to ${params.ENVIRONMENT}?"
            }
        }

        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -input=false tfplan'
            }
        }
    }
}
```

A Jenkins user (or an automated trigger) picks ENVIRONMENT from a dropdown when starting the build — same pipeline, same code, different .tfvars file and (optionally) different remote state key per environment.

Tip: in a real setup, you'd typically also vary the state key per environment (e.g., dev/terraform.tfstate vs prod/terraform.tfstate) — often via -backend-config passed during terraform init — so dev and prod never share the same state file.

```
groovy
stage('Terraform Init') {
    steps {
        sh "terraform init -input=false -backend-config=${params.ENVIRONMENT}-backend.hcl"
    }
}
```

```
# prod-backend.hcl
bucket = "my-terraform-state-bucket"
key    = "prod/terraform.tfstate"
region = "ap-south-1"
```

### Step 8: Notifications (Slack Example)

Teams typically want to know when infra changes happen — success or failure:

```
groovy
post {
    success {
        slackSend(
            channel: '#infra-alerts',
            color: 'good',
            message: "✅ Terraform apply succeeded for ${env.BRANCH_NAME} (build ${env.BUILD_NUMBER})"
        )
    }
    failure {
        slackSend(
            channel: '#infra-alerts',
            color: 'danger',
            message: "❌ Terraform pipeline failed for ${env.BRANCH_NAME} (build ${env.BUILD_NUMBER}) — ${env.BUILD_URL}"
        )
    }
}
```

### Step 9: Best Practices for Terraform in Jenkins

1. Always use remote state with locking — Jenkins agents are ephemeral; local state would be lost or inconsistent between builds.

2. Pin Terraform and provider versions — use required_version in terraform {} block and commit .terraform.lock.hcl, so a Jenkins upgrade doesn't silently change behavior.

```
   terraform {
     required_version = "= 1.7.5"
   }
```

3. Separate plan and apply stages, and always apply a saved plan file (tfplan) — never re-run plan implicitly right before apply, since infrastructure could have changed in between.

4. Require manual approval for production applies — never -auto-approve straight to prod without a human checkpoint.

5. Use least-privilege IAM credentials scoped to what the pipeline actually needs to manage — don't give Jenkins admin/root cloud access.

6. Run terraform fmt -check and terraform validate early in the pipeline — fail fast on style/syntax issues before wasting time on a plan.

7. Archive the plan output as a Jenkins build artifact — useful for audits ("what exactly changed in build #245?").

8. Post plan output to PRs so infrastructure changes get the same review as code changes.

9. Use workspaces or separate state keys per environment — never let dev and prod pipelines share one state file.

10. Consider a policy/security scan stage (e.g., tfsec, checkov, or terraform-compliance) before apply, to catch misconfigurations (like open security groups) automatically:

```
groovy
    stage('Security Scan') {
        steps {
            sh 'tfsec .'
        }
    }
```

### Full Example — Production-Grade Jenkinsfile

```
groovy
pipeline {
    agent any

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        AWS_DEFAULT_REGION    = 'ap-south-1'
        TF_IN_AUTOMATION      = 'true'
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Terraform Init') {
            steps {
                sh "terraform init -input=false -backend-config=${params.ENVIRONMENT}-backend.hcl"
            }
        }

        stage('Format & Validate') {
            steps {
                sh 'terraform fmt -check'
                sh 'terraform validate'
            }
        }

        stage('Security Scan') {
            steps {
                sh 'tfsec . || true'   // report but don't hard-fail, adjust as needed
            }
        }

        stage('Terraform Plan') {
            steps {
                sh "terraform plan -input=false -var-file=${params.ENVIRONMENT}.tfvars -out=tfplan"
            }
        }

        stage('Approval') {
            when { expression { params.ENVIRONMENT == 'prod' } }
            steps {
                input message: "Apply this plan to PRODUCTION?", ok: 'Apply'
            }
        }

        stage('Terraform Apply') {
            steps {
                sh 'terraform apply -input=false tfplan'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'tfplan', allowEmptyArchive: true
        }
        success {
            slackSend(channel: '#infra-alerts', color: 'good',
                message: "✅ Terraform apply succeeded — ${params.ENVIRONMENT} (build ${env.BUILD_NUMBER})")
        }
        failure {
            slackSend(channel: '#infra-alerts', color: 'danger',
                message: "❌ Terraform pipeline failed — ${params.ENVIRONMENT} (build ${env.BUILD_NUMBER})")
        }
    }
}
```

### Quick Recap  of The Full Flow

1. Developer opens PR with .tf changes

2. Jenkins auto-triggers: init → fmt check → validate → security scan → plan

3. Plan output posted to PR for team review

4. PR approved & merged to main

5. Jenkins triggers again on main: init → plan → (manual approval for prod) → apply

6. State updated in remote backend (S3), locked during apply

7. Slack notification sent on success/failure

This turns Terraform from "a tool someone runs from their laptop" into a proper GitOps-style infrastructure delivery pipeline — reviewed, auditable, consistent, and safe.