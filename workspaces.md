# Workspace

Terraform workspaces let you maintain multiple, isolated state files for the same configuration — useful for spinning up parallel environments (dev/staging/prod) or parallel instances (per-feature-branch sandboxes) without duplicating your .tf files. Let's go through this fully, including where they fall short.

## 1. What a workspace actually is

Every backend has a default workspace called default, which is what you've been using this whole conversation without realizing it. A workspace is nothing more than a named, separate slot for state, all still using the exact same configuration files.

## 2. How state is actually stored per workspace

The mechanism differs slightly by backend, but for the S3 backend specifically, workspaces get an automatic key prefix:

    mycompany-terraform-state/
    ├── env:/
    │   ├── dev/prod/networking/terraform.tfstate
    │   └── staging/prod/networking/terraform.tfstate
    └── prod/networking/terraform.tfstate     ← default workspace, no prefix

So the default workspace uses exactly the key you configured, while every non-default workspace gets stored at env:/<workspace_name>/<key>. This is fully automatic — you don't manage these paths yourself.

## 3. Commands

    terraform workspace list          # * marks the currently selected workspace
    terraform workspace new staging   # creates AND switches to a new workspace
    terraform workspace select prod   # switches to an existing workspace
    terraform workspace show          # prints the current workspace name
    terraform workspace delete staging   # deletes (must have zero resources or be empty)

Typical flow to stand up three environments from scratch:

```
    terraform workspace new dev
    terraform apply    # creates dev's resources, tracked in dev's state

    terraform workspace new staging
    terraform apply    # creates staging's resources, separate state

    terraform workspace new prod
    terraform apply    # creates prod's resources, separate state
```

## 4. Using terraform.workspace in configuration

Terraform exposes the current workspace name as a built-in read-only value: terraform.workspace. This is what lets one configuration behave differently per environment:

```
    locals {
    instance_size = {
        dev     = "t3.micro"
        staging = "t3.medium"
        prod    = "t3.large"
    }
    }

    resource "aws_instance" "app" {
    instance_type = local.instance_size[terraform.workspace]
    tags = {
        Environment = terraform.workspace
    }
    }
```

Naming resources to avoid collisions across environments (if they ever did share infrastructure, like an S3 bucket needing a globally unique name):

```
resource "aws_s3_bucket" "assets" {
  bucket = "myapp-assets-${terraform.workspace}"
}
```

## 5. Multiple environments — where this genuinely fits

Workspaces are best suited for many near-identical, short-lived instances of the same configuration — the classic case being ephemeral pre-prod/PR-preview environments, or parallel sandbox environments for a team, where you want terraform workspace new pr-1234, apply, test, then terraform workspace delete pr-1234 and tear it down.

## 6. Where workspaces fall short for dev/staging/prod

This is the part that trips people up, so it's worth being direct about it: HashiCorp itself does not recommend CLI workspaces as the primary tool for separating dev/staging/prod. The reasons:

* **Same backend config, same variables file by default** — all workspaces share one backend block and, unless you explicitly branch on terraform.workspace everywhere, the same .tfvars. This makes it easy to accidentally apply prod-sized values against dev, or vice versa.

* **No credential/access isolation** — workspaces don't map to separate AWS accounts or IAM boundaries. Prod and dev sharing an AWS account because they share a workspace setup is a real blast-radius risk.

* **Silent context switching** — terraform workspace select prod changes your active workspace with no confirmation prompt. Running apply right after switching, without checking terraform workspace show first, is a common way people accidentally apply to the wrong environment.
* **State-only isolation** — workspaces isolate state, not infrastructure boundaries, access control, or configuration. If dev and prod genuinely need different instance types, different VPCs, different account-level security policies, you end up building elaborate terraform.workspace-keyed lookup maps throughout your config — which becomes harder to read than just having separate files.

**Note:**The commonly recommended alternative for dev/staging/prod specifically is directory-per-environment, each with its own backend config, own .tfvars, and often its own module call:

    environments/
    ├── dev/
    │   ├── main.tf          # calls the same shared module
    │   └── backend.tf       # its own S3 key, maybe its own account
    ├── staging/
    │   ├── main.tf
    │   └── backend.tf
    └── prod/
        ├── main.tf
        └── backend.tf

Each environment is a genuinely separate root module (separate init, separate state, separate backend), while still reusing the exact same underlying module for the actual infrastructure — giving you the reuse benefit of modules and real isolation between environments, at the cost of a bit more file duplication in the environment wrapper files.

One last practical safeguard if you do use CLI workspaces for anything meaningful: always run terraform workspace show (or bake it into your prompt/CI output) immediately before any apply, since there's no built-in guardrail stopping you from applying the wrong environment's changes to the wrong state.