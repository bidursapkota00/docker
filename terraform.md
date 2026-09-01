# Terraform Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Terraform](#introducing-terraform)
2. [Installation & Setup](#installation--setup)
3. [HCL Basics](#hcl-basics)
4. [Providers](#providers)
5. [Resources](#resources)
6. [Terraform Workflow](#terraform-workflow)
7. [Variables](#variables)
8. [Outputs](#outputs)
9. [Data Sources](#data-sources)
10. [State Management](#state-management)
11. [Modules](#modules)
12. [Provisioners](#provisioners)
13. [Functions & Expressions](#functions--expressions)
14. [Conditionals & Loops](#conditionals--loops)
15. [Workspaces](#workspaces)
16. [Import Existing Infrastructure](#import-existing-infrastructure)
17. [Backend Configuration](#backend-configuration)
18. [Best Practices](#best-practices)

---

## Introducing Terraform

Terraform is an open-source Infrastructure as Code (IaC) tool created by HashiCorp. It lets you define, provision, and manage cloud infrastructure using a declarative configuration language called HCL (HashiCorp Configuration Language). Instead of manually clicking through cloud consoles or writing imperative scripts, you describe the desired state of your infrastructure in `.tf` files, and Terraform figures out how to create, update, or destroy resources to reach that state.

Terraform is commonly used to provision cloud resources on AWS, Azure, GCP, and hundreds of other providers, manage multi-cloud and hybrid-cloud environments, automate infrastructure deployment in CI/CD pipelines, maintain consistent environments across development, staging, and production, and version-control infrastructure alongside application code.

Core concepts of Terraform are:

- **Provider**: A plugin that interfaces with a specific cloud or service API (e.g., AWS, Azure, GCP, Docker, Kubernetes).
- **Resource**: A single piece of infrastructure managed by Terraform (e.g., an EC2 instance, an S3 bucket, a DNS record).
- **State**: A JSON file (`terraform.tfstate`) that maps your configuration to real-world resources. Terraform uses it to track what exists.
- **Plan**: A preview of what Terraform will create, update, or destroy before making any changes.
- **Module**: A reusable, self-contained package of Terraform configuration.
- **Data Source**: A way to query existing infrastructure or external data without managing it.
- **Backend**: Where Terraform stores its state file (local disk, S3, Azure Blob, etc.).

---

## Installation & Setup

### Install Terraform

**macOS (Homebrew)**:

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

`brew tap` adds the official HashiCorp repository. Installing from the tap ensures you get the latest version directly from HashiCorp.

**Linux (Ubuntu/Debian)**:

```bash
sudo apt update && sudo apt install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform -y
```

This adds the HashiCorp GPG key and APT repository, then installs Terraform from the official source.

**Windows (Chocolatey)**:

```powershell
choco install terraform
```

**Manual Installation (any OS)**:

Download the binary from [terraform.io/downloads](https://developer.hashicorp.com/terraform/downloads), unzip it, and place it in your system's `PATH`.

### Verify Installation

```bash
terraform --version
terraform -help
```

### Enable Tab Completion

```bash
terraform -install-autocomplete        # Add autocomplete to your shell
# Restart your shell after running this
```

---

## HCL Basics

HCL (HashiCorp Configuration Language) is Terraform's declarative language. Files use the `.tf` extension. Terraform reads all `.tf` files in the working directory and merges them into a single configuration.

### File Structure

A typical Terraform project has:

| File                 | Purpose                                   |
| -------------------- | ----------------------------------------- |
| `main.tf`            | Primary resource definitions              |
| `variables.tf`       | Input variable declarations               |
| `outputs.tf`         | Output value definitions                  |
| `providers.tf`       | Provider configuration                    |
| `terraform.tfvars`   | Variable values (not committed to git)    |
| `backend.tf`         | Backend / state configuration             |
| `versions.tf`        | Required provider and Terraform versions  |

These filenames are conventions, not requirements. Terraform loads all `.tf` files in the directory regardless of their names.

### Syntax

```hcl
# Single-line comment

/* Multi-line
   comment */

# Block syntax: type "label1" "label2" { ... }
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  tags = {
    Name = "WebServer"
    Env  = "production"
  }
}
```

Blocks have a type (`resource`), one or more labels (`"aws_instance"`, `"web"`), and a body enclosed in braces. Arguments use `key = value` syntax. Strings are double-quoted. Maps use `{ key = value }` syntax. Lists use `["a", "b", "c"]` syntax.

### Value Types

```hcl
# String
name = "web-server"

# Number
count = 3

# Boolean
enable = true

# List (ordered)
subnets = ["subnet-1a", "subnet-2b", "subnet-3c"]

# Map (key-value pairs)
tags = {
  Name = "web"
  Env  = "prod"
}

# Null
value = null
```

### String Interpolation & Heredoc

```hcl
# Interpolation
name = "server-${var.environment}"

# Directive
description = "Server is %{if var.enabled}active%{else}inactive%{endif}"

# Heredoc (multi-line string)
user_data = <<-EOF
  #!/bin/bash
  echo "Hello, ${var.name}"
  apt update && apt install -y nginx
EOF
```

`${}` embeds expressions inside strings. `<<-EOF` starts a heredoc that allows indentation; the `-` strips leading whitespace. `<<EOF` without the dash preserves whitespace exactly.

---

## Providers

Providers are plugins that let Terraform interact with cloud platforms, SaaS services, and other APIs. Each provider offers a set of resource types and data sources.

### Configuring Providers

```hcl
# providers.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  required_version = ">= 1.5.0"
}

provider "aws" {
  region = "us-east-1"
}

provider "azurerm" {
  features {}
}
```

`required_providers` declares which providers the configuration needs and their version constraints. `source` uses the format `namespace/provider`. `version` accepts constraints: `~> 5.0` means any version `>= 5.0` and `< 6.0`. `required_version` constrains the Terraform CLI version itself.

### Version Constraints

| Constraint   | Meaning                              |
| ------------ | ------------------------------------ |
| `= 5.0.0`   | Exactly version 5.0.0               |
| `>= 5.0`    | Version 5.0 or higher               |
| `~> 5.0`    | Any 5.x version (>= 5.0, < 6.0)    |
| `~> 5.1.0`  | Any 5.1.x version (>= 5.1.0, < 5.2)|
| `>= 5.0, < 6.0` | Between 5.0 (inclusive) and 6.0 (exclusive) |

`~>` is the pessimistic constraint operator. `~> 5.0` allows minor and patch updates but not major version changes.

### Multiple Provider Configurations

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west_server" {
  provider      = aws.west
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
}
```

`alias` creates an alternate provider configuration. Resources use the default provider unless you explicitly set `provider = aws.west`. This is useful for multi-region or multi-account deployments.

### Provider Authentication

Providers typically authenticate via environment variables, shared credentials files, or instance profiles. Avoid hardcoding credentials in `.tf` files.

```bash
# AWS - environment variables
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"

# Azure - CLI login
az login

# GCP - service account
export GOOGLE_CREDENTIALS="/path/to/service-account-key.json"
```

---

## Resources

Resources are the most important element in Terraform. Each resource block declares a single infrastructure object.

### Resource Syntax

```hcl
resource "<provider>_<type>" "<local_name>" {
  # Arguments (configuration)
  argument1 = "value1"
  argument2 = "value2"
}
```

The resource type (`aws_instance`) determines what kind of infrastructure object to manage. The local name (`web`) is used to reference the resource within the configuration. Together, `aws_instance.web` forms a unique identifier.

### AWS Examples

```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

# Security Group
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = "ami-0abcdef1234567890"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]
  key_name               = "my-key-pair"

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }

  tags = {
    Name = "web-server"
  }
}

# S3 Bucket
resource "aws_s3_bucket" "assets" {
  bucket = "my-app-assets-bucket"

  tags = {
    Environment = "production"
  }
}

resource "aws_s3_bucket_versioning" "assets" {
  bucket = aws_s3_bucket.assets.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

`aws_vpc.main.id` references the `id` attribute of the VPC resource. Terraform automatically determines that the subnet depends on the VPC and creates them in the correct order. This is called implicit dependency. `protocol = "-1"` in egress means all protocols.

### Resource References

```hcl
# Reference another resource's attribute
subnet_id = aws_subnet.public.id

# Reference format: <resource_type>.<local_name>.<attribute>
vpc_id    = aws_vpc.main.id
arn       = aws_instance.web.arn
public_ip = aws_instance.web.public_ip
```

When one resource references another, Terraform infers a dependency and creates/updates them in the correct order.

### Explicit Dependencies

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  depends_on = [aws_s3_bucket.assets]
}
```

`depends_on` forces Terraform to create the S3 bucket before the instance, even if there is no direct reference. Use it when there is a hidden dependency that Terraform cannot detect automatically.

### Lifecycle Rules

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true        # Create new before destroying old
    prevent_destroy       = true        # Prevent accidental deletion
    ignore_changes        = [tags]      # Ignore changes to tags
    replace_triggered_by  = [aws_security_group.web_sg.id]
  }
}
```

`create_before_destroy` ensures the replacement is created before the old resource is destroyed, reducing downtime. `prevent_destroy` causes Terraform to error if a plan would destroy this resource. `ignore_changes` tells Terraform to ignore external changes to specified attributes, useful when other tools modify tags. `replace_triggered_by` forces replacement when the referenced resource changes.

### Timeouts

```hcl
resource "aws_db_instance" "main" {
  # ... configuration ...

  timeouts {
    create = "60m"
    update = "30m"
    delete = "30m"
  }
}
```

Some resources support custom timeouts for create, update, and delete operations. This is useful for resources that take a long time to provision, like RDS instances.

---

## Terraform Workflow

The core Terraform workflow consists of Write → Init → Plan → Apply.

### Initialize

```bash
terraform init                         # Download providers, initialize backend
terraform init -upgrade                # Upgrade providers to latest allowed version
terraform init -reconfigure            # Reconfigure backend, ignore existing config
terraform init -migrate-state          # Migrate state to a new backend
terraform init -backend=false          # Skip backend initialization
```

`terraform init` is the first command you run in a new Terraform project. It downloads required provider plugins, initializes the backend (where state is stored), and installs modules. The `.terraform` directory stores downloaded providers and modules. `.terraform.lock.hcl` locks provider versions for reproducibility — commit this file to version control.

### Validate

```bash
terraform validate                     # Check configuration syntax
```

`validate` checks for syntax errors, invalid references, and type mismatches without accessing any remote services or state. Run it before `plan` to catch mistakes early.

### Format

```bash
terraform fmt                          # Format all .tf files in current directory
terraform fmt -recursive               # Format files in subdirectories too
terraform fmt -check                   # Check if files are formatted (CI use)
terraform fmt -diff                    # Show formatting differences
```

`fmt` rewrites `.tf` files to the canonical HCL style. `-check` exits with a non-zero code if files need formatting, useful in CI pipelines.

### Plan

```bash
terraform plan                         # Preview changes
terraform plan -out=tfplan             # Save plan to file
terraform plan -var "region=us-west-2" # Pass a variable
terraform plan -var-file="prod.tfvars" # Use variable file
terraform plan -target=aws_instance.web # Plan only for specific resource
terraform plan -destroy                # Preview what destroy would do
terraform plan -refresh-only           # Only refresh state, no changes
```

`plan` compares your configuration to the current state and shows what Terraform will create (`+`), update (`~`), or destroy (`-`). `-out=tfplan` saves the plan so `apply` can execute exactly what was shown. `-target` limits the plan to a specific resource and its dependencies. `-refresh-only` updates state to match real infrastructure without making changes.

### Apply

```bash
terraform apply                        # Apply changes (prompts for confirmation)
terraform apply -auto-approve          # Skip confirmation prompt
terraform apply tfplan                 # Apply a saved plan file
terraform apply -var "region=us-west-2"
terraform apply -target=aws_instance.web
terraform apply -parallelism=20        # Run up to 20 operations in parallel
```

`apply` executes the changes shown in the plan. By default, it runs a plan first and asks for confirmation. `-auto-approve` skips the prompt, useful in CI/CD. `tfplan` applies the exact saved plan without re-planning. `-parallelism` controls how many resource operations run concurrently (default is 10).

### Destroy

```bash
terraform destroy                      # Destroy all managed resources
terraform destroy -auto-approve        # Skip confirmation
terraform destroy -target=aws_instance.web  # Destroy specific resource
```

`destroy` removes all resources defined in the configuration. It runs a plan first showing what will be destroyed, then prompts for confirmation. `-target` destroys only the specified resource and its dependents.

### Show

```bash
terraform show                         # Show current state (human-readable)
terraform show tfplan                  # Show saved plan details
terraform show -json                   # Output state as JSON
```

### Graph

```bash
terraform graph                        # Output dependency graph in DOT format
terraform graph | dot -Tpng > graph.png  # Generate visual graph (requires Graphviz)
```

`graph` produces a DOT-format dependency graph showing how resources relate to each other. Pipe it through `dot` (from Graphviz) to generate an image.

---

## Variables

Variables parameterize your configuration, making it reusable across environments.

### Declaring Variables

```hcl
# variables.tf

variable "region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Project = "myapp"
    ManagedBy = "terraform"
  }
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true                    # Hide from output
}
```

`description` documents the variable's purpose. `type` enforces the data type. `default` provides a fallback value; variables without a default are required. `sensitive` prevents the value from appearing in logs and plan output.

### Variable Types

| Type               | Example                                   |
| ------------------ | ----------------------------------------- |
| `string`           | `"hello"`                                 |
| `number`           | `42`                                      |
| `bool`             | `true`                                    |
| `list(string)`     | `["a", "b", "c"]`                         |
| `set(string)`      | Unordered unique values                   |
| `map(string)`      | `{ key = "value" }`                       |
| `object({...})`    | Structured type with named attributes     |
| `tuple([...])`     | Fixed-length list with typed elements     |
| `any`              | Accepts any type                          |

### Complex Types

```hcl
variable "server_config" {
  type = object({
    name          = string
    instance_type = string
    disk_size     = number
    tags          = map(string)
  })
  default = {
    name          = "web"
    instance_type = "t3.micro"
    disk_size     = 20
    tags          = { Env = "dev" }
  }
}

variable "servers" {
  type = list(object({
    name = string
    port = number
  }))
  default = [
    { name = "api", port = 8080 },
    { name = "web", port = 80 }
  ]
}
```

`object` defines a structured type where each attribute has a name and type. `list(object({...}))` is a list of structured objects — ideal for defining multiple similar resources.

### Validation

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"

  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}

variable "environment" {
  type = string

  validation {
    condition     = can(regex("^(dev|staging|prod)$", var.environment))
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

`validation` blocks define custom rules. `condition` must evaluate to `true` for the value to be accepted. `contains()` checks if a value exists in a list. `can(regex(...))` returns `true` if the regex matches.

### Setting Variable Values

Variables can be set in multiple ways, listed in order of precedence (highest first):

```bash
# 1. Command-line flags (highest precedence)
terraform apply -var "region=us-west-2"
terraform apply -var-file="prod.tfvars"

# 2. terraform.tfvars or *.auto.tfvars (auto-loaded)
# 3. TF_VAR_ environment variables
export TF_VAR_region="us-west-2"
export TF_VAR_db_password="secretpassword"

# 4. Default value in variable declaration (lowest precedence)
```

### Variable Files

```hcl
# terraform.tfvars (auto-loaded)
region         = "us-east-1"
instance_type  = "t3.small"
instance_count = 3
tags = {
  Project = "myapp"
  Env     = "production"
}

# prod.tfvars (loaded with -var-file)
region         = "us-west-2"
instance_type  = "t3.medium"
instance_count = 5
```

`terraform.tfvars` and `*.auto.tfvars` files are automatically loaded. Any other `.tfvars` file must be specified with `-var-file`. Never commit files containing secrets to version control.

### Using Variables

```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = var.instance_type
  tags          = var.tags
}
```

Access variable values with `var.<variable_name>`.

---

## Outputs

Outputs expose values from your Terraform configuration, making them visible after `apply` and accessible to other configurations or scripts.

### Declaring Outputs

```hcl
# outputs.tf

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "instance_url" {
  description = "URL to access the web server"
  value       = "http://${aws_instance.web.public_ip}:8080"
}

output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive   = true
}

output "all_instance_ips" {
  description = "All instance public IPs"
  value       = aws_instance.web[*].public_ip
}
```

`value` is the expression to output. `sensitive` hides the value from the CLI output but still stores it in state. `[*]` is the splat expression — it collects an attribute from all instances when using `count` or `for_each`.

### Querying Outputs

```bash
terraform output                       # Show all outputs
terraform output instance_public_ip    # Show a specific output
terraform output -json                 # All outputs as JSON
terraform output -raw instance_public_ip  # Raw value (no quotes)
```

`-raw` strips quotes from string values, useful for piping into other commands. `-json` outputs in JSON format for programmatic consumption.

---

## Data Sources

Data sources let you fetch information from existing infrastructure or external services without managing them. They are read-only.

### Syntax

```hcl
data "<provider>_<type>" "<local_name>" {
  # Query arguments
}
```

### Examples

```hcl
# Look up the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Look up an existing VPC by tag
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# Look up availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Look up current AWS account info
data "aws_caller_identity" "current" {}

# Read a local file
data "local_file" "config" {
  filename = "${path.module}/config.json"
}

# Render a template
data "template_file" "user_data" {
  template = file("${path.module}/user_data.sh.tpl")
  vars = {
    server_name = var.server_name
    environment = var.environment
  }
}
```

Data sources are accessed with `data.<type>.<name>.<attribute>`:

```hcl
resource "aws_instance" "web" {
  ami               = data.aws_ami.amazon_linux.id
  instance_type     = "t3.micro"
  availability_zone = data.aws_availability_zones.available.names[0]
  subnet_id         = data.aws_vpc.existing.id
}
```

Data sources are refreshed on every `plan` and `apply`. They query the live API to get current values.

---

## State Management

Terraform state is a JSON file that maps your configuration to the real-world resources it manages. It is critical for Terraform to function correctly.

### State Commands

```bash
terraform state list                   # List all resources in state
terraform state show aws_instance.web  # Show details of a resource
terraform state pull                   # Download remote state to stdout
terraform state push                   # Upload local state to remote backend

terraform state mv aws_instance.web aws_instance.app
# Rename a resource in state (when you rename it in config)

terraform state rm aws_instance.web
# Remove resource from state (Terraform stops managing it)

terraform state replace-provider hashicorp/aws registry.example.com/aws
# Change the provider for all resources
```

`state list` shows every resource Terraform is tracking. `state show` displays the full attributes of a specific resource. `state mv` renames a resource in state so Terraform knows the renamed config still maps to the same infrastructure. `state rm` removes a resource from Terraform's management without destroying it.

### Viewing State

```bash
terraform show                         # Human-readable state
terraform show -json | jq             # JSON state, piped to jq for readability
```

### State Locking

When using a remote backend (S3 + DynamoDB, Azure Blob, etc.), Terraform acquires a lock before writing state. This prevents concurrent operations from corrupting state.

```bash
terraform force-unlock <LOCK_ID>       # Manually release a stuck lock
```

Only use `force-unlock` when you are sure no other operation is running. A stuck lock usually means a previous `apply` crashed or was interrupted.

### State File Security

The state file contains sensitive data (resource IDs, IP addresses, database passwords, etc.). Never commit `terraform.tfstate` to version control. Use a remote backend with encryption:

```gitignore
# .gitignore
*.tfstate
*.tfstate.backup
*.tfvars
.terraform/
```

---

## Modules

Modules are reusable packages of Terraform configuration. Every Terraform configuration is technically a module — the root module. Child modules are called from the root module to organize and reuse code.

### Module Structure

```
modules/
└── ec2-instance/
    ├── main.tf          # Resources
    ├── variables.tf     # Input variables
    ├── outputs.tf       # Output values
    └── README.md        # Documentation
```

### Creating a Module

```hcl
# modules/ec2-instance/variables.tf
variable "instance_name" {
  type        = string
  description = "Name tag for the instance"
}

variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "ami_id" {
  type        = string
  description = "AMI ID for the instance"
}

variable "subnet_id" {
  type        = string
  description = "Subnet to launch the instance in"
}
```

```hcl
# modules/ec2-instance/main.tf
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = {
    Name = var.instance_name
  }
}
```

```hcl
# modules/ec2-instance/outputs.tf
output "instance_id" {
  value = aws_instance.this.id
}

output "public_ip" {
  value = aws_instance.this.public_ip
}
```

### Using a Module

```hcl
# main.tf (root module)
module "web_server" {
  source = "./modules/ec2-instance"

  instance_name = "web-server"
  instance_type = "t3.small"
  ami_id        = data.aws_ami.amazon_linux.id
  subnet_id     = aws_subnet.public.id
}

module "api_server" {
  source = "./modules/ec2-instance"

  instance_name = "api-server"
  instance_type = "t3.medium"
  ami_id        = data.aws_ami.amazon_linux.id
  subnet_id     = aws_subnet.private.id
}

# Access module outputs
output "web_ip" {
  value = module.web_server.public_ip
}
```

`source` specifies where the module lives. Module outputs are accessed with `module.<module_name>.<output_name>`.

### Module Sources

```hcl
# Local path
source = "./modules/ec2-instance"

# Terraform Registry
source  = "terraform-aws-modules/vpc/aws"
version = "~> 5.0"

# GitHub
source = "github.com/hashicorp/example"

# S3 bucket
source = "s3::https://s3-eu-west-1.amazonaws.com/bucket/module.zip"

# Git repository
source = "git::https://example.com/module.git?ref=v1.0.0"
```

The Terraform Registry at [registry.terraform.io](https://registry.terraform.io/) hosts thousands of community and verified modules. `version` constraints work the same as provider version constraints.

### Module Best Practices

- Keep modules focused on a single logical component.
- Expose only necessary variables and outputs.
- Use descriptive variable names with `description` and `type`.
- Pin module versions in production (`version = "~> 5.0"`).
- Document modules with a `README.md`.
- Use `terraform-<provider>-<name>` naming convention for published modules.

---

## Provisioners

Provisioners execute scripts or commands on a local machine or remote resource after creation. They are a last resort — prefer cloud-native solutions (user data, configuration management tools) when possible.

### Local-Exec

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }

  provisioner "local-exec" {
    command     = "ansible-playbook -i '${self.public_ip},' setup.yml"
    working_dir = "./ansible"
  }
}
```

`local-exec` runs a command on the machine where Terraform is running (your workstation or CI server). `self` refers to the parent resource. `working_dir` sets the working directory for the command.

### Remote-Exec

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
  key_name      = "my-key-pair"

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/my-key.pem")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y nginx",
      "sudo systemctl start nginx"
    ]
  }
}
```

`remote-exec` runs commands on the remote resource via SSH or WinRM. The `connection` block defines how to connect. `inline` is a list of commands to execute sequentially. Alternatively, use `script` to upload and run a local script file.

### File Provisioner

```hcl
provisioner "file" {
  source      = "app/config.json"
  destination = "/home/ec2-user/config.json"
}

provisioner "file" {
  content     = templatefile("${path.module}/config.tpl", { port = 8080 })
  destination = "/etc/app/config.json"
}
```

`file` copies files or directories from the local machine to the remote resource. `source` specifies the local file. `content` provides inline content instead of a file. Both require a `connection` block.

### Destroy-Time Provisioner

```hcl
provisioner "local-exec" {
  when    = destroy
  command = "echo 'Destroying ${self.id}' >> destroy.log"
}
```

`when = destroy` runs the provisioner before the resource is destroyed instead of after creation.

### Failure Behavior

```hcl
provisioner "local-exec" {
  command    = "./setup.sh"
  on_failure = continue                # Don't fail the apply if this errors
}
```

`on_failure = continue` allows Terraform to proceed even if the provisioner fails. The default is `fail`, which causes the entire apply to abort.

---

## Functions & Expressions

Terraform includes a library of built-in functions for transforming and combining values.

### String Functions

```hcl
upper("hello")                         # → "HELLO"
lower("HELLO")                         # → "hello"
title("hello world")                   # → "Hello World"
trim("  hello  ")                      # → "hello"
trimprefix("helloworld", "hello")      # → "world"
trimsuffix("helloworld", "world")      # → "hello"
replace("hello world", "world", "HCL") # → "hello HCL"
split(",", "a,b,c")                    # → ["a", "b", "c"]
join("-", ["a", "b", "c"])             # → "a-b-c"
format("Hello, %s! You are %d.", "Bidur", 25)  # → "Hello, Bidur! You are 25."
substr("hello", 0, 3)                  # → "hel"
regex("^([a-z]+)-([0-9]+)$", "app-42") # → ["app", "42"]
regexall("[0-9]+", "abc123def456")     # → ["123", "456"]
```

### Numeric Functions

```hcl
abs(-5)                                # → 5
ceil(4.2)                              # → 5
floor(4.8)                             # → 4
max(1, 5, 3)                           # → 5
min(1, 5, 3)                           # → 1
parseint("FF", 16)                     # → 255
pow(2, 10)                             # → 1024
signum(-5)                             # → -1
```

### Collection Functions

```hcl
length(["a", "b", "c"])                # → 3
contains(["a", "b"], "a")              # → true
concat(["a"], ["b", "c"])              # → ["a", "b", "c"]
flatten([["a", "b"], ["c"]])           # → ["a", "b", "c"]
distinct(["a", "b", "a"])              # → ["a", "b"]
sort(["c", "a", "b"])                  # → ["a", "b", "c"]
reverse(["a", "b", "c"])              # → ["c", "b", "a"]
slice(["a", "b", "c", "d"], 1, 3)     # → ["b", "c"]
element(["a", "b", "c"], 1)            # → "b"
index(["a", "b", "c"], "b")           # → 1
compact(["a", "", "b", ""])            # → ["a", "b"]
coalesce("", "hello")                  # → "hello" (first non-empty)
zipmap(["a", "b"], [1, 2])             # → { a = 1, b = 2 }
lookup({a = 1, b = 2}, "a", 0)         # → 1 (default 0 if not found)
keys({a = 1, b = 2})                   # → ["a", "b"]
values({a = 1, b = 2})                 # → [1, 2]
merge({a = 1}, {b = 2, c = 3})         # → {a = 1, b = 2, c = 3}
```

### Filesystem Functions

```hcl
file("${path.module}/script.sh")       # Read file contents as string
fileexists("${path.module}/config.json") # → true/false
templatefile("${path.module}/user_data.sh.tpl", {
  name = var.name
  port = var.port
})
basename("/path/to/file.txt")          # → "file.txt"
dirname("/path/to/file.txt")           # → "/path/to"
abspath("relative/path")              # → absolute path
```

`file()` reads a file and returns its contents as a string. `templatefile()` reads a file and substitutes variables using `${...}` syntax in the template.

### Encoding Functions

```hcl
jsonencode({name = "Bidur", age = 25}) # → JSON string
jsondecode("{\"name\":\"Bidur\"}")     # → HCL object
yamlencode({name = "Bidur"})           # → YAML string
yamldecode("name: Bidur")             # → HCL object
base64encode("hello")                  # → "aGVsbG8="
base64decode("aGVsbG8=")              # → "hello"
csvdecode(file("data.csv"))           # → list of maps
```

### Type Conversion

```hcl
tostring(42)                           # → "42"
tonumber("42")                         # → 42
tobool("true")                         # → true
tolist(toset(["a", "b"]))             # → ["a", "b"]
toset(["a", "b", "a"])                # → set of "a", "b"
tomap({a = 1})                         # → map
try(var.optional_value, "default")     # → value or default if error
can(regex("^[a-z]+$", var.input))      # → true if expression succeeds
```

`try()` evaluates expressions left to right and returns the first one that does not error. `can()` returns `true` if the expression evaluates without error, `false` otherwise.

### Path References

```hcl
path.module                            # Directory of the current module
path.root                              # Directory of the root module
path.cwd                               # Current working directory
terraform.workspace                    # Current workspace name
```

---

## Conditionals & Loops

### Conditional Expression

```hcl
# condition ? true_value : false_value
instance_type = var.environment == "prod" ? "t3.large" : "t3.micro"

enable_logging = var.environment != "dev" ? true : false

subnet_id = var.is_public ? aws_subnet.public.id : aws_subnet.private.id
```

The ternary syntax `condition ? true_val : false_val` is the only conditional expression in HCL.

### Count

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"

  tags = {
    Name = "web-${count.index}"        # web-0, web-1, web-2
  }
}

# Access specific instance
output "first_ip" {
  value = aws_instance.web[0].public_ip
}

# Access all instances
output "all_ips" {
  value = aws_instance.web[*].public_ip
}
```

`count` creates multiple instances of a resource. `count.index` is the 0-based index of the current instance. Resources with `count` are accessed as a list: `resource[0]`, `resource[1]`, etc. `[*]` is the splat expression that collects a single attribute from all instances.

### Conditional Resource with Count

```hcl
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  count = var.enable_monitoring ? 1 : 0

  alarm_name = "high-cpu"
  # ... configuration ...
}
```

Setting `count = 0` prevents the resource from being created. This is the standard pattern for conditionally creating resources.

### for_each

```hcl
# With a set of strings
resource "aws_iam_user" "users" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.value
}

# With a map
resource "aws_instance" "servers" {
  for_each = {
    web = { type = "t3.small", az = "us-east-1a" }
    api = { type = "t3.medium", az = "us-east-1b" }
    db  = { type = "t3.large", az = "us-east-1c" }
  }

  ami               = "ami-0abcdef1234567890"
  instance_type     = each.value.type
  availability_zone = each.value.az

  tags = {
    Name = each.key
  }
}

# Access specific instance
output "web_ip" {
  value = aws_instance.servers["web"].public_ip
}
```

`for_each` accepts a map or a set. `each.key` is the map key (or set value). `each.value` is the map value (or same as key for sets). Resources with `for_each` are accessed by key: `resource["web"]`. Unlike `count`, adding or removing items does not affect unrelated resources.

### Count vs for_each

| Feature              | `count`                     | `for_each`                   |
| -------------------- | --------------------------- | ---------------------------- |
| Argument Type        | Number                      | Map or set                   |
| Access               | By index (`[0]`, `[1]`)     | By key (`["web"]`)           |
| Add/Remove Behavior  | Shifts indices, causes churn | Only affects the changed key |
| Best For             | Identical resources          | Distinct resources           |

Prefer `for_each` when each instance has a meaningful identity (name, role, etc.). Use `count` only for truly identical copies.

### for Expressions

```hcl
# Transform a list
upper_names = [for name in var.names : upper(name)]
# → ["ALICE", "BOB", "CAROL"]

# Filter a list
short_names = [for name in var.names : name if length(name) <= 4]
# → ["BOB"]

# Transform a map
instance_ids = { for k, v in aws_instance.servers : k => v.id }
# → { web = "i-123", api = "i-456", db = "i-789" }

# List to map
name_map = { for name in var.names : name => upper(name) }
# → { alice = "ALICE", bob = "BOB" }

# Nested for
all_ports = flatten([
  for server in var.servers : [
    for port in server.ports : {
      name = server.name
      port = port
    }
  ]
])
```

`for` expressions transform collections. `[for ... : ...]` produces a list (square brackets). `{for ... : ... => ...}` produces a map (curly braces). `if` at the end filters elements.

### Dynamic Blocks

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { port = 80, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { port = 22, protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] }
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

`dynamic` blocks generate repeated nested blocks from a collection. The block label (`"ingress"`) must match the nested block type. Inside `content`, access the current element with `<label>.value` and its key/index with `<label>.key`.

---

## Workspaces

Workspaces let you manage multiple environments (dev, staging, prod) from the same configuration with separate state files.

### Workspace Commands

```bash
terraform workspace list               # List all workspaces (* = current)
terraform workspace new dev            # Create and switch to "dev"
terraform workspace new staging        # Create "staging"
terraform workspace new prod           # Create "prod"
terraform workspace select dev         # Switch to "dev"
terraform workspace show               # Show current workspace
terraform workspace delete staging     # Delete a workspace
```

Each workspace has its own state file. The `default` workspace always exists and cannot be deleted.

### Using Workspaces in Configuration

```hcl
locals {
  environment = terraform.workspace

  instance_types = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }

  instance_counts = {
    dev     = 1
    staging = 2
    prod    = 3
  }
}

resource "aws_instance" "web" {
  count         = local.instance_counts[local.environment]
  ami           = "ami-0abcdef1234567890"
  instance_type = local.instance_types[local.environment]

  tags = {
    Name        = "web-${local.environment}-${count.index}"
    Environment = local.environment
  }
}
```

`terraform.workspace` returns the name of the current workspace. Use it to look up environment-specific values from maps. This avoids duplicating configuration across environments.

---

## Import Existing Infrastructure

Terraform can import existing resources that were created outside of Terraform into its state and management.

### Import Command

```bash
# terraform import <resource_address> <resource_id>
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_s3_bucket.assets my-bucket-name
terraform import aws_vpc.main vpc-0abc123def456
terraform import 'aws_security_group.web' sg-0abc123
```

`import` adds the resource to the state file but does **not** generate configuration. You must write the corresponding `resource` block in your `.tf` files manually to match the imported resource's settings.

### Import Workflow

```bash
# 1. Write the resource block in your .tf file (even if incomplete)
# 2. Import the resource
terraform import aws_instance.web i-1234567890abcdef0

# 3. Run plan to see differences
terraform plan

# 4. Update your .tf file to match the actual resource
# 5. Run plan again — it should show no changes
terraform plan
```

### Import Block (Terraform 1.5+)

```hcl
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

# Generate config automatically (Terraform 1.5+)
# terraform plan -generate-config-out=generated.tf
```

The `import` block is a declarative alternative to the CLI command. `terraform plan -generate-config-out` can auto-generate the resource configuration for imported resources, which you then review and refine.

### Importing with for_each

```hcl
import {
  to = aws_iam_user.users["alice"]
  id = "alice"
}

import {
  to = aws_iam_user.users["bob"]
  id = "bob"
}
```

---

## Backend Configuration

Backends define where Terraform stores its state. The default is local (a file on disk), but production setups should use a remote backend.

### Local Backend (Default)

```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

State is stored as a file on disk. Simple for learning but not suitable for teams.

### S3 Backend (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

`bucket` is the S3 bucket for state storage. `key` is the path within the bucket. `encrypt` enables server-side encryption. `dynamodb_table` enables state locking using a DynamoDB table, preventing concurrent modifications.

### Create S3 Backend Resources

```hcl
# bootstrap/main.tf — run this first to create backend resources

resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Versioning on the S3 bucket lets you recover previous state versions. The DynamoDB table provides state locking. Public access block prevents accidental exposure.

### Azure Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}
```

### GCS Backend (Google Cloud)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "terraform/state"
  }
}
```

### Migrating Backends

```bash
# Update backend config in .tf files, then:
terraform init -migrate-state          # Migrate state to new backend
```

Terraform will ask for confirmation before copying the state to the new backend.

---

## Best Practices

### Project Structure

```
project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   └── ...
│   └── database/
│       └── ...
└── README.md
```

Separate environments into their own directories, each with its own state and backend. Shared logic lives in modules.

### Naming Conventions

```hcl
# Resources: descriptive, lowercase, underscores
resource "aws_instance" "web_server" { ... }
resource "aws_s3_bucket" "app_assets" { ... }

# Variables: lowercase, underscores, descriptive
variable "instance_type" { ... }
variable "enable_monitoring" { ... }

# Outputs: lowercase, underscores
output "instance_public_ip" { ... }

# Modules: lowercase, hyphens in directory names
module "web_server" {
  source = "./modules/ec2-instance"
}

# Tags: consistent key casing
tags = {
  Name        = "web-server"
  Environment = "production"
  ManagedBy   = "terraform"
  Project     = "myapp"
}
```

### Security

- **Never commit secrets** to version control. Use environment variables, vault, or encrypted variable files.
- **Use remote state** with encryption and locking for team environments.
- **Restrict state access** — the state file contains sensitive data.
- **Use `sensitive = true`** on variables and outputs containing secrets.
- **Pin provider and module versions** to avoid unexpected changes.
- **Enable S3 versioning** on state buckets for recovery.

### Operational

- **Always run `plan` before `apply`** and review the output carefully.
- **Use `-out` to save plans** in CI/CD to ensure what you reviewed is what gets applied.
- **Tag all resources** consistently for cost tracking and organization.
- **Use modules** for reusable infrastructure patterns.
- **Use `for_each` over `count`** when each instance has a distinct identity.
- **Keep state files small** by breaking large projects into smaller, focused configurations.
- **Use `terraform fmt`** and `terraform validate` in CI pipelines.
- **Avoid provisioners** when possible; use cloud-native tools instead.
- **Use `locals`** to reduce repetition and improve readability.
- **Document your infrastructure** with comments, READMEs, and meaningful variable descriptions.

### Common `.gitignore`

```gitignore
# Terraform state
*.tfstate
*.tfstate.backup
*.tfstate.*.backup

# Terraform directories
.terraform/

# Variable files with secrets
*.tfvars
!example.tfvars

# Crash logs
crash.log
crash.*.log

# Plan files
*.tfplan

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Lock file (DO commit .terraform.lock.hcl)
# .terraform.lock.hcl
```

Commit `.terraform.lock.hcl` to ensure consistent provider versions across the team. Do not commit state files, variable files with secrets, or the `.terraform` directory.
