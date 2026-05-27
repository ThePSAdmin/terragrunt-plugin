# Repository Structure

## Table of Contents

- [Stacks + Catalog Pattern (Recommended)](#stacks--catalog-pattern-recommended)
  - [Infrastructure-Live Repo (Stacks)](#infrastructure-live-repo-stacks)
  - [Infrastructure-Catalog Repo](#infrastructure-catalog-repo)
  - [Three-Layer Catalog Architecture](#three-layer-catalog-architecture)
  - [The Values Interface](#the-values-interface)
  - [Version Pinning in Stacks](#version-pinning-in-stacks)
  - [Dependency Paths via Values](#dependency-paths-via-values)
  - [Reusable Stacks in the Catalog](#reusable-stacks-in-the-catalog)
- [Root Configuration (root.hcl)](#root-configuration-roothcl)
- [Hierarchical Configuration](#hierarchical-configuration)
- [Multi-Account Patterns](#multi-account-patterns)
- [Traditional Two-Repo Pattern (Implicit Stacks)](#traditional-two-repo-pattern-implicit-stacks)
  - [Directory Layout Conventions](#directory-layout-conventions)
  - [Environment-Specific Patterns with _env Directory](#environment-specific-patterns-with-_env-directory)
  - [Infrastructure Modules Repo Structure](#infrastructure-modules-repo-structure)
- [Working Locally with Source Overrides](#working-locally-with-source-overrides)
- [.gitignore](#gitignore)

## Stacks + Catalog Pattern (Recommended)

The recommended way to structure Terragrunt projects is with explicit stacks and an infrastructure catalog. This pattern uses two repositories:

- **`infrastructure-live`** contains `terragrunt.stack.hcl` files organized by account and region. Each stack declares which units to deploy and passes configuration via `values`. The stack files are thin -- they define *what* to deploy and *where*, not *how*.

- **`infrastructure-catalog`** contains reusable modules, units, and stacks organized in a three-layer architecture. The catalog is versioned via git tags and treated as an immutable artifact store. Consumers pin to specific versions.

This replaces the older two-repo pattern (infrastructure-live + infrastructure-modules) with a more composable approach where the catalog bundles both the Terraform modules and their Terragrunt wrappers together.

### Infrastructure-Live Repo (Stacks)

```
infrastructure-live/
├── root.hcl                          # Shared root config (remote state, provider)
├── non-prod/
│   ├── account.hcl                   # Account name, account ID
│   └── us-east-1/
│       ├── region.hcl               # AWS region
│       ├── web-service/
│       │   └── terragrunt.stack.hcl  # Stack definition
│       └── data-pipeline/
│           └── terragrunt.stack.hcl
├── prod/
│   ├── account.hcl
│   └── us-east-1/
│       ├── region.hcl
│       ├── web-service/
│       │   └── terragrunt.stack.hcl
│       └── data-pipeline/
│           └── terragrunt.stack.hcl
└── .gitignore
```

Each `terragrunt.stack.hcl` declares the units to deploy for that service/environment combination:

```hcl
# infrastructure-live/prod/us-east-1/web-service/terragrunt.stack.hcl

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  name         = "web-service"
  catalog_url  = "github.com/acme/infrastructure-catalog"
  catalog_ref  = "v1.2.0"
}

unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = {
    version    = local.catalog_ref
    name       = "${local.name}-vpc"
    cidr_block = "10.0.0.0/16"
    aws_region = local.region_vars.locals.aws_region
  }
}

unit "database" {
  source = "${local.catalog_url}//units/rds?ref=${local.catalog_ref}"
  path   = "database"
  values = {
    version        = local.catalog_ref
    name           = "${local.name}-db"
    instance_class = "db.r6g.xlarge"
    vpc_path       = "../vpc"
  }
}

unit "app" {
  source = "${local.catalog_url}//units/ecs-service?ref=${local.catalog_ref}"
  path   = "app"
  values = {
    version       = local.catalog_ref
    name          = local.name
    vpc_path      = "../vpc"
    database_path = "../database"
  }
}
```

The non-prod version of the same stack differs only in values (smaller instances, different naming):

```hcl
# infrastructure-live/non-prod/us-east-1/web-service/terragrunt.stack.hcl

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  name         = "web-service-dev"
  catalog_url  = "github.com/acme/infrastructure-catalog"
  catalog_ref  = "v1.2.0"
}

unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = {
    version    = local.catalog_ref
    name       = "${local.name}-vpc"
    cidr_block = "10.1.0.0/16"
    aws_region = local.region_vars.locals.aws_region
  }
}

unit "database" {
  source = "${local.catalog_url}//units/rds?ref=${local.catalog_ref}"
  path   = "database"
  values = {
    version        = local.catalog_ref
    name           = "${local.name}-db"
    instance_class = "db.t3.medium"
    vpc_path       = "../vpc"
  }
}

unit "app" {
  source = "${local.catalog_url}//units/ecs-service?ref=${local.catalog_ref}"
  path   = "app"
  values = {
    version       = local.catalog_ref
    name          = local.name
    vpc_path      = "../vpc"
    database_path = "../database"
  }
}
```

Running `terragrunt stack generate` in a stack directory creates a `.terragrunt-stack/` directory containing the materialized units. Then `terragrunt run --all plan` or `terragrunt run --all apply` operates on those generated units.

### Infrastructure-Catalog Repo

```
infrastructure-catalog/
├── modules/                          # Layer 1: Pure OpenTofu/Terraform modules
│   ├── networking/vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── database/rds/
│   ├── compute/ecs-service/
│   ├── compute/lambda-service/
│   ├── storage/s3-bucket/
│   ├── storage/dynamodb-table/
│   └── security/iam-role/
├── units/                            # Layer 2: Terragrunt units wrapping modules
│   ├── vpc/terragrunt.hcl
│   ├── rds/terragrunt.hcl
│   ├── ecs-service/terragrunt.hcl
│   ├── lambda-service/terragrunt.hcl
│   ├── dynamodb-table/terragrunt.hcl
│   └── lambda-iam-role/terragrunt.hcl
├── stacks/                           # Layer 3: Reusable stack compositions
│   └── web-service/
│       └── terragrunt.stack.hcl
├── root.hcl                          # Root config for units
├── examples/                         # Usage examples
│   └── terragrunt/
│       ├── root.hcl
│       ├── units/
│       └── stacks/
└── test/                             # Tests (e.g., Terratest)
    ├── tofu/
    └── terragrunt/
```

### Three-Layer Catalog Architecture

The catalog organizes code into three layers, each building on the one below:

**Layer 1: Modules (`modules/`)** -- Pure OpenTofu/Terraform modules with `variables.tf`, `outputs.tf`, `main.tf`. These have no Terragrunt-specific code and define the actual cloud resources. They are generic building blocks.

```hcl
# infrastructure-catalog/modules/database/rds/variables.tf
variable "name" {
  type = string
}

variable "instance_class" {
  type    = string
  default = "db.t3.medium"
}

variable "vpc_id" {
  type = string
}

variable "subnet_ids" {
  type = list(string)
}
```

**Layer 2: Units (`units/`)** -- Terragrunt `terragrunt.hcl` files that wrap modules with the `values.*` interface, dependency blocks, hooks, and business logic. Units are the primary building blocks consumed by stacks.

```hcl
# infrastructure-catalog/units/rds/terragrunt.hcl

include "root" {
  path = find_in_parent_folders("root.hcl")
}

dependency "vpc" {
  config_path = values.vpc_path
  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1", "mock-subnet-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

terraform {
  source = "github.com/acme/infrastructure-catalog//modules/database/rds?ref=${values.version}"
}

inputs = {
  name           = values.name
  instance_class = try(values.instance_class, "db.t3.medium")
  vpc_id         = dependency.vpc.outputs.vpc_id
  subnet_ids     = dependency.vpc.outputs.private_subnets
}
```

Units source modules via full git URLs (not relative paths) because when consumed by an external stack, only the unit directory is fetched -- it cannot resolve relative paths to sibling directories in the catalog. The `values.version` parameter enables version pinning from the stack.

**Layer 3: Stacks (`stacks/`)** -- Reusable `terragrunt.stack.hcl` files that compose multiple units into a deployable collection. Stacks in the catalog are parameterized via `values` and can be consumed by the infrastructure-live repo or by other stacks (stacks-of-stacks).

```hcl
# infrastructure-catalog/stacks/web-service/terragrunt.stack.hcl

unit "vpc" {
  source = "github.com/acme/infrastructure-catalog//units/vpc?ref=${values.version}"
  path   = "vpc"
  values = {
    version    = values.version
    name       = values.name
    cidr_block = values.cidr_block
    aws_region = values.aws_region
  }
}

unit "database" {
  source = "github.com/acme/infrastructure-catalog//units/rds?ref=${values.version}"
  path   = "database"
  values = {
    version        = values.version
    name           = values.name
    instance_class = try(values.instance_class, "db.t3.medium")
    vpc_path       = "../vpc"
  }
}

unit "app" {
  source = "github.com/acme/infrastructure-catalog//units/ecs-service?ref=${values.version}"
  path   = "app"
  values = {
    version       = values.version
    name          = values.name
    vpc_path      = "../vpc"
    database_path = "../database"
  }
}
```

Stacks in the catalog also use full git URLs for unit sources, for the same reason as modules -- when consumed externally, relative paths won't resolve.

### The Values Interface

The `values` mechanism is the key interface between stacks and units, replacing the older `_env/` include pattern. Stacks pass `values = { ... }` to each unit, and units read them via `values.*`:

```hcl
# In the stack file
unit "app" {
  source = "github.com/acme/catalog//units/app?ref=v1.0.0"
  path   = "app"
  values = {
    name       = "my-app"
    memory     = 512
    vpc_path   = "../vpc"
  }
}
```

```hcl
# In the unit's terragrunt.hcl
inputs = {
  name       = values.name
  memory     = try(values.memory, 256)    # try() for optional values with defaults
  vpc_id     = dependency.vpc.outputs.vpc_id
}
```

When `terragrunt stack generate` runs, it creates a `terragrunt.values.hcl` file alongside the unit's `terragrunt.hcl` containing the passed values. Units access values directly via `values.*` without any import.

### Version Pinning in Stacks

Stacks pin to specific catalog versions via git ref tags. This enables safe promotion across environments:

```hcl
locals {
  catalog_url = "github.com/acme/infrastructure-catalog"
  catalog_ref = "v1.2.0"
}

unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = {
    version = local.catalog_ref
    ...
  }
}
```

Units receive the version via `values.version` and use it to pin their module source: `source = "github.com/acme/infrastructure-catalog//modules/vpc?ref=${values.version}"`. This dual pinning ensures both the unit and its module come from the same catalog version.

To upgrade, change the `catalog_ref` in the stack file. Test in non-prod first, then promote to prod by updating the ref.

### Dependency Paths via Values

Dependency paths between units are passed through values, making units relocatable:

```hcl
# Stack defines the layout and wires dependencies via paths
unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = { name = "main" }
}

unit "roles" {
  source = "${local.catalog_url}//units/iam-role?ref=${local.catalog_ref}"
  path   = "roles/app-role"
  values = {
    name                = "app-role"
    dynamodb_table_path = "../../database"
  }
}

unit "database" {
  source = "${local.catalog_url}//units/dynamodb-table?ref=${local.catalog_ref}"
  path   = "database"
  values = { name = "app-data" }
}
```

The unit receives and uses the path:

```hcl
# units/iam-role/terragrunt.hcl
dependency "dynamodb" {
  config_path = values.dynamodb_table_path
  mock_outputs = {
    arn = "arn:aws:dynamodb:us-east-1:123456789012:table/mock"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

The `path` attribute in each unit block determines where the unit is generated within `.terragrunt-stack/`, which controls relative path resolution for dependencies.

### Reusable Stacks in the Catalog

Stacks in the catalog's `stacks/` directory can be consumed by infrastructure-live via the `stack` block, enabling stacks-of-stacks:

```hcl
# infrastructure-live/prod/us-east-1/services/terragrunt.stack.hcl

stack "web" {
  source = "github.com/acme/catalog//stacks/web-service?ref=v1.0.0"
  path   = "web"
  values = {
    name           = "web-service"
    cidr_block     = "10.0.0.0/16"
    instance_class = "db.r6g.xlarge"
    aws_region     = "us-east-1"
  }
}

stack "api" {
  source = "github.com/acme/catalog//stacks/web-service?ref=v1.0.0"
  path   = "api"
  values = {
    name           = "api-service"
    cidr_block     = "10.1.0.0/16"
    instance_class = "db.r6g.large"
    aws_region     = "us-east-1"
  }
}
```

Each nested stack generates its own `.terragrunt-stack/` directory. `run --all` operates across all nested units.

## Root Configuration (root.hcl)

The root configuration file is placed at the repository root and included by every unit via `find_in_parent_folders("root.hcl")`. It centralizes settings that apply across all environments, accounts, and regions.

**Why `root.hcl` and not `terragrunt.hcl`:** Terragrunt's `root-terragrunt-hcl` strict control enforces that the repo root file is named `root.hcl`. A file named `terragrunt.hcl` at the repo root is ambiguous -- Terragrunt cannot tell if it is a root configuration or an executable unit. Using `root.hcl` eliminates this ambiguity.

### Full example root.hcl

```hcl
# infrastructure-live/root.hcl

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}

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

Key points about `root.hcl`:

- **`path_relative_to_include()`** returns the relative path from `root.hcl` down to the including unit. For a unit at `prod/us-east-1/vpc/terragrunt.hcl`, this returns `prod/us-east-1/vpc`, which becomes the S3 state key -- guaranteeing unique state paths.
- **`remote_state` with `generate`** creates a `backend.tf` file in the working directory so OpenTofu/Terraform sees a properly configured backend.
- **`generate "provider"`** creates a `provider.tf` file, centralizing provider configuration so individual units do not repeat it.
- **`inputs`** defined here are merged with inputs from child configurations. Child inputs take precedence.

## Hierarchical Configuration

Both the stacks+catalog pattern and the traditional pattern use the same hierarchical configuration approach with `account.hcl` and `region.hcl` files.

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
# infrastructure-live/non-prod/account.hcl
locals {
  account_name = "non-prod"
  account_id   = "234567890123"
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

These files are read by both `root.hcl` (via `find_in_parent_folders`) and stack files (via `read_terragrunt_config`) to inject environment-specific values.

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

The root config uses the `iam_role` top-level attribute:

```hcl
# infrastructure-live/root.hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_name = local.account_vars.locals.account_name
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}

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
```

### Deploying a single account, region, or stack

```bash
# Plan everything in the prod account
cd prod && terragrunt run --all plan

# Plan only the us-east-1 region in prod
cd prod/us-east-1 && terragrunt run --all plan

# Plan a specific stack
cd prod/us-east-1/web-service && terragrunt stack generate && terragrunt run --all plan
```

## Traditional Two-Repo Pattern (Implicit Stacks)

For simpler projects or teams transitioning to stacks, the traditional two-repo pattern still works. This approach uses implicit stacks (directory-based) and individual `terragrunt.hcl` files per unit.

- **`infrastructure-live`** contains `terragrunt.hcl` files organized by account/region/resource.
- **`infrastructure-modules`** contains reusable `.tf` modules referenced by git URL.

### Directory Layout Conventions

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

- **Directory names mirror the infrastructure purpose**, not the module name. Use `vpc/`, `app/`, `rds/` rather than `terraform-aws-vpc/`.
- **Account directories** use the environment name: `prod/`, `staging/`, `dev/`.
- **Region directories** use the cloud provider's region identifier: `us-east-1/`, `eu-west-1/`.
- **Each leaf directory** contains exactly one `terragrunt.hcl` file representing a single deployable unit.
- **Non-unit `.hcl` files** (`account.hcl`, `region.hcl`) sit alongside the directories they apply to.

### How units reference modules

A `terragrunt.hcl` in infrastructure-live references the module via a git URL with a `ref` tag:

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

### Environment-Specific Patterns with _env Directory

When multiple environments deploy the same type of resource, the `_env/` directory pattern eliminates duplication:

```hcl
# infrastructure-live/_env/app.hcl
terraform {
  source = "git::git@github.com:acme/infrastructure-modules.git//app/ecs-service?ref=v2.1.0"
}

inputs = {
  container_port    = 8080
  health_check_path = "/health"
}
```

```hcl
# infrastructure-live/prod/us-east-1/app/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/app.hcl"
  expose = true
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
  app_name       = "prod-web-app"
  instance_count = 3
  vpc_id         = dependency.vpc.outputs.vpc_id
  subnet_ids     = dependency.vpc.outputs.private_subnets
}
```

The stacks+catalog pattern replaces this `_env/` include approach with the `values` interface, which provides a cleaner, more explicit contract between the configuration layer and the unit definitions.

### Infrastructure Modules Repo Structure

```
infrastructure-modules/
├── networking/
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
├── compute/
│   └── ecs-service/
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
└── storage/
    └── s3-bucket/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── versions.tf
```

Modules are versioned using git tags. The `//` in source URLs separates the repo URL from the subdirectory path. The `?ref=v1.2.0` pins to a specific git tag.

In the stacks+catalog pattern, these modules move into the `modules/` directory of the infrastructure-catalog repo rather than living in a separate repository.

## Working Locally with Source Overrides

During module development, override the remote git source to point at a local checkout:

### Using the --source flag

```bash
cd infrastructure-live/dev/us-east-1/vpc
terragrunt run plan --source ../../../infrastructure-modules//networking/vpc
```

### Using --source with run --all

```bash
cd infrastructure-live/dev/us-east-1
terragrunt run --all plan --source ../../../infrastructure-modules
```

### Using TERRAGRUNT_SOURCE environment variable

```bash
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
.terraform.lock.hcl

# Generated files from terragrunt generate blocks
backend.tf
provider.tf
```

The `.terragrunt-cache/` directory is where Terragrunt downloads remote modules. The `.terragrunt-stack/` directory contains generated files when using explicit stacks. Both are safe to delete at any time -- Terragrunt recreates them on the next run.
