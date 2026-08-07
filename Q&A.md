## State Management

### Q1: Two engineers ran terraform apply at the same time and now the state file is corrupted. How do you handle this, and how would you prevent it?
### Ans:
If the state is corrupted, first restore from a backup — Terraform automatically creates a terraform.tfstate.backup locally, or if using remote state (S3, Terraform Cloud), you can pull a previous version from versioning history. Use terraform state pull to inspect the current state and terraform state push to restore a known-good version carefully.

To prevent this going forward: use a remote backend with locking, like S3 + DynamoDB (DynamoDB provides the lock table) or Terraform Cloud/Enterprise, which locks state automatically during operations so concurrent applies queue up instead of corrupting the file.

### How to Handle State Corruption
* Stop all work: Tell your team to stop running any plan or apply commands to prevent more damage.
* Restore a backup: Look at your remote storage history (like S3 versioning or GCS object history) and roll back to the version saved right before the crash.
* Inspect the JSON: If versioning is off, open a copy of the broken state file, fix any broken JSON syntax or bad serial numbers, and test it.
* Re-import resources: If the file is totally lost, use terraform import or import blocks to link your real cloud resources back to your code.
* Clear stuck locks: If a lock ID blocks your recovery, clear it using terraform force-unlock <LOCK_ID> very carefully.

### How to Prevent It
1. Use a remote backend: Store state in a shared cloud storage tool instead of a local laptop.
    backend "s3" {
        bucket = "terraform-state"
        key = "prod/network.tfstate"
        region = "us-east-1"
    }
2. Turn on state locking: Use a locking tool (like a DynamoDB table for S3) so only one person can write to the state at a time.
3. Enable versioning: Turn on object versioning on your storage bucket so you can always roll back bad changes.
4. Use a CI/CD pipeline: Run your plans and applies through a central pipeline (like GitHub Actions or GitLab CI) instead of local machines to enforce order and control
5. Separate Workspaces: Never let Dev, QA and Production share one state.

    dev.tfstate
    qa.tfstate
    prod.tfstate

#### Interview answers:
"If two engineers run terraform apply simultaneously and the state becomes inconsistent, my first priority is to stop all Terraform operations to avoid further damage. I verify the remote backend, inspect the state, and compare it with the actual infrastructure. If the backend supports version history, I restore the last known good state only after confirming it matches the deployed resources. If specific resources are missing from state, I use terraform import; if stale entries exist, I remove them with terraform state rm. Finally, I validate the environment using terraform plan before allowing another apply. To prevent this in production, I always use a remote backend with state locking and versioning, restrict terraform apply to CI/CD pipelines, enforce code reviews, and isolate environments with separate state files. These controls ensure only one apply can modify the state at a time and make recovery straightforward if an issue occurs."

### --------------------------------------------------------------------------------------------------------------

### Q2
A resource was deleted manually in the AWS console, but Terraform still thinks it exists. What happens on the next apply, and how do you fix it?

On the next plan, Terraform detects drift — it will show the resource as needing to be created again since the actual infrastructure no longer matches state. If you want Terraform to recreate it, just apply. If the deletion was intentional and you don't want it recreated, remove it from state with terraform state rm <resource> so Terraform stops tracking it, or if you deleted it by mistake and want Terraform to reconcile without recreating in a destructive order, review the plan carefully first — terraform plan before every apply is the real safety net here.

Q3: You need to rename a resource in your Terraform code without destroying and recreating the underlying infrastructure. How?

Renaming a resource block changes its address in state, so a plain terraform apply would show a destroy+create. Instead, use terraform state mv old_resource_address new_resource_address (or moved blocks in newer Terraform versions) to tell Terraform the same infrastructure now maps to the new name, avoiding any actual change to real resources.