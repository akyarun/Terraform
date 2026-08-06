# Terraform

Infrastructure as Code (IaC) tool used to build, modify, and manage infrastructure safely and efficiently.

* Automates infrastructure provisioning instead of manual console configuration.

* Enables version control, collaboration, and repeatable deployments.
    
* Reduces human errors while improving scalability and consistency.

## Infrastructure as Code (IaC)
IaC is the practice of managing IT infrastructure using configuration files rather than manual, interactive configuration tools.

#### --> Declarative: You tell Terraform what you want (e.g., "I want 5 servers"), and Terraform figures out how to create them.
    
#### -->  Version Controlled: You can track the history of your infrastructure changes just like application code.

## Key Features
(a) Cloud Agnostic: Unlike CloudFormation (AWS only) or ARM Templates (Azure only), Terraform works with any cloud provider (AWS, Google Cloud, Azure, Kubernetes, Alibaba, etc.).

(b) Immutable Infrastructure: Terraform typically replaces servers rather than changing them, reducing "configuration drift" (where servers become inconsistent over time).

(c) State Management: Terraform keeps track of your real-world resources in a state file, acting as the "source of truth."

(d) Modular: You can package code into Modules to reuse common patterns (e.g., a standard "Web Server" module used by all teams).

## Working
Terraform uses a declarative configuration language to define infrastructure and manage resources in a predictable, automated workflow. It allows teams to provision, update, and delete infrastructure consistently while supporting multiple cloud providers and environments. This flexibility makes Terraform a widely adopted tool for scalable and reusable infrastructure management.

1. Defines infrastructure as code for repeatable and reliable deployments.

2. Provisions, modifies, and destroys resources through an automated workflow.

3. Supports multi-cloud, on-premises, and hybrid environments with a single configuration.

4. Enhances scalability, reusability, and operational efficiency across industries.

## Architecture
To understand how Terraform works, you need to understand its core components.

1. The Core (Engine)
This is the binary you run on your laptop. It reads your configuration files and compares them to the current state of your infrastructure to calculate what needs to be done.

2. Providers
Terraform doesn't know how to talk to AWS or Azure directly. It uses Providers plugins that translate Terraform code into API calls for specific platforms.
Examples: AWS Provider, Azure Provider, Kubernetes Provider.

3. State File (terraform.tfstate)
This is the brain of Terraform. It maps your code to the real-world resources.
If you delete a resource from your code, Terraform looks at the state file to find the ID of the real resource and delete it from the cloud.
Note: In teams, this file is stored remotely (e.g., in an AWS S3 bucket) so everyone works off the same map.

## Core Elements
### 1. Terraform CLI
Terraform helps you automate the creation and management of infrastructure. To see a list of available commands in Terraform, you can run:

#### terraform --help
This command will display all the available commands, with the most commonly used ones listed first. The primary Terraform commands include:

init: Prepares your directory to run other Terraform commands.
validate: Checks if the configuration is valid.
plan: Shows what changes will be made to your infrastructure.
apply: Executes the changes to create or modify your infrastructure.
destroy: Deletes the infrastructure that was previously created.

### 2. Terraform Language
Terraform uses HashiCorp Configuration Language (HCL) to define infrastructure. HCL is designed to be both easy to read by humans and understandable by machines, making it a great fit for DevOps tools.

Infrastructure elements managed by Terraform are called resources. These can include virtual machines, S3 buckets, VPCs, and databases. Each resource is defined in a block, like this example for creating an AWS VPC:

<ul>
resource "aws_vpc" "default_vpc" {
    cidr_block = "172.31.0.0/16"
    tags = {
        Name = "example_vpc"
    }
}
</ul>

### 3. Terraform Provider
Terraform Provider defines the resource types and data sources Terraform can manage for that platform. Providers allow users to provision, configure, and manage cloud services, databases, networks, and more from a single workflow.

#### Key Points:

* Acts as a bridge between Terraform and infrastructure platforms.
* Defines the resources and data sources available for management.
* Supports cloud providers, data centers, network devices, and databases.
* Enables consistent provisioning across multiple environments.

### 4. Terraform Modules
A Terraform module is a container for a set of related resources that perform a specific task, enabling organized and reusable infrastructure code.

Module Block: Defined using the `module` block in Terraform configuration, which includes the following arguments:-
    
    source: Specifies the location of the module, which can be a local path or a URL.
    
    name: Provides a name to reference the module within the configuration.
    
    version: Specifies a particular version of the module to use.
    
    Resources and Variables: Within a module block, users can define the resources that make up the module, along with input and output variables. Input variables allow values to be passed into the module when it is called, and output variables allow the module to return values to the calling configuration.
    
    Nesting: Modules can be nested, enabling the creation of complex infrastructure architectures using a hierarchical structure.

### 5. Terraform Provisioners
Terraform Provisioners are useful for tasks like copying files or installing software on virtual machines. However, provisioners should be used sparingly, as they can introduce complexity and reduce the predictability of deployments.

* Run commands or scripts after resource provisioning.
* Commonly used for file transfers and software setup.
* Can increase complexity and require elevated permissions.
* Recommended only when native Terraform resources cannot achieve the task.

### 6. Terraform State
Terraform state files allows Terraform to compare the current infrastructure with the desired state and apply only the necessary changes. While the state can be stored locally, remote storage is recommended for team environments to maintain consistency and prevent conflicts.

Tracks real-world resources and their configurations.
Enables accurate planning by detecting infrastructure changes.
Can be stored locally but is safer when managed remotely.
Remote state improves collaboration, consistency, and state locking.

### Local State
By default, Terraform stores the state file locally on the machine where it is executed. This approach is simple and effective for individual use or small projects but can introduce risks in collaborative environments

Simple to set up with no additional backend configuration
Ideal for learning, experimentation, and small-scale deployments
Faster access since the state file resides on the local system

### Limitations of Local State:

* Higher risk of accidental deletion or corruption
* Difficult to share safely across teams
* No built-in locking, increasing the chance of concurrent update conflicts

### Remote State
Remote state stores the Terraform state file in a shared backend such as AWS S3, Azure Storage, or Terraform Cloud. It is considered a best practice for production environments because it enhances security, collaboration, and reliability.

* Enables team collaboration through centralized state management
* Supports state locking to prevent simultaneous modifications
* Improves security with encryption and controlled access

### Common Remote Backends:

AWS S3 (often paired with DynamoDB for state locking)
Terraform Cloud
Azure Blob Storage
Google Cloud Storage

### 7.Terraform Private Module Registry
Private module Registry enables teams to manage, reuse, and distribute infrastructure code internally instead of relying on public registries. By configuring authentication, users can seamlessly reference these modules in their Terraform projects.

Stores organization-specific modules with controlled access.
Requires authentication (e.g., access tokens) to use the registry.
Allows modules to be referenced via the module block like public ones.
Improves governance, security, and code reusability across teams.

## Terraform Commands 
#### 1. Terraform init
    Action: Initializes the working directory.
    What it does: Downloads the necessary Providers (plugins) required for your code (e.g., downloads the AWS plugin).
    $ terraform init
    Terraform init

#### 2. Terraform Plan
    Action: Creates an execution plan.
    What it does: It compares your code to the current state and shows you a "preview" of what will happen. It will say, "I plan to create 3 resources and destroy 1." It does not make changes yet.

#### 3. Terraform apply 
    Action: Executes the changes.
    What it does: It reaches out to the Cloud API to create/delete resources to match your plan.
    $ terraform apply
    Terraform apply

#### 4. Terraform destroy 
    Action: Tears down everything.
    What it does: Deletes all resources tracked in the state file.
    $ terraform destroy

#### 5. Terraform import
    Imports an existing resource into the Terraform state, allowing it to be managed by Terraform.

    $ terraform import

#### 6. Terraform console
    Opens an interactive console for evaluating expressions in the Terraform configuration.

    $ terraform console

#### 7. Terraform refresh 
    This command updates the state of your infrastructure to reflect the actual state of your resources. It is useful when you want to ensure that your Terraform state is in sync with the actual state of your infrastructure.

    $ terraform refresh

## Basic Terraform Example
Below is a main.tf file that provisions a simple EC2 instance on AWS.

        # 1. Define the Provider
        provider "aws" {
        region = "us-east-1"
        }

        # 2. Define a Resource (The "What")
        # Syntax: resource "type" "name"
        resource "aws_instance" "my_web_server" {
        ami           = "ami-0c55b159cbfafe1f0" # Ubuntu AMI ID
        instance_type = "t2.micro"

        tags = {
            Name = "DevOps-Server"
        }
        }

#### Explaining the Syntax (HCL)
resource: The keyword to define infrastructure.
aws_instance: The type of resource (provided by the AWS plugin).
my_web_server: The internal name Terraform uses to track this resource.
Arguments: The block inside { } defines the properties (like ami and instance_type).

----------------------****--------------------------------****---------------------****--------------------------****-----------------------------
Terraform vs Other Infrastructure as Code (IaC) Tools
Infrastructure as Code (IaC) tools are essential for automating and managing infrastructure. Terraform is a popular choice, but there are several other tools that serve similar purposes. Here’s a straightforward comparison to help you understand the differences.

1. Terraform vs AWS CloudFormation
The following is the comparison table between Terraform and Cloudformation:

Feature	Terraform	AWS CloudFormation
Scope	Multi-Cloud (AWS, Azure, GCP, etc.).	AWS Only.
Language	HCL (Simple, clean, easy to read).	JSON or YAML (Can get very verbose and complex).
State	managed by user (Local or Remote).	Managed automatically by AWS.
2. Terraform vs Ansible
The following is the comparison table between Terraform and Ansible:

Feature

Terraform

Ansible

Primary Use

Focuses on setting up and managing infrastructure.

Primarily for configuring systems and deploying applications.

Language

Uses HCL for infrastructure definitions.

Uses YAML for defining tasks.

Stability

Automatically ensures resources are created only if necessary.

Requires careful task definition to avoid duplication.

Execution

Manages infrastructure changes using plans and state.

Executes tasks immediately without state tracking.

Cloud Support

Excellent multi-cloud capabilities.

Useful for multi-cloud configurations but limited to system-level tasks.