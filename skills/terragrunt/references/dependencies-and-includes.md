# Dependencies and Includes

How Terragrunt units declare dependencies on each other and inherit shared configuration through includes.

## Table of Contents

- [dependency Block](#dependency-block)
  - [Attributes](#attributes)
  - [Mock Outputs](#mock-outputs)
  - [Merge Strategy](#merge-strategy)
  - [Multiple Dependencies](#multiple-dependencies)
  - [Dependencies in Explicit Stacks](#dependencies-in-explicit-stacks)
- [dependencies Block](#dependencies-block)
  - [dependency vs dependencies](#dependency-vs-dependencies)
- [Passing Outputs Between Units](#passing-outputs-between-units)
  - [Basic Pattern](#basic-pattern)
  - [Multi-Tier Dependency Chain](#multi-tier-dependency-chain)
- [Visualizing the DAG](#visualizing-the-dag)
- [include Block](#include-block)
  - [Basic Include](#basic-include)
  - [Multiple Includes](#multiple-includes)
  - [Exposed Includes](#exposed-includes)
  - [Merge Strategies](#merge-strategies)
  - [Include Nesting](#include-nesting)
  - [What Gets Merged](#what-gets-merged)
- [read_terragrunt_config for Cross-File Data](#read_terragrunt_config-for-cross-file-data)
- [CI/CD Considerations for Includes](#cicd-considerations-for-includes)
- [Common Patterns](#common-patterns)
  - [Root + Account + Region Include Chain](#root--account--region-include-chain)
  - [Shared Environment Config Pattern](#shared-environment-config-pattern)

## dependency Block

The `dependency` block creates a DAG edge and provides access to another unit's outputs.

```hcl
dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
}
```

### Attributes

- `config_path` (required): relative or absolute path to the dependency's terragrunt.hcl
- `mock_outputs` (optional): map of mock output values for when dependency hasn't been applied yet
- `mock_outputs_allowed_terraform_commands` (optional): list of commands that may use mocks (e.g. `["validate", "plan"]`)
- `mock_outputs_merge_strategy_with_state` (optional): `"shallow"` — merge mocked outputs with real state outputs (mocks fill gaps)
- `skip_outputs` (optional): boolean — skip fetching outputs entirely (just establishes ordering)
- `enabled` (optional): boolean — conditionally enable/disable the dependency

### Mock Outputs

Mock outputs are required for `run --all plan` on fresh infrastructure (no state exists yet):

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1", "mock-subnet-2"]
    public_subnets  = ["mock-pub-1"]
  }

  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

Without `mock_outputs_allowed_terraform_commands`, mocks are used for ALL commands. With it, only listed commands use mocks — apply/destroy will fail if real outputs aren't available.

### Merge Strategy

```hcl
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id = "mock-vpc-id"
    extra_field = "only-in-mock"
  }
  mock_outputs_merge_strategy_with_state = "shallow"
}
```

With `"shallow"`: if real state has `vpc_id`, use that, but `extra_field` comes from mocks. Useful for gradual rollout of new outputs.

### Multiple Dependencies

```hcl
dependency "vpc" {
  config_path = "../vpc"
}

dependency "rds" {
  config_path = "../rds"
}

dependency "redis" {
  config_path = "../redis"
}

inputs = {
  vpc_id         = dependency.vpc.outputs.vpc_id
  db_endpoint    = dependency.rds.outputs.endpoint
  redis_endpoint = dependency.redis.outputs.endpoint
}
```

### Dependencies in Explicit Stacks

Pass dependency paths via values:

terragrunt.stack.hcl:
```hcl
unit "vpc" {
  source = "../../units/vpc"
  path   = "vpc"
}

unit "app" {
  source = "../../units/app"
  path   = "app"
  values = {
    vpc_path = "../vpc"
  }
}
```

units/app/terragrunt.hcl:
```hcl
dependency "vpc" {
  config_path = values.vpc_path
}
```

## dependencies Block

The `dependencies` block (plural) establishes ordering WITHOUT output access:

```hcl
dependencies {
  paths = ["../vpc", "../mysql", "../redis"]
}
```

Use when you need units to run in order but don't need their outputs. Multiple `dependencies` blocks are merged (concatenated).

### dependency vs dependencies

| Feature | `dependency` (singular) | `dependencies` (plural) |
|---------|------------------------|------------------------|
| Output access | Yes (`dependency.<name>.outputs.*`) | No |
| Mock outputs | Yes | N/A |
| DAG ordering | Yes | Yes |
| Multiple blocks | Yes (each with unique label) | Yes (merged) |
| Use case | Need outputs from another unit | Just need ordering |

## Passing Outputs Between Units

### Basic Pattern

VPC unit outputs:
```hcl
# vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}
output "private_subnets" {
  value = aws_subnet.private[*].id
}
```

App unit consumes:
```hcl
# app/terragrunt.hcl
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = {
    vpc_id = "mock-vpc-id"
    private_subnets = ["mock-subnet"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vpc_id    = dependency.vpc.outputs.vpc_id
  subnet_id = dependency.vpc.outputs.private_subnets[0]
}
```

### Multi-Tier Dependency Chain

```
vpc → security-groups → ecs-cluster → ecs-service
```

security-groups/terragrunt.hcl:
```hcl
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = { vpc_id = "mock" }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
}
```

ecs-cluster/terragrunt.hcl:
```hcl
dependency "vpc" {
  config_path = "../vpc"
  mock_outputs = { private_subnets = ["mock"] }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
dependency "sg" {
  config_path = "../security-groups"
  mock_outputs = { cluster_sg_id = "mock" }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
inputs = {
  subnet_ids       = dependency.vpc.outputs.private_subnets
  security_group_id = dependency.sg.outputs.cluster_sg_id
}
```

## Visualizing the DAG

```bash
# DOT format (pipe to graphviz)
terragrunt dag graph | dot -Tsvg > graph.svg

# Equivalent using list command
terragrunt list --format=dot --dependencies --external | dot -Tsvg > graph.svg

# List in plan order
terragrunt list --as plan -l

# List in destroy order
terragrunt list --as destroy -l
```

## include Block

Include blocks merge parent Terragrunt configurations into the current unit.

### Basic Include

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}
```

### Multiple Includes

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path = "${get_terragrunt_dir()}/../../_env/app.hcl"
}

inputs = {
  env = "qa"
}
```

### Exposed Includes

Set `expose = true` to access the parent's data:

```hcl
include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/app.hcl"
  expose = true
}

terraform {
  source = "${include.env.locals.source_base_url}?ref=v0.2.0"
}
```

### Merge Strategies

- `shallow` (default): top-level keys from child override parent; nested objects not merged
- `deep`: recursively merge nested maps/objects; child values override at leaf level
- `no_merge`: don't merge at all — just load the data (use with `expose = true`)

```hcl
include "root" {
  path           = find_in_parent_folders("root.hcl")
  merge_strategy = "deep"
}
```

Deep merge limitations: `remote_state` and `generate` blocks don't support deep merge.

### Include Nesting

Only single level of nesting supported. If root.hcl includes another file, that file cannot itself use includes.

### What Gets Merged

- `terraform` block: merged based on strategy
- `inputs`: merged (child overrides parent keys)
- `generate`: replaced (not merged)
- `remote_state`: replaced (not merged)
- `locals`: NOT merged (by design — each config has its own locals)
- `dependency`/`dependencies`: merged (concatenated)

## read_terragrunt_config for Cross-File Data

Load additional context dynamically without using include:

```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  
  account_id   = local.account_vars.locals.account_id
  aws_region   = local.region_vars.locals.aws_region
}
```

Key difference from `include`: `read_terragrunt_config` loads data into locals but does NOT merge blocks. `include` merges the parent's blocks into the child.

## CI/CD Considerations for Includes

Changes to included files affect all units that include them. Use `--queue-include-units-reading` to auto-detect:

```bash
# Include all units that read _env/app.hcl in the run
terragrunt run --all plan --queue-include-units-reading _env/app.hcl
```

Progressive rollout pattern:
```bash
terragrunt run --all plan --queue-include-units-reading _env/app.hcl --working-dir qa
terragrunt run --all apply --queue-include-units-reading _env/app.hcl --working-dir qa

terragrunt run --all plan --queue-include-units-reading _env/app.hcl --working-dir staging
terragrunt run --all apply --queue-include-units-reading _env/app.hcl --working-dir staging

terragrunt run --all plan --queue-include-units-reading _env/app.hcl --working-dir prod
terragrunt run --all apply --queue-include-units-reading _env/app.hcl --working-dir prod
```

## Common Patterns

### Root + Account + Region Include Chain

```hcl
# prod/us-east-1/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
}

terraform {
  source = "git::git@github.com:acme/modules.git//networking/vpc?ref=v1.0.0"
}

inputs = {
  account_id = local.account_vars.locals.account_id
  aws_region = local.region_vars.locals.aws_region
  vpc_name   = "prod-vpc"
  cidr       = "10.0.0.0/16"
}
```

### Shared Environment Config Pattern

_env/app.hcl:
```hcl
locals {
  source_base_url = "git::git@github.com:acme/modules.git//app"
}

terraform {
  source = "${local.source_base_url}?ref=v1.0.0"
}

inputs = {
  instance_type = "t4g.small"
}
```

dev/us-east-1/app/terragrunt.hcl:
```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path   = "${get_terragrunt_dir()}/../../_env/app.hcl"
  expose = true
  merge_strategy = "deep"
}

inputs = {
  env           = "dev"
  instance_type = "t4g.micro"  # Override the default
}
```
