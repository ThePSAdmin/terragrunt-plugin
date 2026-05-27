---
name: terragrunt
description: >-
  Comprehensive Terragrunt guide for infrastructure-as-code with OpenTofu/Terraform.
  Use this skill whenever the user mentions Terragrunt, works with terragrunt.hcl files,
  asks about DRY Terraform/OpenTofu patterns, multi-environment infrastructure deployment,
  infrastructure-live or infrastructure-modules or infrastructure-catalog repositories,
  terragrunt run/plan/apply, remote state management with Terragrunt, cross-unit
  dependencies, dependency blocks, include blocks, stacks (implicit or explicit),
  terragrunt.stack.hcl files, mock_outputs, the values interface, catalog units,
  multi-account AWS/GCP/Azure deployment with Terragrunt, Terragrunt GitHub Actions,
  Terragrunt CLI commands, or any Terragrunt-specific HCL configuration. Also trigger
  when the user references DRY infrastructure patterns, wants to orchestrate multiple
  Terraform/OpenTofu modules together, or asks about Terragrunt filtering, caching,
  hooks, scaffolding, or catalog features.
---

# Terragrunt

Terragrunt is a flexible orchestration tool for OpenTofu/Terraform that keeps configurations DRY, manages remote state, and orchestrates multi-module deployments. It wraps OpenTofu/Terraform with a thin layer defined in `terragrunt.hcl` files called **units**, which can be composed into **stacks** for coordinated deployment across environments and accounts.

## Critical Rules

These are non-negotiable — getting any of these wrong will produce broken configurations:

1. **`terragrunt run` is the primary command.** Use `terragrunt run --all plan`, not `terragrunt plan` or `terragrunt run-all plan`. Direct forwarding (e.g. `terragrunt plan`) still works as a shortcut but `run` is canonical.

2. **`dependency` vs `dependencies` are different blocks.** `dependency` (singular) gives access to outputs via `dependency.<name>.outputs.<key>`. `dependencies` (plural) only controls execution order — no output access.

3. **`include` uses `path`, `terraform` uses `source`.** Never confuse them. `include` pulls in parent Terragrunt config. `terraform.source` points to OpenTofu/Terraform modules.

4. **`find_in_parent_folders()` searches UP the directory tree** from the current `terragrunt.hcl` file. It returns an absolute path to the first match.

5. **`inputs` map directly to Terraform variables** via `TF_VAR_*` environment variables. The keys in `inputs` must match variable names in your `.tf` files.

6. **Root config file is `root.hcl`** (not `terragrunt.hcl` at the repo root). The `root-terragrunt-hcl` strict control enforces this.

7. **`path_relative_to_include()` is evaluated relative to the including child**, not the included parent. It returns the path from the parent config down to the child.

8. **Mock outputs are required for `run --all plan` on fresh infrastructure.** Without them, dependency output resolution fails because no state exists yet.

9. **Never set `TF_PLUGIN_CACHE_DIR` with `run --all`.** Use Terragrunt's built-in Provider Cache Server instead to avoid concurrent access issues.

10. **`run --all apply` auto-adds `-auto-approve`** because stdin is shared across units. Use `--no-auto-approve` to override, but you'll need an alternative approval workflow.

## Decision Tree

Use this to find the right reference file for the task at hand:

**Setting up a new Terragrunt project or restructuring directories?**
Read `references/repository-structure.md`

**Writing or editing a `terragrunt.hcl` file (blocks, attributes, parsing)?**
Read `references/hcl-configuration.md`

**Need a built-in function (`find_in_parent_folders`, `get_env`, `run_cmd`, etc.)?**
Read `references/functions.md`

**Working with stacks (multi-unit orchestration, `terragrunt.stack.hcl`)?**
Read `references/stacks.md`

**Connecting units together (dependencies, includes, passing outputs)?**
Read `references/dependencies-and-includes.md`

**Configuring remote state (S3, GCS, Azure backends)?**
Read `references/remote-state.md`

**Looking up CLI commands, flags, or environment variables?**
Read `references/cli-reference.md`

**Setting up GitHub Actions CI/CD for Terragrunt?**
Read `references/github-actions.md`

**Optimizing performance, filtering, hooks, or caching?**
Read `references/execution-and-performance.md`

**Debugging errors, logging, telemetry, or error handling config?**
Read `references/debugging-and-observability.md`

## Quick-Start Patterns

### Stacks-Based Project (Recommended)

The preferred way to structure Terragrunt is with explicit stacks and a catalog. The infrastructure-live repo contains `terragrunt.stack.hcl` files that source units from an infrastructure-catalog repo:

```hcl
# infrastructure-live/prod/us-east-1/my-service/terragrunt.stack.hcl

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  name         = "my-service"
}

unit "vpc" {
  source = "github.com/acme/infrastructure-catalog//units/vpc?ref=v1.0.0"
  path   = "vpc"
  values = {
    version    = "v1.0.0"
    name       = "${local.name}-vpc"
    cidr_block = "10.0.0.0/16"
    aws_region = local.region_vars.locals.aws_region
  }
}

unit "database" {
  source = "github.com/acme/infrastructure-catalog//units/rds?ref=v1.0.0"
  path   = "database"
  values = {
    version  = "v1.0.0"
    name     = local.name
    engine   = "postgres"
    vpc_path = "../vpc"
  }
}

unit "app" {
  source = "github.com/acme/infrastructure-catalog//units/ecs-service?ref=v1.0.0"
  path   = "app"
  values = {
    version       = "v1.0.0"
    name          = local.name
    vpc_path      = "../vpc"
    database_path = "../database"
  }
}
```

A catalog unit uses `values.*` to receive configuration and `dependency` blocks with paths from values:

```hcl
# infrastructure-catalog/units/rds/terragrunt.hcl

include "root" {
  path = find_in_parent_folders("root.hcl")
}

dependency "vpc" {
  config_path = values.vpc_path
  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

terraform {
  source = "github.com/acme/infrastructure-catalog//modules/database/rds?ref=${values.version}"
}

inputs = {
  name      = values.name
  engine    = try(values.engine, "postgres")
  vpc_id    = dependency.vpc.outputs.vpc_id
  subnet_id = dependency.vpc.outputs.private_subnets[0]
}
```

### Stack Operations

```bash
# Generate units from stack definition
terragrunt stack generate

# Plan all generated units
terragrunt run --all plan

# Apply all units respecting dependency order
terragrunt run --all apply

# Destroy in reverse dependency order
terragrunt run --all destroy

# Run only units matching a filter
terragrunt run --all --filter './prod/**' -- plan

# Get aggregated stack outputs
terragrunt stack output
```

### Basic Unit (Without Stacks)

A minimal standalone `terragrunt.hcl` for simpler setups:

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

terraform {
  source = "git::git@github.com:acme/modules.git//networking/vpc?ref=v1.0.0"
}

inputs = {
  vpc_name = "main"
  cidr     = "10.0.0.0/16"
}
```

### Root Configuration with Remote State

A `root.hcl` at the repository root that all units include:

```hcl
remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket         = "my-tofu-state"
    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "my-lock-table"
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "us-east-1"
}
EOF
}
```

## Common Mistakes

1. **Using `terragrunt.hcl` as root config name** — Use `root.hcl` instead. A file named `terragrunt.hcl` at the repo root is ambiguous (is it a unit or root config?).

2. **Forgetting `mock_outputs` on dependencies** — `run --all plan` fails on fresh infra without mocks because dependency state doesn't exist yet.

3. **Circular dependencies** — Terragrunt's DAG does not allow cycles. If A depends on B and B depends on A, restructure by extracting a shared module or using `dependencies` (ordering-only) instead.

4. **Setting `TF_PLUGIN_CACHE_DIR` with `run --all`** — Causes concurrent access corruption. Use Terragrunt's Provider Cache Server instead.

5. **Using `path_relative_to_include()` outside an included context** — This function only works when called from a config that is included by a child. In a standalone config, it returns `.`.

6. **Forgetting `generate` in `remote_state`** — Without the `generate` block, Terragrunt manages state config via CLI args rather than generating a `backend.tf` file, which can cause issues with some backends.

7. **Not adding `.terragrunt-cache` and `.terragrunt-stack` to `.gitignore`** — These directories contain downloaded modules and generated stack files and should never be committed.

## Recommended Repo Layout (Stacks + Catalog)

```
infrastructure-live/                  # Stacks that define deployed infrastructure
├── root.hcl                          # Shared root config (remote state, provider)
├── non-prod/
│   ├── account.hcl                   # Account name, account ID
│   └── us-east-1/
│       ├── region.hcl               # AWS region
│       └── my-service/
│           └── terragrunt.stack.hcl  # Stack sourcing units from catalog
├── prod/
│   ├── account.hcl
│   └── us-east-1/
│       ├── region.hcl
│       └── my-service/
│           └── terragrunt.stack.hcl  # Same structure, different values
└── .gitignore                        # Must include .terragrunt-stack/

infrastructure-catalog/               # Reusable modules, units, and stacks
├── modules/                          # Pure OpenTofu/Terraform modules
│   ├── networking/vpc/
│   ├── database/rds/
│   └── app/ecs-service/
├── units/                            # Terragrunt units wrapping modules
│   ├── vpc/terragrunt.hcl           # Uses values.* interface
│   ├── rds/terragrunt.hcl
│   └── ecs-service/terragrunt.hcl
├── stacks/                           # Reusable stack compositions (optional)
│   └── web-service/
│       └── terragrunt.stack.hcl
├── root.hcl                          # Root config for catalog units
└── examples/                         # Usage examples and tests
```

For simpler setups without a catalog, the traditional two-repo implicit stacks pattern still works. See `references/repository-structure.md` for both approaches.

## References

- **`references/repository-structure.md`** — Stacks + catalog pattern (recommended), traditional two-repo pattern, directory layout, multi-account and multi-environment patterns, DRY include chains. Read when setting up or restructuring a Terragrunt project.

- **`references/hcl-configuration.md`** — Complete reference for all HCL blocks (`terraform`, `remote_state`, `include`, `locals`, `generate`, `feature`, `exclude`, `errors`) and top-level attributes (`inputs`, `prevent_destroy`, `iam_role`, etc.). Read when writing or debugging `terragrunt.hcl` files.

- **`references/functions.md`** — All 25+ built-in functions with signatures, parameters, and examples. Read when you need `find_in_parent_folders`, `get_env`, `run_cmd`, `read_terragrunt_config`, `sops_decrypt_file`, `deep_merge`, AWS identity functions, or any other Terragrunt function.

- **`references/stacks.md`** — Explicit stacks with catalogs (recommended), implicit stacks (directory-based), the values interface, stack outputs, dependencies in stacks, CAS integration, nested stacks, migration guide, limitations. Read when orchestrating multiple units as a group.

- **`references/dependencies-and-includes.md`** — `dependency` and `dependencies` blocks, mock outputs, passing outputs between units, multiple includes, exposed includes, merge strategies. Read when connecting units together or setting up DRY include hierarchies.

- **`references/remote-state.md`** — S3, GCS, and Azure backend configuration, automatic resource creation, encryption, S3 native locking, DRY remote state patterns. Read when configuring state storage.

- **`references/cli-reference.md`** — All commands (`run`, `exec`, `stack`, `backend`, `catalog`, `scaffold`, `find`, `list`, `dag`, `hcl fmt/validate`, `info`, `render`), all flags, and all `TG_*` environment variables. Read when looking up CLI usage.

- **`references/github-actions.md`** — Terragrunt Scale, OIDC authentication, GitOps workflows, IAM policies, bootstrap stack, `terragrunt-action`. Read when setting up CI/CD.

- **`references/execution-and-performance.md`** — Run queue, DAG, parallelism, filtering expressions, provider caching, hooks, TFLint integration, `extra_arguments`, auto-init. Read when optimizing runs or configuring hooks.

- **`references/debugging-and-observability.md`** — Log levels, formatting, strict controls, OpenTelemetry, error handling (`retry`/`ignore`), feature flags, run summary/report. Read when debugging or setting up observability.
