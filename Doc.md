# Terraform

## Why Infrastructure Automation (Terraform)?

## What is infrastructure?

In a software context, **infrastructure** means all the underlying resources an application needs to run — not the application code itself, but everything it sits on top of:

**Compute**: servers, virtual machines, containers

**Networking**: VPCs, subnets, load balancers, DNS, firewalls

**Storage**: disks, object storage (like S3), databases

**Identity & access**: users, roles, permissions

**Supporting services**: message queues, CDNs, monitoring, secrets managers

Basically, if you deployed an app to the cloud, everything you'd need to click through in AWS/Azure/GCP's console to make that app reachable and functional — that's infrastructure.

## Manual infrastructure vs automated infrastructure

Manual provisioning means a human logs into a cloud console (or SSHs into a server) and creates resources by hand — clicking "Create VM," configuring a security group through a UI, setting up a database instance step by step.

Automated provisioning means you write code that describes the infrastructure you want, and a tool reads that code and creates/updates/destroys the actual resources for you. Terraform is one such tool — you declare "I want a VM with these specs, in this network, with this firewall rule," and Terraform figures out the API calls needed to make that true.

This is often called **Infrastructure as Code (IaC)** : infrastructure defined in files, checked into version control(GIT), just like application code.

## Problems in manual provisioning

a. **Not repeatable / inconsistent** — Doing the same steps twice (e.g., for dev, staging, prod) rarely produces identical environments. Small human differences creep in, leading to the classic "works on staging, breaks in prod" problem.

b. **No version history** — If someone changes a firewall rule or resizes a server manually, there's often no record of what changed, who changed it, or why. Rolling back is guesswork.

c. **Slow and doesn't scale** — Clicking through a console to create one server is fine. Doing it for 50 servers, or recreating an entire environment after a disaster, is painfully slow and error-prone.

d. **Human error** — Manual steps mean typos, forgotten steps, misconfigured settings. A missed checkbox in a security group can cause an outage or a security hole.

e. **No single source of truth** — Nobody can look at one place and know exactly what infrastructure exists. You'd have to log into the console and manually audit everything ("configuration drift" — the real world silently diverging from what people think is deployed).

f. **Hard to collaborate** — Multiple people manually changing the same environment can step on each other's changes, with no merge/review process like code has (pull requests, diffs, approvals).

g. **Poor disaster recovery** — If a manually-built environment gets deleted or corrupted, rebuilding it from memory or scattered documentation is slow and risky.

h. **No testing or preview** — You can't easily "dry run" a manual change to see its blast radius before applying it.

" Automation tools like Terraform solve these by giving you declarative, version-controlled, repeatable, previewable infrastructure — you write the desired end state once, and Terraform handles creating it consistently, every time, anywhere ".

# Introduction to Infrastructure as Code (IaC)

## What is IaC?

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than through manual processes or interactive configuration tools.

Instead of clicking around a cloud console, you write code (in a file) that describes what infrastructure you want — servers, networks, databases, permissions — and a tool reads that file and creates it for you. That code is:

**Version-controlled** (stored in Git, just like application code)

**Reusable** (same code can spin up dev, staging, prod)

**Reviewable** (changes go through pull requests, diffs, approvals)

**Executable** (running the code produces real infrastructure)

Popular IaC tools: Terraform, AWS CloudFormation, Pulumi, Ansible, Azure Resource Manager (ARM) templates.

## Declarative vs Imperative approach

This is one of the most important distinctions in IaC.

### Imperative approach 
    you specify the exact steps to reach a goal, in order.

**"Create a VM. Then attach a disk. Then install these packages. Then start this service."**

You're telling the system how to do something, step by step. Tools like shell scripts, Ansible (mostly), and Chef tend to lean imperative.

### Declarative approach 
    you specify the desired end state, and let the tool figure out how to get there.

**"I want 1 VM with this disk, this OS, and this service running."**

You don't say how — you say what. The tool (e.g., Terraform) compares the desired state to the current state and figures out the steps needed to reconcile the difference.

Terraform is declarative.

Aspect	                    Imperative	                     Declarative
--------------------------------------------------------------------------------------------------------------
You define	                 Steps to execute	            Desired end state
Order matters?	                Yes, strictly	            Tool figures out order (dependency graph)
Re-running same code	May cause errors/duplicates	        Safe — tool only makes needed changes (idempotent)
Example tools	        Bash scripts, Ansible playbooks	    Terraform, CloudFormation
Mental model	        "Do this, then this, then this"	    "Make it look like this"

### Why declarative matters in practice: 

if you run your Terraform code today, and again next week, Terraform checks the current real-world state against your code and only changes what's different. Run an imperative script twice and you might try to create the same VM twice, or hit an error because a resource already exists — unless someone carefully coded in checks for that.

## Benefits of IaC

1. **Consistency** — Same code = same result, every time, in every environment. No more "it worked when I clicked it manually last time."

2. **Speed** — Spin up entire environments in minutes instead of hours/days of manual clicking.

3. **Version control & audit trail** — Every infrastructure change is a commit. You know who changed what, when, and why — and can roll back if needed.

4. **Collaboration** — Infrastructure changes go through the same review process as code (pull requests, code review) instead of being invisible manual actions.

5. **Reusability** — Write a module once (e.g., "a standard web server setup"), reuse it across projects and teams.

6. **Preview before applying** — Tools like Terraform let you run a "plan" to see exactly what will change before you commit to the change — reducing risk.

7. **Disaster recovery** — If an environment is destroyed, you can rebuild it from code in minutes rather than trying to remember/reconstruct manual steps.

8. **Reduced human error** — No manual clicking means fewer typos, forgotten steps, and misconfigurations.

9. **Documentation as a side effect** — The code itself documents exactly what infrastructure exists — no need for separate, often-outdated wiki pages.

10. **Cost control** — Easier to spin down unused/temporary environments (e.g., a terraform destroy after a demo) since everything is tracked and reproducible.

## What is Terraform?

### Terraform overview

Terraform is an open-source **Infrastructure as Code (IaC)** tool created by **HashiCorp**. It lets you define infrastructure — servers, networks, storage, DNS, databases, and more — using a declarative configuration language, and then automatically creates, updates, or destroys that infrastructure to match your code.

Core workflow, in three steps:

**Write** — Define desired infrastructure in .tf files using HashiCorp Configuration Language (HCL).

**Plan** — Run terraform plan to preview exactly what Terraform will create, change, or destroy — before anything actually happens.

**Apply** — Run terraform apply to execute those changes against real infrastructure (AWS, Azure, GCP, etc.).

Terraform keeps track of everything it manages in a state file, which acts as its source of truth for what currently exists — this is how it knows what to change on the next run instead of recreating everything from scratch.

## HashiCorp Terraform

Terraform is built and maintained by HashiCorp, a company known for a suite of DevOps/infrastructure tools:

**Terraform** — infrastructure provisioning

**Vault** — secrets management

**Consul** — service networking/discovery

**Nomad** — workload orchestration

**Packer** — machine image building

Terraform is written in Go and uses HCL (HashiCorp Configuration Language) — a language designed to be both human-readable and machine-friendly, striking a balance between JSON's machine-parsability and a more approachable syntax for humans.

One important note for anyone following Terraform closely: in 2023, HashiCorp changed Terraform's license from the open-source MPL 2.0 to the Business Source License (BSL), which restricts certain competing commercial uses. In response, the community forked the last MPL-licensed version into OpenTofu, now a Linux Foundation project that remains fully open-source and is largely a drop-in replacement. This is worth knowing since some organizations choose OpenTofu over Terraform specifically because of this licensing shift.

## Why Terraform is popular in DevOps

1. **Declarative & predictable** — You describe the end state; Terraform figures out the steps, and terraform plan shows you exactly what will happen before it happens — reducing risky surprises.

2. **State management** — Terraform tracks real-world infrastructure state, enabling it to detect drift (when reality doesn't match code) and reconcile it.

3. **Huge ecosystem of providers** — Thousands of providers/plugins exist (AWS, Azure, GCP, Kubernetes, GitHub, Datadog, Cloudflare, and many more), so Terraform can manage almost any tool or service with an API, not just cloud infrastructure.

4. **Modularity & reusability** — Terraform modules let teams package reusable infrastructure patterns (e.g., "standard VPC setup") and share them across projects.

5. **Strong community & documentation** — Large user base, an official public registry of modules/providers, and years of battle-testing across the industry.

6. **Fits naturally into CI/CD** — Terraform code can be linted, tested, reviewed via pull requests, and applied automatically in pipelines — bringing infrastructure changes into the same rigor as application code deploys.

7. **Plan/Apply safety net** — The separation between preview (plan) and execution (apply) is a major trust-builder for teams making changes to production infrastructure.

## Cloud-agnostic tool

Terraform is cloud-agnostic, meaning it isn't tied to a single cloud provider. Unlike AWS CloudFormation (AWS-only) or Azure Resource Manager templates (Azure-only), Terraform uses a provider plugin model:

Same HCL language, same workflow (init, plan, apply) regardless of which infrastructure you're managing.
You can manage AWS, Azure, GCP, Kubernetes, Oracle Cloud, DigitalOcean, and hundreds of other platforms — even multiple clouds in the same configuration.
This makes Terraform especially valuable for organizations that are multi-cloud, migrating between clouds, or want to avoid vendor lock-in on the tooling side (though the underlying resources themselves are still provider-specific — Terraform doesn't abstract away the differences between, say, an AWS EC2 instance and an Azure VM, it just lets you manage both with one tool and workflow).

Important nuance: "cloud-agnostic tool" ≠ "cloud-agnostic infrastructure." Terraform gives you one consistent way to manage different clouds, but your actual .tf code for AWS resources will look different from your code for Azure resources, since each cloud's services and APIs are different. What's portable is the tool and workflow, not necessarily the configuration itself.

## Terraform Architecture

Terraform's architecture is built around a simple loop: you declare what you want → Terraform figures out what needs to change → Terraform calls the right APIs to make it happen → Terraform remembers what it did.

![alt text](image.png)

Four components make this work: 
**CLI (Core), Providers, Resources, and the State file** . Let's go through each, then go deep on state.

1. Terraform CLI (Core)

Terraform Core is the engine. It does four jobs:

1. Reads your **.tf** configuration files

2. Builds a dependency graph of all resources (so it knows what order to create/destroy things in)

3. Diffs **desired state** (your code) against **current state** (the state file) to compute a plan

4. Executes that plan by calling provider APIs

Core commands and what happens internally:
```
    terraform init
```
* Downloads and installs the providers listed in **required_providers**

* Sets up the backend (where state will be stored)

* Creates a **.terraform/** directory locally with provider binaries

```
    terraform plan
```
* Reads current .tf code

* Reads current state file

* (Optionally) refreshes state by querying real infrastructure

* Computes a diff: what needs to be added, changed, or destroyed

* Shows you this diff — nothing is changed yet


```
    terraform apply
```

* Re-runs the plan (or uses a saved plan file)

* Asks for confirmation (yes)

* Calls provider APIs in dependency order to create/update/destroy real resources

* Writes the results into the state file

```
    terraform destroy
```

* Same as apply, but the **"desired state"** is treated as empty — so everything Terraform is tracking gets deleted

### Core commands:

```
terraform init      # Initializes working directory, downloads providers
terraform plan       # Shows what WILL change (preview, no actual changes made)
terraform apply       # Executes the changes (creates/updates/destroys real resources)
terraform destroy    # Tears down everything Terraform manages in this config
terraform validate   # Checks syntax/config validity without contacting the cloud
terraform fmt        # Auto-formats .tf files to standard style
```

### Example workflow:

```
    $ terraform init
    Initializing the backend...
    Initializing provider plugins...
    - Installing hashicorp/aws v5.40.0...
    Terraform has been successfully initialized!

    $ terraform plan
    Terraform will perform the following actions:
    # aws_instance.web will be created
    + resource "aws_instance" "web" {
        + ami           = "ami-0abcdef1234567890"
        + instance_type = "t2.micro"
        + id            = (known after apply)
        }
    Plan: 1 to add, 0 to change, 0 to destroy.

    $ terraform apply
    Do you want to perform these actions?
    Terraform will perform the actions described above.
    Only 'yes' will be accepted to approve.
    Enter a value: yes

    aws_instance.web: Creating...
    aws_instance.web: Creation complete after 32s [id=i-0a1b2c3d4e5f]
    Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Note**:Notice init runs once per project setup, then plan → apply becomes your everyday loop.

2. Providers

A provider is a plugin that lets Terraform talk to a specific platform's API — AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare, etc. The provider translates your HCL resource blocks into actual API calls.

You declare which provider(s) you're using at the top of your config:

hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"   # Hyderabad-region users often pick ap-south-1 (Mumbai)
}
source tells Terraform where to download the provider from (the public Terraform Registry, by default hashicorp/aws).
version pins which version(s) are acceptable, so your infra doesn't silently break when a new provider version changes behavior.
provider "aws" { region = ... } configures how that provider connects (region, credentials, etc.)

You can even use multiple providers in one config — e.g., managing both AWS and Cloudflare DNS at once:

hcl
provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}
3. Resources

A resource is the actual infrastructure object you want Terraform to create/manage — a VM, a network, a database, a DNS record. This is the core building block of any Terraform config.

Syntax:

hcl
resource "<PROVIDER_TYPE>" "<LOCAL_NAME>" {
  argument1 = value1
  argument2 = value2
}

Example — an AWS EC2 instance:

hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"

  tags = {
    Name = "MyWebServer"
  }
}
aws_instance → resource type (defined by the AWS provider)
web → your local name, used to reference this resource elsewhere in code (e.g., aws_instance.web.id)
Everything inside {} → configuration arguments specific to that resource type

Example — referencing one resource from another (this is where Terraform's dependency graph shines):

hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id      # references the VPC created above
  cidr_block = "10.0.1.0/24"
}

Terraform automatically figures out that aws_subnet.public depends on aws_vpc.main, and creates the VPC first, then the subnet — you don't have to specify order manually. This dependency graph is a key part of why Terraform is declarative rather than imperative.

4. State file

The state file (terraform.tfstate) is how Terraform keeps track of what infrastructure it has actually created, and maps it back to your configuration. It's Terraform's "source of truth" for the real world.

Why it's needed: Terraform config describes desired state. The state file records current (last-known) state. When you run terraform plan, Terraform compares:

Your .tf code (desired)  vs  State file (last known)  vs  Real infrastructure (actual)

...and figures out the minimum changes needed to reconcile them.

Example — simplified terraform.tfstate content (JSON):

json
{
  "version": 4,
  "terraform_version": "1.7.0",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "i-0a1b2c3d4e5f",
            "ami": "ami-0abcdef1234567890",
            "instance_type": "t2.micro",
            "public_ip": "13.234.56.78"
          }
        }
      ]
    }
  ]
}

Notice it stores actual real-world values (like id and public_ip) that only exist after creation — your .tf code doesn't know these in advance, but the state file does, once applied.

What the state file enables:

Without state	With state
Terraform doesn't know what already exists	Terraform knows exactly what it manages
Every apply might try to recreate everything	Only diffs are applied (idempotent)
No way to map .tf resource → real cloud object	State maps aws_instance.web → i-0a1b2c3d4e5f
Can't detect drift	terraform plan shows drift vs. code

Important practical notes:

State often contains sensitive data (e.g., DB passwords in attributes), so it should never be committed to public Git repos.
In teams, state is usually stored remotely (e.g., an S3 bucket + DynamoDB lock table, or Terraform Cloud) instead of a local file, so multiple people don't overwrite each other's state and to avoid "I ran apply on my laptop and now my state is out of sync with everyone else's."
hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "prod/terraform.tfstate"
    region = "ap-south-1"
    dynamodb_table = "terraform-locks"   # prevents concurrent applies
  }
}

Putting it all together — a minimal end-to-end example:

hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  tags = {
    Name = "MyWebServer"
  }
}

Run terraform init (downloads AWS provider) → terraform plan (shows it will create 1 instance) → terraform apply (creates it, records details in terraform.tfstate) → run terraform plan again later and Terraform compares your code + state + real AWS to tell you if anything drifted.