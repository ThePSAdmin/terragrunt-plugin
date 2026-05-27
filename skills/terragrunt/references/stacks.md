# Terragrunt Stacks

Stacks are how Terragrunt orchestrates multiple infrastructure units together. Explicit stacks using `terragrunt.stack.hcl` files and an infrastructure catalog are the recommended approach for structuring Terragrunt projects. Implicit stacks (directory-based with `run --all`) remain available for simpler use cases.

## Table of Contents

- [Explicit Stacks (Recommended)](#explicit-stacks-recommended)
  - [terragrunt.stack.hcl Structure](#terragruntstack-hcl-structure)
  - [Unit Block](#unit-block)
  - [Generated Structure](#generated-structure)
  - [The Values Interface](#the-values-interface)
  - [Working with Explicit Stacks](#working-with-explicit-stacks)
  - [Stack Block (Nested Stacks)](#stack-block-nested-stacks)
- [Implicit Stacks](#implicit-stacks)
- [Choosing Between Implicit and Explicit Stacks](#choosing-between-implicit-and-explicit-stacks)
- [Stack Outputs](#stack-outputs)
- [Dependencies in Stacks](#dependencies-in-stacks)
- [Using Local State with Stacks](#using-local-state-with-stacks)
- [CAS Integration (Content-Addressable Storage)](#cas-integration-content-addressable-storage)
- [Nested Stacks](#nested-stacks)
- [Migration from Implicit to Explicit Stacks](#migration-from-implicit-to-explicit-stacks)
- [Known Limitations](#known-limitations)

## Explicit Stacks (Recommended)

Explicit stacks use `terragrunt.stack.hcl` files as blueprints that programmatically generate units. A stack file declares what units to create, where to source them from, and what values to pass to them. Running `terragrunt stack generate` materializes the units into a `.terragrunt-stack/` directory.

This is the recommended approach because it provides reusability, consistency across environments, version pinning, and a clean interface between configuration and implementation via the `values` mechanism.

### terragrunt.stack.hcl Structure

```hcl
locals {
  catalog_url = "github.com/acme/infrastructure-catalog"
  catalog_ref = "v1.0.0"
  name        = "my-service"
}

unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = {
    version    = local.catalog_ref
    name       = "${local.name}-vpc"
    cidr_block = "10.0.0.0/16"
  }
}

unit "database" {
  source = "${local.catalog_url}//units/dynamodb-table?ref=${local.catalog_ref}"
  path   = "database"
  values = {
    version  = local.catalog_ref
    name     = "${local.name}-data"
    hash_key = "id"
  }
}

unit "app" {
  source = "${local.catalog_url}//units/lambda-service?ref=${local.catalog_ref}"
  path   = "service"
  values = {
    version             = local.catalog_ref
    name                = local.name
    runtime             = "nodejs22.x"
    role_path           = "../roles/app-role"
    dynamodb_table_path = "../database"
  }
}

unit "role" {
  source = "${local.catalog_url}//units/lambda-iam-role?ref=${local.catalog_ref}"
  path   = "roles/app-role"
  values = {
    version             = local.catalog_ref
    name                = "${local.name}-role"
    dynamodb_table_path = "../../database"
  }
}
```

Each `unit` block specifies a source (a git URL or local path containing a `terragrunt.hcl`), a path where it will be placed, and optional values to pass into the unit.

### Unit Block

The `unit` block has these attributes:

- **`source`** (required): Where to fetch the unit configuration from. Accepts GitHub shorthand (`github.com/org/repo//path?ref=tag`), full git URLs (`git::git@github.com:org/repo.git//path?ref=tag`), or local paths (`../../units/vpc`).
- **`path`** (required): Where to place the generated unit within the `.terragrunt-stack/` directory. This determines relative dependency paths and the `path_relative_to_include()` value used for state keys.
- **`values`** (optional): A map of key-value pairs passed to the unit. Available in the unit's config via `values.*`.
- **`update_source_with_cas`**: Boolean. When `true`, enables CAS (Content-Addressable Storage) integration for the source.
- **`mutable`**: Boolean. When `true`, makes cached content editable after generation.

### Generated Structure

After running `terragrunt stack generate`:

```
.terragrunt-stack/
├── vpc/
│   ├── terragrunt.hcl
│   └── terragrunt.values.hcl
├── database/
│   ├── terragrunt.hcl
│   └── terragrunt.values.hcl
├── service/
│   ├── terragrunt.hcl
│   └── terragrunt.values.hcl
└── roles/
    └── app-role/
        ├── terragrunt.hcl
        └── terragrunt.values.hcl
```

The `terragrunt.values.hcl` file contains the values passed from the stack definition, accessible via `values.*` in the unit's `terragrunt.hcl`.

### The Values Interface

The `values` mechanism is the key interface between stacks and units. It replaces the older `_env/` include pattern with an explicit, typed contract:

**Stack side** -- defines what configuration to pass:
```hcl
unit "app" {
  source = "${local.catalog_url}//units/ecs-service?ref=v1.0.0"
  path   = "app"
  values = {
    version       = "v1.0.0"
    name          = "web-app"
    memory        = 512
    cpu           = 256
    vpc_path      = "../vpc"
    database_path = "../database"
  }
}
```

**Unit side** -- consumes configuration via `values.*`:
```hcl
# units/ecs-service/terragrunt.hcl

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

dependency "database" {
  config_path = values.database_path
  mock_outputs = {
    endpoint = "mock-endpoint:5432"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

terraform {
  source = "github.com/acme/infrastructure-catalog//modules/compute/ecs-service?ref=${values.version}"
}

inputs = {
  name          = values.name
  memory        = try(values.memory, 256)
  cpu           = try(values.cpu, 256)
  vpc_id        = dependency.vpc.outputs.vpc_id
  subnet_ids    = dependency.vpc.outputs.private_subnets
  db_endpoint   = dependency.database.outputs.endpoint
}
```

Use `try(values.foo, default)` for optional values with defaults.

### Working with Explicit Stacks

```bash
# Generate units from terragrunt.stack.hcl
terragrunt stack generate

# Plan all generated units
terragrunt run --all plan

# Apply all generated units
terragrunt run --all apply

# Or use the stack subcommand directly
terragrunt stack run -- apply

# Get stack outputs
terragrunt stack output

# Get JSON outputs
terragrunt stack output --format json

# Clean generated directories
terragrunt stack clean
```

The `stack generate` command fetches sources and creates the `.terragrunt-stack/` directory. After generation, `run --all` or `stack run` operates on all generated units. The `stack clean` command removes the `.terragrunt-stack/` directory entirely.

### Stack Block (Nested Stacks)

The `stack` block has the same arguments as the `unit` block but generates another `terragrunt.stack.hcl` instead of a unit. This enables hierarchical composition:

```hcl
stack "non_prod" {
  source = "github.com/acme/catalog//stacks/web-service?ref=v1.0.0"
  path   = "non-prod"
  values = {
    version        = "v1.0.0"
    name           = "web-service-dev"
    cidr_block     = "10.0.0.0/16"
    instance_class = "db.t3.medium"
  }
}

stack "prod" {
  source = "github.com/acme/catalog//stacks/web-service?ref=v1.0.0"
  path   = "prod"
  values = {
    version        = "v1.0.0"
    name           = "web-service"
    cidr_block     = "10.1.0.0/16"
    instance_class = "db.r6g.xlarge"
  }
}
```

The referenced stack (`stacks/web-service/terragrunt.stack.hcl`) uses the passed values:

```hcl
unit "vpc" {
  source = "github.com/acme/catalog//units/vpc?ref=${values.version}"
  path   = "vpc"
  values = {
    version    = values.version
    name       = values.name
    cidr_block = values.cidr_block
  }
}

unit "database" {
  source = "github.com/acme/catalog//units/rds?ref=${values.version}"
  path   = "database"
  values = {
    version        = values.version
    name           = values.name
    instance_class = values.instance_class
    vpc_path       = "../vpc"
  }
}
```

## Implicit Stacks

Implicit stacks are the traditional approach -- organizing units in a directory tree and using `run --all` to operate on them. A stack is simply a directory containing multiple units (directories with `terragrunt.hcl` files). No special file declares the stack; the directory structure itself defines it.

```
prod/
├── vpc/terragrunt.hcl
├── database/terragrunt.hcl
├── app/terragrunt.hcl
└── frontend/terragrunt.hcl
```

```bash
cd prod
terragrunt run --all plan
terragrunt run --all apply
```

Dependencies between units are defined via `dependency` and `dependencies` blocks in each unit's `terragrunt.hcl`.

Implicit stacks are simpler and familiar but lead to repetition across environments and don't support reusable patterns or the `values` interface. Consider migrating to explicit stacks when you have multiple environments sharing the same infrastructure patterns.

## Choosing Between Implicit and Explicit Stacks

| Criteria | Implicit | Explicit |
|----------|----------|----------|
| Multiple similar environments | Repetitive | Best choice -- define once, reuse |
| Infrastructure catalogs/templates | Manual | Native support |
| Reusable infrastructure patterns | Not supported | Best choice |
| Version pinning across environments | Manual (per-unit ref tags) | Centralized (single catalog ref) |
| Getting started / simple projects | Good choice | Add when repetition grows |
| Maximum transparency | Best choice | Generated code in .terragrunt-stack/ |
| Small number of unique units | Simpler | More structure than needed |

**Recommendation**: Use explicit stacks with an infrastructure catalog for any project with multiple environments or teams sharing infrastructure patterns. The stacks+catalog approach scales better and keeps environments consistent. For single-environment projects or quick prototypes, implicit stacks are fine to start with.

## Stack Outputs

`terragrunt stack output` aggregates outputs from all units in the stack:

```bash
$ terragrunt stack output
backend_app = {
  domain = "backend-app.example.com"
}
mysql = {
  endpoint = "terraform-2025.us-east-1.rds.amazonaws.com"
}
vpc = {
  vpc_id = "vpc-1234567890"
}
```

Use `--format json` for programmatic access:

```bash
$ terragrunt stack output --format json
{
  "backend_app": {
    "domain": "backend-app.example.com"
  },
  "mysql": {
    "endpoint": "terraform-2025.us-east-1.rds.amazonaws.com"
  },
  "vpc": {
    "vpc_id": "vpc-1234567890"
  }
}
```

## Dependencies in Stacks

In the stacks+catalog pattern, dependency paths are passed through `values` rather than hardcoded in units. This makes units relocatable across different stack layouts.

**Stack defines layout and wires dependencies:**
```hcl
unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=${local.catalog_ref}"
  path   = "vpc"
  values = { name = "main" }
}

unit "app" {
  source = "${local.catalog_url}//units/ecs-service?ref=${local.catalog_ref}"
  path   = "app"
  values = {
    name     = "web"
    vpc_path = "../vpc"
  }
}
```

**Unit receives dependency paths from values:**
```hcl
# units/ecs-service/terragrunt.hcl
dependency "vpc" {
  config_path = values.vpc_path
  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

The `path` attribute in each unit block determines where within `.terragrunt-stack/` the unit is generated, which controls how relative dependency paths resolve. Plan your `path` values and dependency paths together.

For ordering-only dependencies (no output access), units can also use the `dependencies` block:
```hcl
dependencies {
  paths = [values.sg_rule_path]
}
```

## Using Local State with Stacks

The `.terragrunt-stack/` directory can be deleted and regenerated at any time. State files should be stored outside it so they persist across regeneration.

Configure a `root.hcl` that places state outside the generated directory:

```hcl
remote_state {
  backend = "local"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    path = "${get_parent_terragrunt_dir()}/.terragrunt-local-state/${path_relative_to_include()}/tofu.tfstate"
  }
}
```

This stores state files in `.terragrunt-local-state/`, which persists across `stack clean` and `stack generate` cycles. Add `.terragrunt-local-state/` to `.gitignore`.

## CAS Integration (Content-Addressable Storage)

CAS deduplicates repository clones when working with stack catalogs. Enable it with `--experiment cas`.

**Without CAS** (requires network for each generation, error-prone):

```hcl
unit "service" {
  source = "github.com/acme/catalog//units/service?ref=${values.version}"
  path   = "service"
}
```

**With CAS** (recommended for catalogs):

```hcl
unit "service" {
  source = "../..//units/service"
  update_source_with_cas = true
  path = "service"
}
```

How CAS works:

1. Terragrunt clones the catalog repository once via CAS.
2. It follows relative paths within the clone to find the requested unit.
3. It rewrites the source to a `cas::sha1:abc123...` reference.
4. Subsequent generations resolve from the CAS store without network access.

CAS also works in `terraform` blocks within unit configurations:

```hcl
terraform {
  source = "../..//modules/service"
  update_source_with_cas = true
}
```

Requirements for CAS:

- The `cas` experiment must be enabled (`--experiment cas`).
- The `--no-cas` flag must NOT be set.
- Sources must be relative paths within the same repository.

## Nested Stacks

Stacks can contain other stacks for hierarchical infrastructure organization. Each nested stack generates its own `.terragrunt-stack/` directory:

```
.terragrunt-stack/
├── non-prod/
│   ├── terragrunt.stack.hcl
│   └── .terragrunt-stack/
│       ├── vpc/
│       │   ├── terragrunt.hcl
│       │   └── terragrunt.values.hcl
│       └── database/
│           ├── terragrunt.hcl
│           └── terragrunt.values.hcl
└── prod/
    ├── terragrunt.stack.hcl
    └── .terragrunt-stack/
        ├── vpc/
        │   ├── terragrunt.hcl
        │   └── terragrunt.values.hcl
        └── database/
            ├── terragrunt.hcl
            └── terragrunt.values.hcl
```

`run --all` works at any depth within nested stacks. You can run from the top-level to operate on everything, or descend into a specific environment to operate on just that subset.

## Migration from Implicit to Explicit Stacks

When migrating existing infrastructure from implicit to explicit stacks, use `no_dot_terragrunt_stack = true` on units that need to preserve existing state file paths:

```hcl
unit "vpc" {
  source = "${local.catalog_url}//units/vpc?ref=v1.0.0"
  path   = "vpc"
  no_dot_terragrunt_stack = true
  values = { ... }
}
```

This causes the unit to generate directly into the current directory rather than `.terragrunt-stack/`, preserving the `path_relative_to_include()` value and therefore the state key.

## Known Limitations

1. **Dependencies cannot be set on stacks** -- the `dependency` block cannot target a stack's `config_path`. Workaround: set `config_path` to a specific unit within the stack rather than the stack itself.

2. **Deeply nested generation can be slow** -- every generation step may involve network traffic to fetch sources. Mitigation: enable CAS with `update_source_with_cas = true` to cache and deduplicate fetches.

3. **Includes not supported in terragrunt.stack.hcl** -- `include` blocks cannot be used in stack files. Nested stacks provide organizational structure without relying on includes. This may change based on community feedback.

4. **Invalid to have both terragrunt.hcl and terragrunt.stack.hcl** -- a directory is either a unit or a stack, never both. Terragrunt will error if both files exist in the same directory.
