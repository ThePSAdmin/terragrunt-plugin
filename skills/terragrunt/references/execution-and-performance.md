# Execution Model and Performance

## Table of Contents

- [Run Queue and DAG](#run-queue-and-dag)
- [Parallelism](#parallelism)
- [Queue Control Flags](#queue-control-flags)
- [Filtering Expressions](#filtering-expressions)
- [Provider Caching](#provider-caching)
- [Hooks](#hooks)
- [extra_arguments Block](#extra_arguments-block)
- [Auto-Init](#auto-init)
- [Saving Plan Output](#saving-plan-output)

## Run Queue and DAG

Terragrunt builds a Directed Acyclic Graph (DAG) from `dependency` and `dependencies` blocks.

### Execution Model
1. Discovery: Terragrunt finds units in the current working directory
2. Queue Construction: builds ordered queue based on command
   - `plan`/`apply`: dependencies run BEFORE dependents
   - `destroy`: dependents run BEFORE dependencies
3. Execution: dequeues units, runs concurrently up to parallelism limit, waits for dependency completion

### Example
```
root/
├── vpc/terragrunt.hcl           (no dependencies)
├── security-groups/terragrunt.hcl  (depends on vpc)
├── ecs-cluster/terragrunt.hcl      (depends on vpc, security-groups)
└── independent/terragrunt.hcl      (no dependencies)
```

Plan order: `vpc` + `independent` (concurrent) → `security-groups` → `ecs-cluster`
Destroy order: `ecs-cluster` + `independent` (concurrent) → `security-groups` → `vpc`

## Parallelism

```bash
# Limit to 4 concurrent units
terragrunt run --all --parallelism 4 -- apply

# Environment variable
export TG_PARALLELISM=4

# Stack output parallelism
terragrunt stack output --parallelism 4
```

## Queue Control Flags

### --queue-construct-as (--as)
Build the queue as if a different command is being run. Useful for dry-runs:
```bash
# List units in destroy order
terragrunt list --as destroy -l

# List units in plan order
terragrunt list --as plan -l
```

### --queue-ignore-dag-order
Run units concurrently ignoring dependency order:
```bash
terragrunt run --all plan --queue-ignore-dag-order
```
Safe for `validate`, `plan`. DANGEROUS for `apply`, `destroy`.

### --queue-ignore-errors
Continue processing queue even if units fail:
```bash
terragrunt run --all plan --queue-ignore-errors
```
Useful for identifying all errors at once.

### --fail-fast
Stop immediately on first failure:
```bash
terragrunt run --all plan --fail-fast
```

### --queue-include-units-reading
Include units that read a specific file:
```bash
terragrunt run --all plan --queue-include-units-reading _env/app.hcl
```

### --queue-exclude-dir / --queue-include-dir
```bash
terragrunt run --all --queue-exclude-dir prod -- plan
terragrunt run --all --queue-include-dir dev -- plan
```

## Filtering Expressions

The `--filter` flag supports rich expressions for selecting units.

### Name Expressions
```bash
# Exact match
terragrunt run --all --filter 'vpc' -- plan

# Glob pattern
terragrunt run --all --filter 'app*' -- plan

# Wrapped (explicit name expression)
terragrunt run --all --filter 'name=web' -- plan
```

### Path Expressions
```bash
# Relative paths
terragrunt run --all --filter './prod/**' -- plan

# Absolute paths
terragrunt run --all --filter '/full/path/to/prod/**' -- plan

# Wrapped paths
terragrunt run --all --filter '{./envs/prod/apps/app2}' -- plan
```

### Attribute Expressions
```bash
# Filter by component type
terragrunt run --all --filter 'type=unit' -- plan
terragrunt run --all --filter 'type=stack' -- plan

# Filter by external dependency status
terragrunt run --all --filter 'external=true' -- plan

# Filter by source
terragrunt run --all --filter 'source=github.com/acme/foo' -- plan

# Source with glob
terragrunt run --all --filter 'source=github.com/acme/*' -- plan
```

### Reading-Based Expressions
```bash
# Units that read a specific file
terragrunt run --all --filter 'reading=shared.hcl' -- plan
```

### Git Expressions
```bash
# Changes between branches
terragrunt run --all --filter '[main...HEAD]' -- plan

# Changes between commits
terragrunt run --all --filter '[abc1234...def5678]' -- plan

# Changes between tags
terragrunt run --all --filter '[v1.0.0...v2.0.0]' -- plan

# Shorthand (compare to HEAD)
terragrunt run --all --filter '[main...]' -- plan
```

### Graph Expressions (DAG Traversal)
```bash
# service and everything it depends on
terragrunt run --all --filter 'service...' -- plan

# vpc and everything that depends on it
terragrunt run --all --filter '...vpc' -- plan

# db and its complete dependency graph (both directions)
terragrunt run --all --filter '...db...' -- plan

# Exclude the target itself
terragrunt run --all --filter '...vpc' --filter '!vpc' -- plan

# Depth-limited traversal
terragrunt run --all --filter 'service...(1)' -- plan   # 1 level of deps
terragrunt run --all --filter '...(2)vpc' -- plan       # 2 levels of dependents
```

### Negated Expressions
```bash
# Exclude by name
terragrunt run --all --filter '!app1' -- plan

# Exclude by path
terragrunt run --all --filter '!./prod/**' -- plan

# Exclude by type
terragrunt run --all --filter '!type=stack' -- plan
```

### Intersection Expressions (AND)
```bash
# Components in ./prod/** that are also units
terragrunt run --all --filter './prod/** & type=unit' -- plan

# Components in ./prod/** that are NOT units
terragrunt run --all --filter './prod/** & !type=unit' -- plan
```

### Union Expressions (OR)
```bash
# Components named unit1 OR stack1
terragrunt run --all --filter 'unit1 | stack1' -- plan

# Multiple paths
terragrunt run --all --filter './envs/prod/* | ./envs/stage/*' -- plan
```

### Combining Expressions
```bash
# Affected by git changes AND in prod path
terragrunt run --all --filter '[main...HEAD] & ./prod/**' -- plan
```

### Filters File
```bash
# Load filters from file
terragrunt run --all --filters-file /tmp/diffs.txt -- plan
```

## Provider Caching

### Provider Cache Server
Terragrunt runs a local caching proxy for provider downloads. Solves concurrent access issues with `run --all`.

```bash
# Enable provider cache
terragrunt run --all --provider-cache -- apply

# Custom cache directory
terragrunt run --all --provider-cache --provider-cache-dir /tmp/cache -- apply
```

Flags:
- `--provider-cache`: enable
- `--provider-cache-dir`: custom path
- `--provider-cache-hostname`: server hostname (default: localhost)
- `--provider-cache-port`: server port (default: random)
- `--provider-cache-token`: auth token (default: random)
- `--provider-cache-registry-names`: registries to cache (default: registry.terraform.io, registry.opentofu.org)

NEVER use `TF_PLUGIN_CACHE_DIR` with `run --all` -- use Provider Cache Server instead.

### Auto Provider Cache Dir
For OpenTofu >= 1.10, Terragrunt automatically sets a shared provider cache directory.

### Content-Addressable Storage (CAS)
Enable with `--experiment cas`. Deduplicates git clones for remote sources.
See references/stacks.md for details on CAS with stacks.

## Hooks

### before_hook
Runs before OpenTofu/Terraform command:
```hcl
terraform {
  before_hook "build_docker" {
    commands = ["apply", "plan"]
    execute  = ["docker", "build", "-t", "myapp:latest", "."]
  }
}
```

### after_hook
Runs after command:
```hcl
terraform {
  after_hook "smoke_test" {
    commands     = ["apply"]
    execute      = ["./run_smoke_tests.sh"]
    run_on_error = false
  }
}
```

### error_hook
Runs only on errors:
```hcl
terraform {
  error_hook "handle_error" {
    commands  = ["apply"]
    execute   = ["./notify_slack.sh", "Deploy failed"]
    on_errors = [".*Error: creating.*"]
  }
}
```

### Hook Attributes
- `commands` (required): list of commands to hook into
- `execute` (required): command to run as list [binary, arg1, arg2, ...]
- `working_dir` (optional): working directory
- `run_on_error` (optional): boolean -- run even if IaC failed (after_hook only, default true)
- `suppress_stdout` (optional): boolean -- hide output
- `if` (optional): boolean expression -- conditional execution

### Hook Environment Variables
- `TG_CTX_TF_PATH`: path to tofu/terraform binary
- `TG_CTX_COMMAND`: command being run (plan, apply, etc.)
- `TG_CTX_HOOK_NAME`: name of the hook
- All `inputs` available as `TF_VAR_*` env vars

### Special Hook Names
- `read-config`: runs after config loading (after_hook only)
- `init-from-module`: runs when downloading remote configs
- `init`: runs during tofu/terraform init

### Hook Ordering
Multiple hooks execute in definition order. Before hooks run sequentially before the IaC command. After hooks run sequentially after.

### Hook Exit Codes
Non-zero exit causes terragrunt to exit non-zero. A failing before_hook prevents the IaC command from running.

### TFLint Integration
```hcl
terraform {
  before_hook "tflint" {
    commands = ["apply", "plan"]
    execute  = ["tflint"]
  }
}
```

Requires `.tflint.hcl` in same directory or parent with `config { module = true }` block.

## extra_arguments Block

Add CLI arguments to OpenTofu/Terraform commands:

```hcl
terraform {
  extra_arguments "retry_lock" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }

  extra_arguments "custom_vars" {
    commands           = ["apply", "plan"]
    required_var_files = ["${get_parent_terragrunt_dir()}/common.tfvars"]
    optional_var_files = ["${get_terragrunt_dir()}/${get_env("TF_VAR_env", "dev")}.tfvars"]
  }

  extra_arguments "env_vars" {
    commands = ["apply", "plan"]
    env_vars = {
      TF_VAR_region = "us-east-1"
    }
  }
}
```

Multiple extra_arguments blocks are applied in order of appearance.

Whitespace handling -- each space-separated argument must be a separate list item:
```hcl
arguments = ["-var", "bucket=example.bucket.name"]
```

## Auto-Init

Terragrunt automatically runs `tofu init` before other commands when:
- Init has never been called
- Source needs downloading
- `.terragrunt-init-required` file exists
- Modules or remote state changed since last init

Disable:
```bash
terragrunt run --no-auto-init -- plan
export TG_NO_AUTO_INIT=true
```

extra_arguments for init apply to both explicit and automatic init:
```hcl
extra_arguments "init_args" {
  commands  = ["init"]
  arguments = ["-plugin-dir=/my/plugin/dir"]
}
```

## Saving Plan Output

```bash
# Binary plan files
terragrunt run --all --out-dir /tmp/tfplan -- plan

# JSON plan output
terragrunt run --all --json-out-dir /tmp/json -- plan

# Apply saved plans (must use same --filter if used during plan)
terragrunt run --all --out-dir /tmp/tfplan -- apply
```
