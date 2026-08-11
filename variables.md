# Terraform Variables

Variables are what turn a static, hardcoded Terraform config into a reusable, parameterized one. Instead of hardcoding "t2.micro" everywhere, you define a variable once and reference it — letting the same code deploy differently for dev, staging, and prod.

---------------------------------------------------------------------------------------------------

**1. Input Variables**

An input variable is a named parameter your Terraform configuration accepts from the outside — similar to a function parameter in programming.

### Declaring a variable

You declare variables in a variable block, typically in a file called variables.tf (by convention, not a requirement — Terraform reads all .tf files in a directory together):

```
variable "instance_type" {
  description = "The EC2 instance type to use"
  type        = string
}
```

Using it in your config

```
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type    # ← reference with var.<name>
}
```

**Anatomy of a variable block**

```
variable "instance_count" {
  description = "Number of EC2 instances to create"
  type        = number
  default     = 2
  sensitive   = false

  validation {
    condition     = var.instance_count > 0
    error_message = "instance_count must be greater than 0."
  }
}
```

Argument	Purpose
description	Documents what the variable is for (shown in terraform plan, docs, and -help)
type	Enforces the data type — Terraform errors if you pass the wrong type
default	Value used if none is provided (makes the variable optional)
sensitive	Hides the value from CLI output/logs (e.g., for passwords)
validation	Custom rule(s) to reject invalid input with a clear error message


### Supported types

```
variable "name" {
  type = string
}

variable "port" {
  type = number
}

variable "enable_monitoring" {
  type = bool
}

variable "availability_zones" {
  type = list(string)
  default = ["ap-south-1a", "ap-south-1b"]
}

variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Team        = "devops"
  }
}

variable "instance_config" {
  type = object({
    instance_type = string
    ami           = string
    monitoring    = bool
  })
}

variable "allowed_ports" {
  type = set(number)
  default = [22, 80, 443]
}
```

**Example — using variables together**


```
#variables.tf

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_name" {
  description = "Name tag for the instance"
  type        = string
}

variable "tags" {
  type    = map(string)
  default = {}
}

#main.tf

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type

  tags = merge(
    var.tags,
    { Name = var.instance_name }
  )
}
```

Validation example

```
variable "environment" {
  type = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}
```

If someone passes environment = "test":

```
│ Error: Invalid value for variable
│
│   on variables.tf line 1:
│    1: variable "environment" {
│
│ environment must be one of: dev, staging, prod.
```

**2. Default Values**

What a default does, If a variable has a default, it becomes optional — you don't have to supply a value; Terraform uses the default if none is given.

```
variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

If you never override this, every terraform plan/apply uses t2.micro.

### No default = required variable

```
variable "instance_name" {
  type = string
  # no default → Terraform WILL prompt you if you don't supply a value
}
```

If you run terraform plan without supplying instance_name, Terraform interactively prompts:

```
var.instance_name
  Name tag for the instance

  Enter a value:
```

This is usually undesirable in automated pipelines (CI/CD would hang waiting for input) — so in practice, required variables should always be supplied via -var, .tfvars, or environment variables in automation contexts.

### Best practice

* Give sensible defaults to variables that rarely change (e.g., region = "ap-south-1")

* Leave critical/environment-specific variables (e.g., environment, instance_name) without defaults, so they must be deliberately supplied — reducing the risk of accidentally deploying with the wrong values

**3. .tfvars Files**
What they are (.tfvars) files let you supply variable values in bulk, from a file, instead of typing -var flags repeatedly. This is the standard way to manage different environments (dev/staging/prod) with the same codebase.

#### Example — dev.tfvars

```
instance_type = "t2.micro"
instance_name = "dev-web-server"
tags = {
  Environment = "dev"
  Team        = "devops"
}
```

#### Example — prod.tfvars

```
instance_type = "t2.large"
instance_name = "prod-web-server"
tags = {
  Environment = "prod"
  Team        = "devops"
}
```
**Using a .tfvars file**
```
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="prod.tfvars"
```

Same code, different .tfvars file → completely different environment deployed.

**Auto-loaded .tfvars files (no flag needed)**

Terraform automatically loads certain filenames without you passing -var-file:

|Filename	             |           Auto-loaded?                                       |
|------------------------|--------------------------------------------------------------|
|terraform.tfvars	     |          ✅ Yes, always                                      |
|terraform.tfvars.json	 |          ✅ Yes, always                                      |
|*.auto.tfvars	         |          ✅ Yes, always (e.g., dev.auto.tfvars)              |    
|*.auto.tfvars.json	     |          ✅ Yes, always                                      |
|dev.tfvars (custom name) |	        ❌ No — must pass -var-file="dev.tfvars" explicitly |

Example — terraform.tfvars (auto-loaded):

```
instance_type = "t2.micro"
instance_name = "auto-loaded-server"
```
```
$ terraform plan
# automatically picks up terraform.tfvars — no flag needed
```

**JSON format is also supported**

```
json
// terraform.tfvars.json
{
  "instance_type": "t2.micro",
  "instance_name": "json-configured-server"
}
```

Useful when values are generated programmatically (e.g., by another script or pipeline step).

**Sensitive values — avoid .tfvars in Git**

```
# secrets.tfvars  ← DO NOT commit this to Git
db_password = "SuperSecret123!"
```
```
terraform apply -var-file="secrets.tfvars"
```

Add *.tfvars (except example/template files) to .gitignore, and prefer a secrets manager (Vault, AWS Secrets Manager) or CI/CD secret variables for truly sensitive values instead of plain .tfvars files.

**4. CLI Variables (-var flag)**

What it does, You can pass individual variable values directly on the command line — useful for quick overrides, scripting, or CI/CD pipelines that inject values dynamically.

```
terraform plan -var="instance_type=t2.small"
terraform apply -var="instance_type=t2.small" -var="instance_name=quick-test"
```

**Multiple -var flags**

```
terraform apply \
  -var="instance_type=t2.large" \
  -var="instance_name=prod-web" \
  -var='tags={"Environment"="prod","Team"="devops"}'
```

**Note:** complex types (maps, lists) need careful quoting — this is one reason .tfvars files are usually preferred over long -var chains for anything beyond a couple of simple string overrides.

#### Environment variables (TF_VAR_ prefix)

Terraform also reads variables from environment variables prefixed with TF_VAR_ — very common in CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins) where secrets are injected as env vars:

```
export TF_VAR_instance_type="t2.micro"
export TF_VAR_db_password="SuperSecret123!"
```
```
terraform plan
# Terraform automatically picks up TF_VAR_instance_type as var.instance_type
```

This is often the preferred way to pass secrets — the value never appears in a file or shell history the way -var="password=..." would (though it can still show up in process listings, so a dedicated secrets manager is even safer for highly sensitive data).

#### Variable Precedence (Important!)

When the same variable is defined in multiple places, Terraform uses this order — later sources override earlier ones:

1. Environment variables (TF_VAR_*)                    ← lowest priority
2. terraform.tfvars (if present)
3. terraform.tfvars.json (if present)
4. *.auto.tfvars / *.auto.tfvars.json (alphabetical order)
5. -var and -var-file flags on the command line         ← highest priority
   (and within these, later flags on the same command override earlier ones)

**Example demonstrating precedence**
```
# variables.tf
variable "instance_type" {
  type    = string
  default = "t2.nano"     # priority 0 (lowest — used only if nothing else set)
}
```
```
export TF_VAR_instance_type="t2.micro"     # priority 1
```

```
# terraform.tfvars
instance_type = "t2.small"                  # priority 2
```
```
terraform apply -var="instance_type=t2.large"    # priority 3 — WINS
```

**Result:** instance_type = "t2.large" — because -var on the command line has the highest precedence, overriding the environment variable, the .tfvars file, and the default.

------------------------------------------------------------------------------------------------------------

Full Practical Example — Multi-Environment Setup

**Directory structure:**

    .
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── dev.tfvars
    ├── staging.tfvars
    └── prod.tfvars

**variables.tf:**

```
variable "environment" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "instance_count" {
  type    = number
  default = 1
}
```

**main.tf:**

```
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type

  tags = {
    Name        = "web-${var.environment}-${count.index}"
    Environment = var.environment
  }
}
```

**dev.tfvars:**
 
```
    environment    = "dev"
    instance_type  = "t2.micro"
    instance_count = 1
```

**prod.tfvars:**

```
environment    = "prod"
instance_type  = "t2.large"
instance_count = 3
```

**Deploy dev:**

```
terraform apply -var-file="dev.tfvars"
# → 1 x t2.micro instance, tagged "web-dev-0"
```

**Deploy prod:**

```
terraform apply -var-file="prod.tfvars"
# → 3 x t2.large instances, tagged "web-prod-0", "web-prod-1", "web-prod-2"
```


Same main.tf, completely different deployments — this is the core value of variables: one codebase, many environments.
