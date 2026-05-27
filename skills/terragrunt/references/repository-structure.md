# Repository Structure

## Table of Contents

- [Two-Repo Pattern](#two-repo-pattern)
- [Directory Layout Conventions](#directory-layout-conventions)
- [Root Configuration (root.hcl)](#root-configuration-roothcl)
- [Hierarchical Configuration with Includes](#hierarchical-configuration-with-includes)
- [Environment-Specific Patterns with _env Directory](#environment-specific-patterns-with-_env-directory)
- [Multi-Account Patterns](#multi-account-patterns)
- [Infrastructure Modules Repo Structure](#infrastructure-modules-repo-structure)
- [Working Locally with Source Overrides](#working-locally-with-source-overrides)
- [.gitignore](#gitignore)

## Two-Repo Pattern

Terragrunt projects follow a two-repository pattern that separates **live infrastructure configuration** from **reusable modules**:

- **`infrastructure-live`** contains `terragrunt.hcl` files that define what infrastructure to deploy, where to deploy it, and with what inputs. This is the "live" configuration that represents the actual state of each environment.

- **`infrastructure-modules`** contains `.tf` files organized as reusable OpenTofu/Terraform modules. Each module encapsulates a logical piece of infrastructure (a VPC, an RDS instance, an ECS service, etc.).

**Why two repos:** Modules are versioned and tested independently of the live configurations. The infrastructure-live repo pins specific module versions via git ref tags, so a module upgrade in one environment does not automatically affect others. This enables safe promotion workflows: update the ref tag in dev, test, then update staging, then prod.

### How it works together

A module defines its interface with `variables.tf`:

```hcl
# infrastructure-modules/networking/vpc/variables.tf
variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "enable_dns_support" {
  description = "Enable DNS support in the VPC"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

A `terragrunt.hcl` in infrastructure-live references the module via a git URL with a `ref` tag and provides the variable values through `inputs`:

```hcl
# infrastructure-live/prod/us-east-1/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "git::git@github.com:acme/infrastructure-modules.git//networking/vpc?ref=v1.2.0"
}

inputs = {
  vpc_name           = "prod-vpc"
  cidr_block         = "10.0.0.0/16"
  enable_dns_support = true
}
```

The `//` in the source URL separates the repo URL from the subdirectory path within the repo. The `?ref=v1.2.0` pins to a specific git tag.

## Directory Layout Conventions

The infrastructure-live repo uses a hierarchical directory structure that mirrors the deployment topology: account, then region, then resource.

### Complete tree for multi-account AWS

```
infrastructure-live/
├── root.hcl                          # Root config included by all units
├── _env/                             # Shared environment configs (not in the hierarchy)
│   ├── app.hcl
│   ├── database.hcl
│   └── vpc.hcl
├── prod/
│   ├── account.hcl                   # Prod account-level locals
│   ├── us-east-1/
│   │   ├── region.hcl                # Region-level locals
│   │   ├── vpc/
│   │   │   └── terragrunt.hcl
│   │   ├── app/
│   │   │   └── terragrunt.hcl
│   │   └── rds/
│   │       └── terragrunt.hcl
│   └── eu-west-1/
│       ├── region.hcl
│       ├── vpc/
│       │   └── terragrunt.hcl
│       └── app/
│           └── terragrunt.hcl
├── staging/
│   ├── account.hcl
│   └── us-east-1/
│       ├── region.hcl
│       ├── vpc/
│       │   └── terragrunt.hcl
│       ├── app/
│       │   └── terragrunt.hcl
│       └── rds/
│           └── terragrunt.hcl
└── dev/
    ├── account.hcl
    └── us-east-1/
        ├── region.hcl
        ├── vpc/
        │   └── terragrunt.hcl
        ├── app/
        │   └── terragrunt.hcl
        └── rds/
            └── terragrunt.hcl
```

### Naming conventions

- **Directory names mirror the infrastructure purpose**, not the module name. Use `vpc/`, `app/`, `rds/` rather than `terraform-aws-vpc/` or `module-rds/`.
- **Account directories** use the environment name: `prod/`, `staging/`, `dev/`. For AWS Organizations with many accounts, use descriptive names: `security/`, `logging/`, `shared-services/`.
- **Region directories** use the cloud provider's region identifier: `us-east-1/`, `eu-west-1/`, `us-central1/`.
- **Each leaf directory** contains exactly one `terragrunt.hcl` file representing a single deployable unit.
- **Non-unit `.hcl` files** (`account.hcl`, `region.hcl`) sit alongside the directories they apply to and are never inside a leaf directory.

## Root Configuration (root.hcl)

The root configuration file is placed at the repository root and included by every unit via `find_in_parent_folders("root.hcl")`. It centralizes settings that apply across all environments, accounts, and regions.

**Why `root.hcl` and not `terragrunt.hcl`:** Terragrunt's `root-terragrunt-hcl` strict control enforces that the repo root file is named `root.hcl`. A file named `terragrunt.hcl` at the repo root is ambiguous -- Terragrunt cannot tell if it is a root configuration or an executable unit. Using `root.hcl` eliminates this ambiguity.

### Full example root.hcl

```hcl
# infrastructure-live/root.hcl

# Load account-level and region-level variables from the hierarchy
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}

# Configure remote state storage in S3
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "acme-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Generate the AWS provider configuration
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.aws_region}"

      default_tags {
        tags = {
          ManagedBy   = "terragrunt"
          Account     = "${local.account_name}"
          Environment = "${local.account_name}"
        }
      }
    }
  EOF
}

# Common inputs applied to all units
inputs = {
  aws_region   = local.aws_region
  account_id   = local.account_id
  account_name = local.account_name
}
```

Key points about `root.hcl`:

- **`path_relative_to_include()`** returns the relative path from `root.hcl` down to the including unit. For a unit at `prod/us-east-1/vpc/terragrunt.hcl`, this returns `prod/us-east-1/vpc`, which becomes the S3 state key -- guaranteeing unique state paths.
- **`remote_state` with `generate`** creates a `backend.tf` file in the working directory so OpenTofu/Terraform sees a properly configured backend.
- **`generate "provider"`** creates a `provider.tf` file, centralizing provider configuration so individual units do not repeat it.
- **`inputs`** defined here are merged with inputs from child configurations. Child inputs take precedence.

## Hierarchical Configuration with Includes

Terragrunt supports a layered configuration pattern where each level of the directory hierarchy contributes context-specific values:

```
root.hcl          → remote state, provider, common tags
  account.hcl     → account ID, account name, IAM role
    region.hcl    → AWS region
      terragrunt.hcl → module source, resource-specific inputs
```

### account.hcl

Placed in each account directory. Contains account-level locals:

```hcl
# infrastructure-live/prod/account.hcl
locals {
  account_name = "prod"
  account_id   = "123456789012"
}
```

```hcl
# infrastructure-live/staging/account.hcl
locals {
  account_name = "staging"
  account_id   = "234567890123"
}
```

```hcl
# infrastructure-live/dev/account.hcl
locals {
  account_name = "dev"
  account_id   = "345678901234"
}
```

### region.hcl

Placed in each region directory. Contains region-level locals:

```hcl
# infrastructure-live/prod/us-east-1/region.hcl
locals {
  aws_region = "us-east-1"
}
```

```hcl
# infrastructure-live/prod/eu-west-1/region.hcl
locals {
  aws_region = "eu-west-1"
}
```

### How units read hierarchical configuration

Each `terragrunt.hcl` unit reads the parent configs using `read_terragrunt_config` and `find_in_parent_folders`:

```hcl
# infrastructure-live/prod/us-east-1/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}

terraform {
  source = "git::git@github.com:acme/infrastructure-modules.git//networking/vpc?ref=v1.2.0"
}

inputs = {
  vpc_name   = "${local.account_name}-vpc"
  cidr_block = "10.0.0.0/16"
  aws_region = local.aws_region
}
```

`find_in_parent_folders("account.hcl")` searches upward from the current `terragrunt.hcl` and returns the absolute path to the first `account.hcl` it finds. `read_terragrunt_config` parses that file and makes its `locals` block available.

This pattern means adding a new region or account requires only creating the appropriate directory with its `.hcl` file and copying or linking the unit directories -- no per-unit edits needed.

## Environment-Specific Patterns with _env Directory

When multiple environments deploy the same type of resource with the same module and mostly the same inputs, the `_env/` directory pattern eliminates duplication. The `_env/` directory sits at the repo root (or another shared location) outside the account/region hierarchy.

### Example _env/app.hcl

```hcl
# infrastructure-live/_env/app.hcl
terraform {
  source = "git::git@github.com:acme/infrastructure-modules.git//app/ecs-service?ref=v2.1.0"
}

locals {
  # Defaults that units can override
  container_port = 8080
  health_check_path = "/health"
}

inputs = {
  container_port    = local.container_port
  health_check_path = local.health_check_path
}
```

### Units include both root.hcl and the _env file

```hcl
# infrastructure-live/prod/us-east-1/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/app.hcl"
  expose = true
}

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  account_name = local.account_vars.locals.account_name
}

dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1", "mock-subnet-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  app_name        = "${local.account_name}-web-app"
  instance_count  = 3
  vpc_id          = dependency.vpc.outputs.vpc_id
  subnet_ids      = dependency.vpc.outputs.private_subnets
}
```

Key points:

- **`expose = true`** on the `include "env"` block makes the included config's locals accessible as `include.env.locals.*` if needed.
- **`get_terragrunt_dir()`** returns the directory of the current `terragrunt.hcl` file (e.g., `prod/us-east-1/app/`). The relative path `../../_env/app.hcl` navigates up to the repo root's `_env/` directory.
- The `terraform.source` from `_env/app.hcl` is inherited, so units do not repeat it. When upgrading the module version, change it once in `_env/app.hcl`.
- **`inputs` are deep-merged** by default when using `include`. The unit's `inputs` override or extend those from `_env/app.hcl`.
- A dev unit might override the instance count:

```hcl
# infrastructure-live/dev/us-east-1/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/app.hcl"
  expose = true
}

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  account_name = local.account_vars.locals.account_name
}

dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  app_name       = "${local.account_name}-web-app"
  instance_count = 1  # Smaller for dev
  vpc_id         = dependency.vpc.outputs.vpc_id
  subnet_ids     = dependency.vpc.outputs.private_subnets
}
```

## Multi-Account Patterns

### AWS Organizations with per-account IAM role assumption

Each account directory specifies the IAM role Terragrunt should assume. The root config reads this and configures role assumption:

```hcl
# infrastructure-live/prod/account.hcl
locals {
  account_name = "prod"
  account_id   = "123456789012"
  iam_role     = "arn:aws:iam::123456789012:role/TerragruntDeployRole"
}
```

```hcl
# infrastructure-live/staging/account.hcl
locals {
  account_name = "staging"
  account_id   = "234567890123"
  iam_role     = "arn:aws:iam::234567890123:role/TerragruntDeployRole"
}
```

The root config uses the `iam_role` top-level attribute to assume the correct role:

```hcl
# infrastructure-live/root.hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}

# Assume the IAM role for the target account
iam_role = local.account_vars.locals.iam_role

remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "acme-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.aws_region}"

      assume_role {
        role_arn = "${local.account_vars.locals.iam_role}"
      }

      default_tags {
        tags = {
          ManagedBy   = "terragrunt"
          Account     = "${local.account_name}"
          Environment = "${local.account_name}"
        }
      }
    }
  EOF
}

inputs = {
  aws_region   = local.aws_region
  account_id   = local.account_id
  account_name = local.account_name
}
```

With this setup, running `terragrunt run --all plan` from the repo root plans across all accounts, each unit automatically assuming the correct IAM role based on which account directory it sits in.

### Deploying a single account or region

```bash
# Plan everything in the prod account
cd prod && terragrunt run --all plan

# Plan only the us-east-1 region in prod
cd prod/us-east-1 && terragrunt run --all plan

# Plan a single unit
cd prod/us-east-1/vpc && terragrunt run plan
```

## Infrastructure Modules Repo Structure

The infrastructure-modules repository contains reusable OpenTofu/Terraform modules, each organized as a self-contained directory with standard files.

### Directory tree

```
infrastructure-modules/
├── networking/
│   └── vpc/
│       ├── main.tf           # Resource definitions
│       ├── variables.tf      # Input variable declarations
│       ├── outputs.tf        # Output values
│       └── versions.tf       # Required providers and version constraints
├── compute/
│   ├── ecs-service/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   └── ec2-instance/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── database/
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── storage/
│   └── s3-bucket/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
└── monitoring/
    └── cloudwatch-alarms/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── versions.tf
```

### Standard module files

**`variables.tf`** -- declares all inputs the module accepts:

```hcl
# infrastructure-modules/networking/vpc/variables.tf
variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**`outputs.tf`** -- declares outputs that other Terragrunt units can consume via `dependency` blocks:

```hcl
# infrastructure-modules/networking/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "private_subnets" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnets" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}
```

**`versions.tf`** -- pins provider and OpenTofu/Terraform version requirements:

```hcl
# infrastructure-modules/networking/vpc/versions.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

### Versioning via git tags

Modules are versioned using git tags in the infrastructure-modules repo:

```bash
# Tag a new release
git tag -a v1.2.0 -m "Add NAT gateway support to VPC module"
git push origin v1.2.0
```

### Pinning versions in terragrunt.hcl

The `terraform.source` URL pins to a specific tag:

```hcl
terraform {
  # Pin to exact version
  source = "git::git@github.com:acme/infrastructure-modules.git//networking/vpc?ref=v1.2.0"
}
```

To upgrade a module, change the `ref` tag in the `terragrunt.hcl` (or in `_env/*.hcl` if using the environment pattern):

```hcl
terraform {
  # Upgrade from v1.2.0 to v1.3.0
  source = "git::git@github.com:acme/infrastructure-modules.git//networking/vpc?ref=v1.3.0"
}
```

## Working Locally with Source Overrides

During module development, you can override the remote git source to point at a local checkout of the infrastructure-modules repo. This avoids committing, pushing, and tagging just to test changes.

### Using the --source flag

```bash
# Override the source for a single unit
cd infrastructure-live/dev/us-east-1/vpc
terragrunt run plan --source ../../../infrastructure-modules//networking/vpc
```

The `//` separator is still required -- it tells Terragrunt which subdirectory within the source path contains the module.

### Using --source with run --all

```bash
# Override source for all units (each unit's terraform.source is replaced)
cd infrastructure-live/dev/us-east-1
terragrunt run --all plan --source ../../../infrastructure-modules
```

When using `--source` with `run --all`, Terragrunt appends each unit's module subdirectory path to the base source path automatically.

### Using TERRAGRUNT_SOURCE environment variable

```bash
# Set once, applies to all subsequent commands
export TERRAGRUNT_SOURCE="../../../infrastructure-modules"
cd infrastructure-live/dev/us-east-1
terragrunt run --all plan
```

## .gitignore

Every infrastructure-live repository must include a `.gitignore` that excludes Terragrunt and OpenTofu/Terraform generated files:

```gitignore
# Terragrunt cache -- downloaded modules and temporary files
.terragrunt-cache/

# Terragrunt stack generated files
.terragrunt-stack/

# OpenTofu/Terraform working directories
.terraform/

# State files (should be in remote backend, never committed)
*.tfstate
*.tfstate.*

# Crash logs
crash.log
crash.*.log

# Lock file for remote modules (regenerated on init)
# Only ignore this if all modules come from remote sources.
# If you use local modules or want reproducible builds, keep .terraform.lock.hcl tracked.
.terraform.lock.hcl

# Generated files from terragrunt generate blocks
backend.tf
provider.tf
```

The `.terragrunt-cache/` directory is where Terragrunt downloads remote modules and copies them for execution. It can grow large and must never be committed. The `.terragrunt-stack/` directory contains generated files when using explicit stacks. Both are safe to delete at any time -- Terragrunt recreates them on the next run.
