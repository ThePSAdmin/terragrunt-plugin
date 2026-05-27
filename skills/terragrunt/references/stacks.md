# Terragrunt Stacks

Stacks are how Terragrunt orchestrates multiple infrastructure units together. There are two approaches: implicit stacks (directory-based) and explicit stacks (blueprint-based with `terragrunt.stack.hcl`).

## Table of Contents

- [Implicit Stacks](#implicit-stacks)
  - [Advantages](#advantages)
  - [Limitations](#limitations)
- [Explicit Stacks](#explicit-stacks)
  - [terragrunt.stack.hcl Structure](#terragruntstack-hcl-structure)
  - [Generated Structure](#generated-structure)
  - [Unit Block](#unit-block)
  - [Stack Block (Nested Stacks)](#stack-block-nested-stacks)
  - [Working with Explicit Stacks](#working-with-explicit-stacks)
  - [Advantages](#advantages-1)
  - [Limitations](#limitations-1)
- [Choosing Between Implicit and Explicit Stacks](#choosing-between-implicit-and-explicit-stacks)
- [Stack Outputs](#stack-outputs)
- [Using Local State with Stacks](#using-local-state-with-stacks)
- [CAS Integration (Content-Addressable Storage)](#cas-integration-content-addressable-storage)
- [Nested Stacks](#nested-stacks)
- [Known Limitations](#known-limitations)

## Implicit Stacks

Implicit stacks are the traditional approach -- organizing units in a directory tree and using `run --all` to operate on them. A stack is simply a directory containing multiple units (directories with `terragrunt.hcl` files). No special file declares the stack; the directory structure itself defines it.

Example directory structure:

```
prod/
├── vpc/terragrunt.hcl
├── database/terragrunt.hcl
├── app/terragrunt.hcl
└── frontend/terragrunt.hcl
```

Run commands across all units from any parent directory:

```bash
# Plan all units under prod/
cd prod
terragrunt run --all plan

# Apply all units under prod/
terragrunt run --all apply

# Destroy all units under prod/ (reverse dependency order)
terragrunt run --all destroy
```

Run commands on the DAG (directed acyclic graph) of the current unit and its dependencies:

```bash
# From within a specific unit directory
cd prod/app
terragrunt run --graph plan
terragrunt run --graph apply
```

Dependencies between units are defined via `dependency` and `dependencies` blocks in each unit's `terragrunt.hcl`:

```hcl
# prod/app/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"
}

dependency "database" {
  config_path = "../database"
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  db_host    = dependency.database.outputs.endpoint
}
```

```hcl
# prod/database/terragrunt.hcl

dependency "vpc" {
  config_path = "../vpc"
}

inputs = {
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

When `run --all` executes, Terragrunt builds the dependency graph and processes units in the correct order: `vpc` first, then `database`, then `app` and `frontend` in parallel (if no dependency between them).

### Advantages

- **Simple**: just organize units in directories -- no special files or abstractions needed.
- **Familiar**: follows standard IaC repository patterns that teams already know.
- **Flexible**: add or remove units by creating or deleting directories.
- **Version control friendly**: each unit has its own file history.
- **Backwards compatible**: this pattern has been used in Terragrunt for 8+ years.

### Limitations

- **Manual management**: each unit must be created and configured individually.
- **No reusability**: patterns cannot be easily shared across environments.
- **Repetitive**: similar configurations are duplicated or referenced through complex includes.

## Explicit Stacks

Explicit stacks use `terragrunt.stack.hcl` files as blueprints that programmatically generate units. A stack file defines what units to create, where to source them from, and what values to pass to them. Running `terragrunt stack generate` materializes the units into a `.terragrunt-stack/` directory.

### terragrunt.stack.hcl Structure

```hcl
unit "vpc" {
  source = "git::git@github.com:acme/infrastructure-catalog.git//units/vpc?ref=v0.0.1"
  path   = "vpc"
  values = {
    vpc_name = "main"
    cidr     = "10.0.0.0/16"
  }
}

unit "database" {
  source = "git::git@github.com:acme/infrastructure-catalog.git//units/database?ref=v0.0.1"
  path   = "database"
  values = {
    engine   = "postgres"
    version  = "13"
    vpc_path = "../vpc"
  }
}
```

Each `unit` block specifies a source (a git URL or local path containing a `terragrunt.hcl`), a path where it will be placed, and optional values to pass into the unit.

### Generated Structure

After running `terragrunt stack generate`:

```
.terragrunt-stack/
├── vpc/
│   ├── terragrunt.hcl
│   └── terragrunt.values.hcl
└── database/
    ├── terragrunt.hcl
    └── terragrunt.values.hcl
```

The `terragrunt.values.hcl` file contains the values passed from the stack definition, accessible via `values.*` in the unit's `terragrunt.hcl`.

### Unit Block

The `unit` block has these attributes:

- **`source`** (required): Where to fetch the unit configuration from. Accepts git URLs (`git::git@github.com:org/repo.git//path?ref=tag`) or local paths (`../../units/vpc`).
- **`path`** (required): Where to place the generated unit within the `.terragrunt-stack/` directory.
- **`values`** (optional): A map of key-value pairs passed to the unit. Available in the unit's config via `values.*`.
- **`update_source_with_cas`**: Boolean. When `true`, enables CAS (Content-Addressable Storage) integration for the source.
- **`mutable`**: Boolean. When `true`, makes cached content editable after generation.

### Stack Block (Nested Stacks)

The `stack` block has the same arguments as the `unit` block but generates another `terragrunt.stack.hcl` instead of a unit. This enables hierarchical composition of infrastructure:

```hcl
stack "dev" {
  source = "git::git@github.com:acme/catalog.git//stacks/environment?ref=v0.0.1"
  path   = "dev"
  values = {
    environment = "development"
    cidr        = "10.0.0.0/16"
  }
}

stack "prod" {
  source = "git::git@github.com:acme/catalog.git//stacks/environment?ref=v0.0.1"
  path   = "prod"
  values = {
    environment = "production"
    cidr        = "10.1.0.0/16"
  }
}
```

The referenced stack (`stacks/environment/terragrunt.stack.hcl`) uses the passed values:

```hcl
unit "vpc" {
  source = "git::git@github.com:acme/catalog.git//units/vpc?ref=v0.0.1"
  path   = "vpc"
  values = {
    vpc_name = values.environment
    cidr     = values.cidr
  }
}

unit "database" {
  source = "git::git@github.com:acme/catalog.git//units/database?ref=v0.0.1"
  path   = "database"
  values = {
    environment = values.environment
    vpc_path    = "../vpc"
  }
}
```

### Working with Explicit Stacks

```bash
# Generate units from terragrunt.stack.hcl
terragrunt stack generate

# Deploy all generated units
terragrunt stack run -- apply

# Get stack outputs
terragrunt stack output

# Clean generated directories
terragrunt stack clean
```

The `stack generate` command fetches sources and creates the `.terragrunt-stack/` directory. After generation, `stack run` operates on all generated units. The `stack clean` command removes the `.terragrunt-stack/` directory entirely.

### Advantages

- **Reusability**: define infrastructure patterns once and reuse them across environments.
- **Consistency**: all environments follow the same structure by default.
- **Version control**: version collections of infrastructure patterns as a unit.
- **Automation**: generate complex infrastructure from simple blueprints.

### Limitations

- **Complexity**: adds an abstraction layer on top of units.
- **Generation overhead**: network traffic to fetch sources during generation.
- **Debugging**: generated code can be harder to trace back to its source.

## Choosing Between Implicit and Explicit Stacks

| Criteria | Implicit | Explicit |
|----------|----------|----------|
| Small number of unique units | Best choice | Overkill |
| Multiple similar environments | Repetitive | Best choice |
| Getting started with Terragrunt | Start here | Add later |
| Reusable infrastructure patterns | Not supported | Best choice |
| Maximum transparency | Best choice | Generated code |
| Infrastructure catalogs/templates | Manual | Native support |

**Recommendation**: Start with implicit stacks. Introduce explicit stacks when you have multiple environments sharing the same patterns and repetition becomes a maintenance burden.

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
  source = "git::git@github.com:acme/catalog.git//units/service?ref=${values.version}"
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
├── dev/
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

## Known Limitations

1. **Dependencies cannot be set on stacks** -- the `dependency` block cannot target a stack's `config_path`. Workaround: set `config_path` to a specific unit within the stack rather than the stack itself.

2. **Deeply nested generation can be slow** -- every generation step may involve network traffic to fetch sources. Mitigation: enable CAS with `update_source_with_cas = true` to cache and deduplicate fetches.

3. **Includes not supported in terragrunt.stack.hcl** -- `include` blocks cannot be used in stack files. Nested stacks provide organizational structure without relying on includes. This may change based on community feedback.

4. **Invalid to have both terragrunt.hcl and terragrunt.stack.hcl** -- a directory is either a unit or a stack, never both. Terragrunt will error if both files exist in the same directory.
