# HCL Configuration Reference

Complete reference for all Terragrunt HCL blocks, sub-blocks, and top-level attributes. This covers everything that can appear in a `terragrunt.hcl`, `root.hcl`, or `terragrunt.stack.hcl` file.

## Table of Contents

- [Configuration File Resolution](#configuration-file-resolution)
- [Configuration Parsing Order](#configuration-parsing-order)
- [Blocks](#blocks)
  - [terraform](#terraform-block)
    - [source](#source)
    - [include_in_copy](#include_in_copy)
    - [exclude_from_copy](#exclude_from_copy)
    - [copy_terraform_lock_file](#copy_terraform_lock_file)
    - [mutable](#mutable)
    - [extra_arguments](#extra_arguments)
    - [before_hook](#before_hook)
    - [after_hook](#after_hook)
    - [error_hook](#error_hook)
  - [remote_state](#remote_state-block)
  - [include](#include-block)
  - [locals](#locals-block)
  - [generate](#generate-block)
  - [dependency](#dependency-block)
  - [dependencies](#dependencies-block)
  - [feature](#feature-block)
  - [exclude](#exclude-block)
  - [errors](#errors-block)
  - [catalog](#catalog-block)
  - [engine](#engine-block)
  - [unit (terragrunt.stack.hcl)](#unit-block)
  - [stack (terragrunt.stack.hcl)](#stack-block)
- [Top-Level Attributes](#top-level-attributes)
  - [inputs](#inputs)
  - [download_dir](#download_dir)
  - [prevent_destroy](#prevent_destroy)
  - [iam_role](#iam_role)
  - [iam_assume_role_duration](#iam_assume_role_duration)
  - [iam_assume_role_session_name](#iam_assume_role_session_name)
  - [iam_web_identity_token](#iam_web_identity_token)
  - [terraform_binary](#terraform_binary)
  - [terraform_version_constraint](#terraform_version_constraint)
  - [terragrunt_version_constraint](#terragrunt_version_constraint)

---

## Configuration File Resolution

Terragrunt resolves which configuration file to use in this order (first match wins):

1. `--config` CLI flag (e.g., `terragrunt run --config custom.hcl plan`)
2. `TG_CONFIG` environment variable
3. `terragrunt.hcl` in the current directory
4. `terragrunt.hcl.json` in the current directory

For stack files, Terragrunt looks for `terragrunt.stack.hcl` in the current directory.

---

## Configuration Parsing Order

Terragrunt parses configuration in this strict order. Earlier-parsed blocks are available for use in later-parsed blocks, but not vice versa:

1. `include` block
2. `locals` block
3. IAM role attributes (`iam_role`, `iam_assume_role_duration`, `iam_assume_role_session_name`, `iam_web_identity_token`)
4. `dependencies` block
5. `dependency` blocks (with output retrieval)
6. Everything else (`terraform`, `remote_state`, `generate`, `inputs`, etc.)
7. The included config referenced by `include`
8. Merge operation (combining included and current config)

**Key constraint:** `locals` are evaluated before `dependency` blocks, so you cannot use `dependency.*.outputs.*` inside `locals`. However, you can use `local.*` values inside `dependency` blocks.

**`run --all` uses two-pass parsing:**

- **First pass (dependency graph construction):** Parses `include` -> `locals` -> `dependency` (without fetching outputs) -> `terraform` -> `dependencies`. This determines execution order without actually running anything.
- **Second pass (execution):** Full parsing with output retrieval for each unit as it executes.

---

## Blocks

### terraform block

Configures where to find the OpenTofu/Terraform module source, extra CLI arguments, file copy behavior, and lifecycle hooks.

#### source

String specifying where to find the OpenTofu/Terraform configuration. Supports multiple source types:

- **Local paths:** Relative or absolute filesystem paths
- **Git URLs:** `git::git@github.com:org/repo.git//subdir?ref=v1.0.0`
- **HTTPS Git:** `git::https://github.com/org/repo.git//subdir?ref=v1.0.0`
- **Terraform Registry:** `tfr://registry.terraform.io/terraform-aws-modules/vpc/aws?version=3.3.0`
- **Registry shorthand** (defaults to `registry.terraform.io`): `tfr:///terraform-aws-modules/vpc/aws?version=3.3.0`
- **S3:** `s3::https://s3-eu-west-1.amazonaws.com/bucket/module.zip`
- **GCS:** `gcs::https://www.googleapis.com/storage/v1/bucket/module.zip`

For private Terraform registries, set the `TG_TF_REGISTRY_TOKEN` environment variable.

```hcl
terraform {
  # Git source with ref tag
  source = "git::git@github.com:acme/infrastructure-modules.git//networking/vpc?ref=v1.0.0"
}
```

```hcl
terraform {
  # Local path (relative to terragrunt.hcl)
  source = "../../modules/vpc"
}
```

```hcl
terraform {
  # Terraform Registry source
  source = "tfr:///terraform-aws-modules/vpc/aws?version=5.1.0"
}
```

```hcl
terraform {
  # Dynamic source using locals
  source = "git::git@github.com:acme/modules.git//networking/vpc?ref=${local.module_version}"
}
```

#### include_in_copy

List of glob patterns for additional files to copy into the Terragrunt working directory alongside the module source. Useful for copying shared files that are not part of the module but needed at runtime.

```hcl
terraform {
  source = "git::git@github.com:acme/modules.git//vpc?ref=v1.0.0"

  include_in_copy = [
    "*.json",
    "templates/**/*.tpl",
  ]
}
```

#### exclude_from_copy

List of glob patterns for files to exclude from copying. Takes precedence over `include_in_copy`.

```hcl
terraform {
  source = "../../modules/app"

  exclude_from_copy = [
    "**/.terraform/**",
    "**/tests/**",
  ]
}
```

#### copy_terraform_lock_file

Boolean (default: `true`). Controls whether the `.terraform.lock.hcl` file is copied back from the working directory to the module source directory after `init`.

```hcl
terraform {
  source = "../../modules/vpc"

  copy_terraform_lock_file = false
}
```

#### mutable

Boolean (default: `false`). Used with Content Addressable Storage (CAS) to make cached content editable.

```hcl
terraform {
  source = "git::git@github.com:acme/modules.git//vpc?ref=v1.0.0"

  mutable = true
}
```

#### extra_arguments

Named sub-block that injects additional CLI arguments into specific OpenTofu/Terraform commands.

| Attribute | Type | Description |
|-----------|------|-------------|
| `commands` | list(string) | Which commands to apply these args to. Can also use `get_terraform_commands_that_need_locking()`, `get_terraform_commands_that_need_vars()`, etc. |
| `arguments` | list(string) | CLI flags to append |
| `env_vars` | map(string) | Environment variables to set |
| `required_var_files` | list(string) | Paths to `.tfvars` files that must exist |
| `optional_var_files` | list(string) | Paths to `.tfvars` files that are used if they exist |

```hcl
terraform {
  source = "../../modules/vpc"

  # Add lock timeout for all commands that use locking
  extra_arguments "retry_lock" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }

  # Pass common var files for plan and apply
  extra_arguments "common_vars" {
    commands = [
      "apply",
      "plan",
      "import",
      "push",
      "refresh",
    ]

    required_var_files = [
      "${get_parent_terragrunt_dir()}/common.tfvars",
    ]

    optional_var_files = [
      "${get_parent_terragrunt_dir()}/region.tfvars",
      "${get_terragrunt_dir()}/overrides.tfvars",
    ]
  }

  # Set environment variables for all commands
  extra_arguments "env" {
    commands = get_terraform_commands_that_need_vars()
    env_vars = {
      TF_VAR_region = "us-east-1"
    }
  }
}
```

#### before_hook

Named sub-block that executes commands before specific OpenTofu/Terraform commands run. Hooks execute in the order they are defined.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `commands` | list(string) | required | Which terraform commands trigger this hook |
| `execute` | list(string) | required | Command and arguments to run (first element is the binary) |
| `working_dir` | string | terragrunt working dir | Directory to run the command in |
| `run_on_error` | bool | `false` | Whether to run even if a previous hook failed |
| `suppress_stdout` | bool | `false` | Whether to suppress stdout from the hook |
| `if` | bool expression | `true` | Conditional expression; hook runs only when true |

Special hook command names (use in `commands`):
- `"init-from-module"` -- triggers during the source download phase
- `"init"` -- triggers during `terraform init`
- `"read-config"` -- triggers after Terragrunt finishes loading configuration (only valid as `after_hook`)

```hcl
terraform {
  source = "../../modules/app"

  before_hook "validate_inputs" {
    commands = ["apply", "plan"]
    execute  = ["bash", "-c", "echo 'Validating inputs...'"]
  }

  before_hook "tflint" {
    commands = ["apply", "plan"]
    execute  = ["tflint", "--terragrunt-external-module"]
  }

  before_hook "conditional_check" {
    commands = ["apply"]
    execute  = ["bash", "scripts/pre-deploy-check.sh"]
    if       = get_env("RUN_PRE_CHECKS", "true") == "true"
  }
}
```

#### after_hook

Named sub-block that executes commands after specific OpenTofu/Terraform commands complete. Same attributes as `before_hook`.

```hcl
terraform {
  source = "../../modules/app"

  # Run after apply/plan, even on failure
  after_hook "notify" {
    commands     = ["apply", "plan"]
    execute      = ["bash", "scripts/notify.sh"]
    run_on_error = true
  }

  # Copy a file after source download
  after_hook "copy_shared" {
    commands = ["init-from-module"]
    execute  = ["cp", "${get_parent_terragrunt_dir()}/shared.tf", "."]
  }

  # Run after config is read (special hook)
  after_hook "read-config" {
    commands = ["read-config"]
    execute  = ["bash", "scripts/get_aws_credentials.sh"]
  }
}
```

#### error_hook

Named sub-block that executes commands when specific OpenTofu/Terraform commands produce errors matching given patterns.

| Attribute | Type | Description |
|-----------|------|-------------|
| `commands` | list(string) | Which terraform commands this error hook applies to |
| `execute` | list(string) | Command and arguments to run on error |
| `on_errors` | list(string) | Regex patterns to match against stderr; `".*"` matches all errors |

```hcl
terraform {
  source = "../../modules/app"

  error_hook "handle_apply_error" {
    commands  = ["apply", "plan"]
    execute   = ["bash", "scripts/alert-on-error.sh"]
    on_errors = [".*"]
  }

  error_hook "handle_download_error" {
    commands  = ["init-from-module"]
    execute   = ["echo", "Source download failed — check the source URL"]
    on_errors = [".*"]
  }
}
```

#### Complete terraform block example

```hcl
terraform {
  source = "git::git@github.com:acme/modules.git//app?ref=v2.0.0"

  include_in_copy = ["*.json"]
  copy_terraform_lock_file = true

  extra_arguments "lock_timeout" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }

  extra_arguments "var_files" {
    commands           = get_terraform_commands_that_need_vars()
    required_var_files = ["${get_parent_terragrunt_dir()}/common.tfvars"]
    optional_var_files = ["${get_terragrunt_dir()}/overrides.tfvars"]
  }

  before_hook "pre_validate" {
    commands = ["apply", "plan"]
    execute  = ["echo", "Running pre-validation"]
  }

  after_hook "post_apply" {
    commands     = ["apply"]
    execute      = ["bash", "scripts/post-deploy.sh"]
    run_on_error = true
  }

  error_hook "alert" {
    commands  = ["apply"]
    execute   = ["bash", "scripts/send-alert.sh"]
    on_errors = [".*Error applying.*"]
  }
}
```

---

### remote_state block

Configures the OpenTofu/Terraform backend for state storage. Terragrunt can automatically create the backend resources (S3 bucket, DynamoDB table, GCS bucket, etc.) if they do not exist.

| Attribute | Type | Description |
|-----------|------|-------------|
| `backend` | string | Backend type: `"s3"`, `"gcs"`, `"azurerm"`, `"consul"`, etc. |
| `disable_init` | bool | When `true`, skip automatic backend resource creation |
| `generate` | map | Generate a `backend.tf` file; contains `path` (string) and `if_exists` (string) |
| `config` | map | Backend-specific configuration key-value pairs |
| `encryption` | map | OpenTofu state encryption configuration |

The `generate.if_exists` attribute accepts:
- `"overwrite"` -- always overwrite the file
- `"overwrite_terragrunt"` -- overwrite only if the file was generated by Terragrunt (checks for signature comment)
- `"skip"` -- do not generate if the file already exists
- `"error"` -- raise an error if the file already exists

#### S3 backend example

```hcl
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket         = "my-company-tofu-state"
    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-lock-table"

    # S3 native locking (OpenTofu 1.10+, removes need for DynamoDB)
    # use_lockfile = true
  }
}
```

#### GCS backend example

```hcl
remote_state {
  backend = "gcs"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    project  = "my-gcp-project"
    location = "us"
    bucket   = "my-company-tofu-state"
    prefix   = "${path_relative_to_include()}/tofu.tfstate"
  }
}
```

#### Azure backend example

```hcl
remote_state {
  backend = "azurerm"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    resource_group_name  = "tofu-state-rg"
    storage_account_name = "tofustatesa"
    container_name       = "tfstate"
    key                  = "${path_relative_to_include()}/tofu.tfstate"
  }
}
```

#### With OpenTofu encryption

```hcl
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket  = "my-tofu-state"
    key     = "${path_relative_to_include()}/tofu.tfstate"
    region  = "us-east-1"
    encrypt = true
  }

  encryption = {
    key_provider = "pbkdf2"
    passphrase   = get_env("TF_ENCRYPTION_PASSPHRASE")
  }
}
```

---

### include block

Includes another Terragrunt configuration file (typically a parent `root.hcl` or shared environment config) and optionally merges its contents. Supports multiple labeled include blocks.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | required | Path to the configuration file to include. Typically uses `find_in_parent_folders()`. |
| `expose` | bool | `false` | When `true`, the included config's data is accessible via `include.<label>.*` |
| `merge_strategy` | string | `"shallow"` | How to merge the included config: `"no_merge"`, `"shallow"`, `"deep"` |

Merge strategies:
- `"no_merge"` -- Include for reference only (with `expose = true`). Nothing from the included config is merged into the current config.
- `"shallow"` (default) -- Top-level attributes from the included config are merged. If the same attribute exists in both, the child wins.
- `"deep"` -- Recursively merge maps and objects. Child values override parent values at every nesting level.

**Constraints:**
- Only a single level of include nesting is supported. An included file cannot itself include another file.
- When an included config has `dependency` blocks, only `locals` and `include` are exposed to the child.

#### Single include

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}
```

#### Multiple includes with expose and merge strategies

```hcl
include "root" {
  path   = find_in_parent_folders("root.hcl")
  expose = true
}

include "env" {
  path           = find_in_parent_folders("env.hcl")
  expose         = true
  merge_strategy = "deep"
}

include "region" {
  path           = find_in_parent_folders("region.hcl")
  expose         = true
  merge_strategy = "no_merge"
}

inputs = {
  # Access data from exposed includes
  state_bucket = include.root.remote_state.config.bucket
  environment  = include.env.locals.environment
  aws_region   = include.region.locals.aws_region
}
```

---

### locals block

Defines local variables that can be referenced as `local.<name>` throughout the configuration. There are no predefined attributes -- all attributes become local variables.

Locals can use functions like `read_terragrunt_config()`, `run_cmd()`, `get_env()`, and can contain complex types (lists, maps, objects).

**Parsing order constraint:** Locals are parsed before `dependency` blocks, so `dependency.*.outputs.*` cannot be used in locals. However, `local.*` values can be used in `dependency` blocks.

```hcl
locals {
  # Read and parse another HCL file
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  # Extract values
  account_id   = local.account_vars.locals.account_id
  account_name = local.account_vars.locals.account_name
  aws_region   = local.region_vars.locals.aws_region

  # Computed values
  name_prefix = "${local.account_name}-${local.aws_region}"

  # Environment variable with default
  deploy_env = get_env("DEPLOY_ENV", "dev")

  # Complex types
  common_tags = {
    Environment = local.deploy_env
    Account     = local.account_name
    ManagedBy   = "Terragrunt"
  }

  # Conditional logic
  is_production = local.deploy_env == "prod"
  instance_type = local.is_production ? "m5.xlarge" : "t3.medium"
}
```

#### Pattern: hierarchical config files

A common pattern uses `locals` in small config files at each directory level:

`account.hcl`:
```hcl
locals {
  account_id   = "123456789012"
  account_name = "production"
}
```

`region.hcl`:
```hcl
locals {
  aws_region = "us-east-1"
}
```

`env.hcl`:
```hcl
locals {
  environment = "prod"
}
```

Then in `terragrunt.hcl`:
```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_vars     = read_terragrunt_config(find_in_parent_folders("env.hcl"))
}

inputs = {
  account_id  = local.account_vars.locals.account_id
  aws_region  = local.region_vars.locals.aws_region
  environment = local.env_vars.locals.environment
}
```

---

### generate block

Generates a file in the Terragrunt working directory. Commonly used to create `provider.tf`, `backend.tf`, `versions.tf`, or other shared configuration files.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `path` | string | required | File path to generate (relative to working directory) |
| `if_exists` | string | required | Behavior when file exists: `"overwrite"`, `"overwrite_terragrunt"`, `"skip"`, `"error"` |
| `contents` | string | required | Content of the generated file |
| `comment_prefix` | string | `"# "` | Prefix for the Terragrunt signature comment |
| `disable_signature` | bool | `false` | When `true`, omit the "generated by Terragrunt" signature comment |
| `disable` | bool | `false` | When `true`, do not generate this file |
| `if_disabled` | string | `"skip"` | Behavior when `disable = true` |

The `if_exists` values:
- `"overwrite"` -- always overwrite the file
- `"overwrite_terragrunt"` -- overwrite only if the file contains a Terragrunt signature
- `"skip"` -- do not generate if the file already exists
- `"error"` -- raise an error if the file already exists

#### Generate a provider configuration

```hcl
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.aws_region}"

      default_tags {
        tags = {
          Environment = "${local.environment}"
          ManagedBy   = "Terragrunt"
        }
      }
    }
  EOF
}
```

#### Generate a versions constraint file

```hcl
generate "versions" {
  path      = "versions.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    terraform {
      required_version = ">= 1.5.0"

      required_providers {
        aws = {
          source  = "hashicorp/aws"
          version = "~> 5.0"
        }
      }
    }
  EOF
}
```

#### Generate with disabled signature

```hcl
generate "terraformrc" {
  path              = ".terraformrc"
  if_exists         = "overwrite"
  disable_signature = true
  comment_prefix    = "# "
  contents          = <<-EOF
    plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
  EOF
}
```

#### Conditional generate

```hcl
generate "datadog_provider" {
  path      = "datadog_provider.tf"
  if_exists = "overwrite_terragrunt"
  disable   = !local.enable_monitoring
  contents  = <<-EOF
    provider "datadog" {
      api_key = var.datadog_api_key
      app_key = var.datadog_app_key
    }
  EOF
}
```

---

### dependency block

Declares a dependency on another Terragrunt unit and provides access to its outputs. See `references/dependencies-and-includes.md` for full details.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `config_path` | string | required | Relative path to the dependency unit's directory |
| `mock_outputs` | map | `{}` | Fake outputs used when real outputs are unavailable (e.g., fresh `plan`) |
| `mock_outputs_allowed_terraform_commands` | list(string) | `[]` | Commands for which mock outputs are allowed (e.g., `["validate", "plan"]`) |
| `mock_outputs_merge_strategy_with_state` | string | `"no_merge"` | How to merge mock outputs with real state: `"no_merge"`, `"shallow"`, `"deep"` |
| `skip_outputs` | bool | `false` | Skip fetching outputs (use when you only need ordering) |
| `enabled` | bool | `true` | When `false`, the dependency is ignored entirely |

Access outputs via `dependency.<label>.outputs.<key>`.

```hcl
dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1", "mock-subnet-2"]
    public_subnets  = ["mock-subnet-3"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "security_group" {
  config_path = "../security-group"

  mock_outputs = {
    sg_id = "mock-sg-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

inputs = {
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnets
  sg_id      = dependency.security_group.outputs.sg_id
}
```

#### Conditional dependency

```hcl
dependency "monitoring" {
  config_path = "../monitoring"
  enabled     = local.enable_monitoring

  mock_outputs = {
    sns_topic_arn = "arn:aws:sns:us-east-1:000000000000:mock-topic"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}
```

---

### dependencies block

Declares ordering-only dependencies. Unlike `dependency` (singular), this block does NOT provide access to outputs -- it only ensures the listed units are applied before the current one during `run --all`.

| Attribute | Type | Description |
|-----------|------|-------------|
| `paths` | list(string) | List of relative paths to dependency unit directories |

```hcl
dependencies {
  paths = ["../vpc", "../iam", "../kms"]
}
```

See `references/dependencies-and-includes.md` for details on when to use `dependency` vs `dependencies`.

---

### feature block

Defines feature flags for dynamic configuration. Feature flags can be overridden at runtime via CLI or environment variables without modifying the HCL file.

| Attribute | Type | Description |
|-----------|------|-------------|
| `default` | any | Required. The default value for the feature flag. |

Access feature values via `feature.<name>.value`.

Override mechanisms:
- CLI: `--feature name=value`
- Environment variable: `TG_FEATURE="name=value"`
- Multiple flags: `TG_FEATURE="flag1=val1" --feature flag2=val2`

```hcl
feature "s3_version" {
  default = "v1.0.0"
}

feature "enable_waf" {
  default = false
}

feature "instance_count" {
  default = 2
}

terraform {
  source = "git::git@github.com:acme/modules.git//s3?ref=${feature.s3_version.value}"
}

inputs = {
  enable_waf     = feature.enable_waf.value
  instance_count = feature.instance_count.value
}
```

Usage:
```bash
# Override at runtime
terragrunt run --feature s3_version=v2.0.0 --feature enable_waf=true plan

# Or via environment variable
TG_FEATURE="s3_version=v2.0.0" terragrunt run plan
```

---

### exclude block

Excludes a unit from `run --all` operations based on a conditional expression. Does NOT prevent direct execution of the unit -- only affects multi-unit runs.

| Attribute | Type | Description |
|-----------|------|-------------|
| `if` | bool expression | When `true`, the unit is excluded |
| `actions` | list(string) | Which actions to exclude: `"apply"`, `"destroy"`, `"all"`, `"all_except_output"` |

```hcl
# Exclude from all run --all operations in non-prod
exclude {
  if      = local.environment != "prod"
  actions = ["all"]
}
```

```hcl
# Block deployments during maintenance windows
exclude {
  if      = timestamp() >= "2025-12-20T00:00:00Z" && timestamp() <= "2026-01-02T00:00:00Z"
  actions = ["apply", "destroy"]
}
```

```hcl
# Exclude from destroy but allow apply and output
exclude {
  if      = true
  actions = ["destroy"]
}
```

```hcl
# Exclude from everything except output (useful for data-only units)
exclude {
  if      = local.skip_deployment
  actions = ["all_except_output"]
}
```

---

### errors block

Configures error handling behavior with retry and ignore strategies.

#### retry sub-block

Named sub-block that retries when errors match specified regex patterns.

| Attribute | Type | Description |
|-----------|------|-------------|
| `retryable_errors` | list(string) | Regex patterns to match against error output |
| `max_attempts` | number | Maximum number of retry attempts |
| `sleep_interval_sec` | number | Seconds to wait between retries |

#### ignore sub-block

Named sub-block that ignores errors matching specified regex patterns. Prefix a pattern with `!` to negate it (the error must NOT match).

| Attribute | Type | Description |
|-----------|------|-------------|
| `ignorable_errors` | list(string) | Regex patterns to match. Prefix with `!` to negate. |
| `message` | string | Message to display when an error is ignored |
| `signals` | map | Key-value pairs passed to hooks or external systems |

```hcl
errors {
  # Retry transient AWS errors
  retry "aws_transient" {
    retryable_errors = [
      ".*Error: RequestLimitExceeded.*",
      ".*Error: Throttling.*",
      ".*Error: connection reset by peer.*",
    ]
    max_attempts       = 5
    sleep_interval_sec = 30
  }

  # Also include default retryable errors
  retry "default" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }

  # Ignore known warnings that are safe
  ignore "safe_warnings" {
    ignorable_errors = [
      ".*Warning: Resource already exists.*",
      "!.*Error: do not ignore this.*",
    ]
    message = "Ignoring known safe warnings"
    signals = {
      alert_team = false
    }
  }
}
```

---

### catalog block

Configures the `terragrunt catalog` command, which provides an interactive UI for browsing and scaffolding modules.

| Attribute | Type | Description |
|-----------|------|-------------|
| `default_template` | string | Git URL for the default scaffold template |
| `no_shell` | bool | Disable interactive shell features |
| `no_hooks` | bool | Disable hook execution during catalog operations |

```hcl
catalog {
  default_template = "git::git@github.com:acme/scaffold-templates.git//default?ref=v1.0.0"
}
```

---

### engine block

Configures an alternative IaC engine (experimental). Replaces the default OpenTofu/Terraform execution with a custom engine.

| Attribute | Type | Description |
|-----------|------|-------------|
| `source` | string | Engine source: GitHub URL, HTTPS URL, or local path |
| `version` | string | Engine version |
| `meta` | map | Key-value pairs passed to the engine |

Requires the experimental flag: `export TG_EXPERIMENTAL_ENGINE=1`

Engine plugins are cached at: `~/.cache/terragrunt/plugins/iac-engine/rpc/<version>`

```hcl
engine {
  source  = "github.com/gruntwork-io/terragrunt-engine-opentofu"
  version = "v1.0.0"
  meta = {
    parallelism = 4
    log_level   = "info"
  }
}
```

---

### unit block

Defined in `terragrunt.stack.hcl` files. Declares a unit within an explicit stack. When `terragrunt stack generate` runs, it creates a directory for each unit with a `terragrunt.hcl` file.

| Attribute | Type | Description |
|-----------|------|-------------|
| `source` | string | Required. Path to the source Terragrunt unit configuration. |
| `path` | string | Required. Directory name for the generated unit. |
| `values` | map | Optional. Values passed to the unit (accessible in the source via `unit.values`). |
| `update_source_with_cas` | bool | Use Content Addressable Storage for source updates. |
| `mutable` | bool | Allow modification of CAS-cached content. |

```hcl
# terragrunt.stack.hcl

locals {
  name       = "my-project-dev"
  aws_region = "us-east-1"
  units_path = find_in_parent_folders("catalog/units")
}

unit "vpc" {
  source = "${local.units_path}/vpc"
  path   = "vpc"

  values = {
    name       = local.name
    cidr_block = "10.0.0.0/16"
    aws_region = local.aws_region
  }
}

unit "database" {
  source = "${local.units_path}/rds"
  path   = "database"

  values = {
    name           = local.name
    instance_class = "db.t3.medium"
  }
}

unit "app" {
  source = "${local.units_path}/ecs"
  path   = "app"

  values = {
    name       = local.name
    aws_region = local.aws_region
  }
}
```

---

### stack block

Defined in `terragrunt.stack.hcl` files. Declares a nested stack -- generates another `terragrunt.stack.hcl` at the specified path, enabling composition of stacks.

| Attribute | Type | Description |
|-----------|------|-------------|
| `source` | string | Required. Path or URL to the stack configuration. |
| `path` | string | Required. Directory name for the generated nested stack. |
| `values` | map | Optional. Values passed to the nested stack. |
| `update_source_with_cas` | bool | Use Content Addressable Storage for source updates. |
| `mutable` | bool | Allow modification of CAS-cached content. |

```hcl
# terragrunt.stack.hcl

stack "services" {
  source = "github.com/gruntwork-io/terragrunt-stacks//stacks/services?ref=v1.0.0"
  path   = "services"

  values = {
    project = "dev-services"
    cidr    = "10.0.0.0/16"
  }
}

stack "monitoring" {
  source = "github.com/gruntwork-io/terragrunt-stacks//stacks/monitoring?ref=v1.0.0"
  path   = "monitoring"

  values = {
    project     = "dev-monitoring"
    environment = "dev"
  }
}
```

---

## Top-Level Attributes

These attributes are set at the top level of a `terragrunt.hcl` file (not inside any block).

### inputs

Map of values passed to the OpenTofu/Terraform module as `TF_VAR_*` environment variables. The keys must match variable names defined in the module's `.tf` files.

Supports all types: string, number, bool, list, map, object.

**Variable precedence** (highest to lowest):
1. `inputs` in `terragrunt.hcl` (set as `TF_VAR_*` env vars)
2. Explicit `TF_VAR_*` environment variables set outside Terragrunt
3. `terraform.tfvars`
4. `*.auto.tfvars`
5. `-var` / `-var-file` CLI flags

```hcl
inputs = {
  # Simple types
  name          = "my-application"
  instance_type = local.instance_type
  replicas      = 3
  enable_ssl    = true

  # List
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  # Map
  tags = merge(local.common_tags, {
    Name = "my-application"
  })

  # From dependency outputs
  vpc_id     = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.private_subnets

  # From environment
  api_key = get_env("API_KEY")
}
```

---

### download_dir

String overriding the default `.terragrunt-cache` directory where Terragrunt downloads and caches module source code.

**Precedence** (highest to lowest):
1. `--download-dir` CLI flag
2. `TG_DOWNLOAD_DIR` environment variable
3. `download_dir` in the current module's `terragrunt.hcl`
4. `download_dir` in the included (parent) config

```hcl
download_dir = "/tmp/terragrunt-cache/${path_relative_to_include()}"
```

---

### prevent_destroy

Boolean that blocks `terragrunt run destroy` and `terragrunt run --all destroy` for this unit. Use to protect critical resources like databases, encryption keys, and authentication systems from accidental deletion.

When set to `true`, any attempt to run `destroy` on this unit will produce an error.

```hcl
prevent_destroy = true
```

```hcl
# Conditional based on environment
prevent_destroy = local.is_production
```

---

### iam_role

String specifying an IAM role ARN for Terragrunt to assume via STS before running OpenTofu/Terraform commands.

**Precedence** (highest to lowest):
1. `--iam-assume-role` CLI flag
2. `TG_IAM_ASSUME_ROLE` environment variable
3. `iam_role` in the current module's `terragrunt.hcl`
4. `iam_role` in the included (parent) config

```hcl
iam_role = "arn:aws:iam::${local.account_id}:role/TerragruntDeployRole"
```

---

### iam_assume_role_duration

Integer specifying the STS session duration in seconds when assuming the IAM role.

```hcl
iam_role                  = "arn:aws:iam::123456789012:role/DeployRole"
iam_assume_role_duration  = 3600  # 1 hour
```

---

### iam_assume_role_session_name

String specifying the STS session name when assuming the IAM role. Useful for CloudTrail auditing.

```hcl
iam_role                      = "arn:aws:iam::123456789012:role/DeployRole"
iam_assume_role_session_name  = "terragrunt-deploy-prod"
```

---

### iam_web_identity_token

String specifying a web identity token value or file path for OIDC-based role assumption. Must be paired with `iam_role`. Used in CI/CD environments (e.g., GitHub Actions OIDC, GitLab CI) where short-lived tokens replace long-lived credentials.

```hcl
iam_role               = "arn:aws:iam::123456789012:role/GitHubActionsRole"
iam_web_identity_token = get_env("AWS_WEB_IDENTITY_TOKEN")
```

```hcl
# Using a token file
iam_role               = "arn:aws:iam::123456789012:role/GitHubActionsRole"
iam_web_identity_token = "/var/run/secrets/oidc/token"
```

---

### terraform_binary

String specifying the path to the OpenTofu or Terraform binary. Defaults to `tofu`.

**Precedence** (highest to lowest):
1. `--tf-path` CLI flag
2. `TG_TF_PATH` environment variable
3. `terraform_binary` in the current module's `terragrunt.hcl`
4. `terraform_binary` in the included (parent) config

```hcl
terraform_binary = "/usr/local/bin/terraform"
```

```hcl
# Use terraform instead of tofu
terraform_binary = "terraform"
```

---

### terraform_version_constraint

String specifying a version constraint for the OpenTofu/Terraform binary. Terragrunt will check the binary's version and fail if it does not satisfy the constraint.

```hcl
terraform_version_constraint = ">= 1.5.0"
```

```hcl
terraform_version_constraint = "~> 1.8.0"
```

---

### terragrunt_version_constraint

String specifying a version constraint for Terragrunt itself. Terragrunt will check its own version and fail if it does not satisfy the constraint. Useful to enforce minimum Terragrunt versions across a team.

```hcl
terragrunt_version_constraint = ">= 0.68.0"
```

```hcl
terragrunt_version_constraint = "~> 0.75"
```

---

## Complete Example: Root Configuration

A typical `root.hcl` that child units include:

```hcl
locals {
  account_vars = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region_vars  = read_terragrunt_config(find_in_parent_folders("region.hcl"))

  account_id = local.account_vars.locals.account_id
  aws_region = local.region_vars.locals.aws_region
}

# Configure remote state
remote_state {
  backend = "s3"

  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }

  config = {
    bucket         = "acme-tofu-state-${local.account_id}"
    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = local.aws_region
    encrypt        = true
    dynamodb_table = "tofu-lock-table"
  }
}

# Generate provider configuration
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<-EOF
    provider "aws" {
      region = "${local.aws_region}"

      default_tags {
        tags = {
          ManagedBy = "Terragrunt"
          Account   = "${local.account_id}"
        }
      }
    }
  EOF
}

# Retry transient errors
errors {
  retry "default" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }
}

# Version constraints
terraform_version_constraint  = ">= 1.5.0"
terragrunt_version_constraint = ">= 0.68.0"
```

## Complete Example: Child Unit

A typical child `terragrunt.hcl`:

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

locals {
  env_vars    = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  environment = local.env_vars.locals.environment
}

terraform {
  source = "git::git@github.com:acme/modules.git//database/rds?ref=v2.1.0"
}

dependency "vpc" {
  config_path = "../vpc"

  mock_outputs = {
    vpc_id          = "mock-vpc-id"
    private_subnets = ["mock-subnet-1", "mock-subnet-2"]
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

dependency "kms" {
  config_path = "../kms"

  mock_outputs = {
    key_arn = "arn:aws:kms:us-east-1:000000000000:key/mock-key-id"
  }
  mock_outputs_allowed_terraform_commands = ["validate", "plan"]
}

prevent_destroy = local.environment == "prod"

inputs = {
  name              = "app-database-${local.environment}"
  vpc_id            = dependency.vpc.outputs.vpc_id
  subnet_ids        = dependency.vpc.outputs.private_subnets
  kms_key_arn       = dependency.kms.outputs.key_arn
  instance_class    = local.environment == "prod" ? "db.r6g.xlarge" : "db.t3.medium"
  multi_az          = local.environment == "prod"
  deletion_protection = local.environment == "prod"
}
```
