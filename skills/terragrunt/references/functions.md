# Terragrunt Built-in Functions Reference

Complete reference for all Terragrunt built-in functions. Terragrunt also supports all OpenTofu/Terraform built-in functions in HCL expressions.

## Table of Contents

- [OpenTofu/Terraform Built-in Functions](#opentofuterraform-built-in-functions)
- [Path Functions](#path-functions)
  - [find_in_parent_folders](#find_in_parent_folders)
  - [path_relative_to_include](#path_relative_to_include)
  - [path_relative_from_include](#path_relative_from_include)
  - [get_terragrunt_dir](#get_terragrunt_dir)
  - [get_working_dir](#get_working_dir)
  - [get_parent_terragrunt_dir](#get_parent_terragrunt_dir)
  - [get_original_terragrunt_dir](#get_original_terragrunt_dir)
  - [get_repo_root](#get_repo_root)
  - [get_path_from_repo_root](#get_path_from_repo_root)
  - [get_path_to_repo_root](#get_path_to_repo_root)
- [Environment and Platform Functions](#environment-and-platform-functions)
  - [get_env](#get_env)
  - [get_platform](#get_platform)
- [Terraform Command Functions](#terraform-command-functions)
  - [get_terraform_commands_that_need_vars](#get_terraform_commands_that_need_vars)
  - [get_terraform_commands_that_need_input](#get_terraform_commands_that_need_input)
  - [get_terraform_commands_that_need_locking](#get_terraform_commands_that_need_locking)
  - [get_terraform_commands_that_need_parallelism](#get_terraform_commands_that_need_parallelism)
  - [get_terraform_command](#get_terraform_command)
  - [get_terraform_cli_args](#get_terraform_cli_args)
- [AWS Identity Functions](#aws-identity-functions)
  - [get_aws_account_id](#get_aws_account_id)
  - [get_aws_account_alias](#get_aws_account_alias)
  - [get_aws_caller_identity_arn](#get_aws_caller_identity_arn)
  - [get_aws_caller_identity_user_id](#get_aws_caller_identity_user_id)
- [Execution Functions](#execution-functions)
  - [run_cmd](#run_cmd)
- [Configuration Reading Functions](#configuration-reading-functions)
  - [read_terragrunt_config](#read_terragrunt_config)
  - [read_tfvars_file](#read_tfvars_file)
- [Encryption and Secrets](#encryption-and-secrets)
  - [sops_decrypt_file](#sops_decrypt_file)
- [Data Processing Functions](#data-processing-functions)
  - [deep_merge](#deep_merge)
- [Special Functions](#special-functions)
  - [get_terragrunt_source_cli_flag](#get_terragrunt_source_cli_flag)
  - [get_default_retryable_errors](#get_default_retryable_errors)
  - [mark_as_read](#mark_as_read)
  - [mark_glob_as_read](#mark_glob_as_read)
  - [constraint_check](#constraint_check)

---

## OpenTofu/Terraform Built-in Functions

Terragrunt supports **all** OpenTofu/Terraform built-in functions in HCL expressions. These can be used anywhere in `terragrunt.hcl` -- in `locals`, `inputs`, `generate` contents, and other blocks.

### String Functions

| Function | Description | Example |
|----------|-------------|---------|
| `format(spec, values...)` | Formats a string using printf-style verbs | `format("env-%s-%s", var.env, var.region)` |
| `join(separator, list)` | Joins list elements into a string | `join(",", ["a", "b", "c"])` -> `"a,b,c"` |
| `split(separator, string)` | Splits a string into a list | `split(",", "a,b,c")` -> `["a", "b", "c"]` |
| `replace(string, search, replace)` | Replaces occurrences in a string | `replace("hello-world", "-", "_")` -> `"hello_world"` |
| `regex(pattern, string)` | Applies a regex to a string | `regex("^([a-z]+)-", "dev-app")` -> `"dev"` |
| `trimspace(string)` | Removes leading/trailing whitespace | `trimspace("  hello  ")` -> `"hello"` |
| `upper(string)` | Converts to uppercase | `upper("hello")` -> `"HELLO"` |
| `lower(string)` | Converts to lowercase | `lower("HELLO")` -> `"hello"` |
| `title(string)` | Capitalizes first letter of each word | `title("hello world")` -> `"Hello World"` |
| `substr(string, offset, length)` | Extracts a substring | `substr("hello", 0, 3)` -> `"hel"` |
| `startswith(string, prefix)` | Checks if string starts with prefix | `startswith("hello", "hel")` -> `true` |
| `endswith(string, suffix)` | Checks if string ends with suffix | `endswith("hello", "llo")` -> `true` |
| `strcontains(string, substr)` | Checks if string contains substring | `strcontains("hello", "ell")` -> `true` |

### Collection Functions

| Function | Description | Example |
|----------|-------------|---------|
| `lookup(map, key, default)` | Looks up a key in a map | `lookup({a = 1}, "b", 0)` -> `0` |
| `merge(map1, map2, ...)` | Merges maps (later wins) | `merge({a = 1}, {b = 2})` -> `{a = 1, b = 2}` |
| `concat(list1, list2, ...)` | Concatenates lists | `concat(["a"], ["b"])` -> `["a", "b"]` |
| `length(collection)` | Returns length of list/map/string | `length(["a", "b"])` -> `2` |
| `element(list, index)` | Returns element at index (wraps) | `element(["a", "b"], 0)` -> `"a"` |
| `contains(list, value)` | Checks if list contains value | `contains(["a", "b"], "a")` -> `true` |
| `keys(map)` | Returns sorted list of map keys | `keys({b = 2, a = 1})` -> `["a", "b"]` |
| `values(map)` | Returns list of map values | `values({a = 1, b = 2})` -> `[1, 2]` |
| `flatten(list)` | Flattens nested lists one level | `flatten([["a"], ["b"]])` -> `["a", "b"]` |
| `distinct(list)` | Removes duplicates from a list | `distinct(["a", "a", "b"])` -> `["a", "b"]` |
| `sort(list)` | Sorts list of strings lexically | `sort(["c", "a", "b"])` -> `["a", "b", "c"]` |

### Encoding Functions

| Function | Description | Example |
|----------|-------------|---------|
| `jsonencode(value)` | Encodes value as JSON string | `jsonencode({a = 1})` -> `"{\"a\":1}"` |
| `jsondecode(string)` | Decodes JSON string to value | `jsondecode("{\"a\":1}")` -> `{a = 1}` |
| `yamldecode(string)` | Decodes YAML string to value | `yamldecode(file("config.yaml"))` |

### Type Conversion Functions

| Function | Description | Example |
|----------|-------------|---------|
| `tobool(value)` | Converts to boolean | `tobool("true")` -> `true` |
| `tostring(value)` | Converts to string | `tostring(42)` -> `"42"` |
| `tonumber(value)` | Converts to number | `tonumber("42")` -> `42` |
| `tolist(value)` | Converts to list | `tolist(toset(["a", "b"]))` |
| `tomap(value)` | Converts to map | `tomap({a = "1", b = "2"})` |
| `toset(value)` | Converts to set | `toset(["a", "b", "a"])` -> `toset(["a", "b"])` |

### Filesystem Functions

| Function | Description | Example |
|----------|-------------|---------|
| `file(path)` | Reads file contents as string | `file("${get_terragrunt_dir()}/data.txt")` |
| `templatefile(path, vars)` | Renders a template file | `templatefile("tpl.tftpl", {name = "app"})` |
| `fileexists(path)` | Checks if file exists | `fileexists("${get_terragrunt_dir()}/override.hcl")` |
| `fileset(path, pattern)` | Finds files matching a glob | `fileset(".", "*.tf")` |
| `abspath(path)` | Converts to absolute path | `abspath(".")` |
| `dirname(path)` | Returns directory portion | `dirname("/a/b/c.txt")` -> `"/a/b"` |
| `basename(path)` | Returns filename portion | `basename("/a/b/c.txt")` -> `"c.txt"` |
| `pathexpand(path)` | Expands `~` to home directory | `pathexpand("~/.ssh/id_rsa")` |

### Date/Time Functions

| Function | Description | Example |
|----------|-------------|---------|
| `formatdate(spec, timestamp)` | Formats a timestamp | `formatdate("YYYY-MM-DD", timestamp())` |
| `timestamp()` | Returns current UTC timestamp | `timestamp()` -> `"2026-01-15T12:00:00Z"` |

### Error Handling Functions

| Function | Description | Example |
|----------|-------------|---------|
| `try(expr1, expr2, ...)` | Returns first non-error expression | `try(local.optional.value, "default")` |
| `can(expression)` | Returns true if expression succeeds | `can(local.optional.value)` -> `true` or `false` |

---

## Path Functions

### find_in_parent_folders

Searches UP the directory tree from the current `terragrunt.hcl` file and returns the absolute path to the first matching file.

**Signature:**

```
find_in_parent_folders()
find_in_parent_folders(name)
find_in_parent_folders(name, fallback)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | No | Filename to search for. Defaults to `terragrunt.hcl` or `root.hcl`. |
| `fallback` | string | No | Path to return if file is not found. Without this, a missing file causes an error. |

**Return value:** `string` -- absolute path to the found file, or the fallback value.

**Example 1: Find root configuration**

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}
```

**Example 2: Find optional config with fallback**

```hcl
locals {
  account_vars = read_terragrunt_config(
    find_in_parent_folders("account.hcl", "${get_terragrunt_dir()}/empty.hcl")
  )
}
```

---

### path_relative_to_include

Returns the relative path from the directory of the included (parent) config to the directory of the current (child) `terragrunt.hcl`.

**Signature:**

```
path_relative_to_include()
path_relative_to_include(include_name)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `include_name` | string | No | Name of a specific `include` block to compute the path relative to. Required when multiple `include` blocks exist and you need a specific one. |

**Return value:** `string` -- relative path from parent config directory to current config directory.

**Example 1: Generate unique state key per unit**

In `root.hcl`:

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-state-bucket"
    key    = "${path_relative_to_include()}/tofu.tfstate"
    # For dev/us-east-1/vpc/terragrunt.hcl -> "dev/us-east-1/vpc/tofu.tfstate"
  }
}
```

**Example 2: With named include**

```hcl
include "root" {
  path = find_in_parent_folders("root.hcl")
}

include "env" {
  path = find_in_parent_folders("env.hcl")
}

locals {
  path_from_root = path_relative_to_include("root")
  path_from_env  = path_relative_to_include("env")
}
```

---

### path_relative_from_include

Returns the relative path FROM the current config directory TO the directory of the included (parent) config. This is the inverse of `path_relative_to_include`.

**Signature:**

```
path_relative_from_include()
path_relative_from_include(include_name)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `include_name` | string | No | Name of a specific `include` block. |

**Return value:** `string` -- relative path from current config directory to parent config directory.

**Example 1: Construct module source path**

In `root.hcl`:

```hcl
terraform {
  source = "${path_relative_from_include()}/../modules//${path_relative_to_include()}"
}
```

If the child is at `dev/us-east-1/vpc/terragrunt.hcl` and root is at the repo root, this resolves to `../../../modules//dev/us-east-1/vpc`.

---

### get_terragrunt_dir

Returns the absolute path to the directory containing the current `terragrunt.hcl` file.

**Signature:**

```
get_terragrunt_dir()
```

**Parameters:** None.

**Return value:** `string` -- absolute path to the directory of the current `terragrunt.hcl`.

**Example 1: Reference a file relative to the unit**

```hcl
locals {
  vars = yamldecode(file("${get_terragrunt_dir()}/vars.yaml"))
}
```

**Example 2: Extra arguments with a local tfvars file**

```hcl
terraform {
  extra_arguments "custom_vars" {
    commands  = get_terraform_commands_that_need_vars()
    arguments = ["-var-file=${get_terragrunt_dir()}/extra.tfvars"]
  }
}
```

---

### get_working_dir

Returns the absolute path to the directory where Terragrunt will run OpenTofu/Terraform commands. This is typically the `.terragrunt-cache` directory where source is downloaded.

**Signature:**

```
get_working_dir()
```

**Parameters:** None.

**Return value:** `string` -- absolute path to the working directory.

**Example 1: Reference a file in the working directory**

```hcl
terraform {
  after_hook "copy_output" {
    commands = ["apply"]
    execute  = ["cp", "${get_working_dir()}/output.json", "${get_terragrunt_dir()}/"]
  }
}
```

---

### get_parent_terragrunt_dir

Returns the absolute path to the directory of the parent Terragrunt configuration (the config that is being included).

**Signature:**

```
get_parent_terragrunt_dir()
get_parent_terragrunt_dir(include_name)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `include_name` | string | No | Name of a specific `include` block. |

**Return value:** `string` -- absolute path to the parent config directory.

**Example 1: Reference a file relative to the parent config**

```hcl
locals {
  common_vars = yamldecode(file("${get_parent_terragrunt_dir()}/common_vars.yaml"))
}
```

---

### get_original_terragrunt_dir

Returns the absolute path to the directory of the original config being read. This is useful when using `read_terragrunt_config` to read another config file -- it returns the directory of the config being read, not the calling config.

**Signature:**

```
get_original_terragrunt_dir()
```

**Parameters:** None.

**Return value:** `string` -- absolute path to the directory of the config currently being parsed.

**Example 1: Reference sibling files from within a read config**

In a shared config file read via `read_terragrunt_config`:

```hcl
locals {
  data = file("${get_original_terragrunt_dir()}/data.json")
}
```

---

### get_repo_root

Returns the absolute path to the root of the Git repository containing the current `terragrunt.hcl`. Errors if the file is not inside a Git repository.

**Signature:**

```
get_repo_root()
```

**Parameters:** None.

**Return value:** `string` -- absolute path to the Git repository root.

**Example 1: Reference a module in the same repo**

```hcl
terraform {
  source = "${get_repo_root()}//modules/vpc"
}
```

**Example 2: Reference shared configuration**

```hcl
locals {
  global_vars = yamldecode(file("${get_repo_root()}/global.yaml"))
}
```

---

### get_path_from_repo_root

Returns the relative path from the Git repository root to the current directory.

**Signature:**

```
get_path_from_repo_root()
```

**Parameters:** None.

**Return value:** `string` -- relative path from repo root to current directory.

**Example 1: Use as a state key**

```hcl
remote_state {
  config = {
    key = "${get_path_from_repo_root()}/tofu.tfstate"
  }
}
```

---

### get_path_to_repo_root

Returns the relative path from the current directory to the Git repository root.

**Signature:**

```
get_path_to_repo_root()
```

**Parameters:** None.

**Return value:** `string` -- relative path from current directory to repo root.

**Example 1: Reference repo root relatively**

```hcl
terraform {
  source = "${get_path_to_repo_root()}/../infrastructure-modules//vpc"
}
```

---

## Environment and Platform Functions

### get_env

Returns the value of an environment variable, or a default value if the variable is not set.

**Signature:**

```
get_env(NAME)
get_env(NAME, DEFAULT)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `NAME` | string | Yes | Name of the environment variable. |
| `DEFAULT` | string | No | Value to return if the variable is not set. If omitted and the variable is unset, returns empty string. |

**Return value:** `string` -- the environment variable value or the default.

**Example 1: AWS profile with fallback**

```hcl
locals {
  aws_profile = get_env("AWS_PROFILE", "default")
}
```

**Example 2: Dynamic state bucket**

```hcl
remote_state {
  config = {
    bucket = get_env("TF_STATE_BUCKET", "my-company-tf-state")
    region = get_env("AWS_DEFAULT_REGION", "us-east-1")
  }
}
```

---

### get_platform

Returns the current operating system platform.

**Signature:**

```
get_platform()
```

**Parameters:** None.

**Return value:** `string` -- one of `"darwin"`, `"freebsd"`, `"linux"`, or `"windows"`.

**Example 1: Platform-specific script**

```hcl
terraform {
  before_hook "setup" {
    commands = ["apply"]
    execute  = [get_platform() == "windows" ? "setup.bat" : "./setup.sh"]
  }
}
```

---

## Terraform Command Functions

### get_terraform_commands_that_need_vars

Returns a list of OpenTofu/Terraform commands that accept `-var` and `-var-file` arguments.

**Signature:**

```
get_terraform_commands_that_need_vars()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of command names (e.g., `["apply", "destroy", "plan", ...]`).

**Example 1: Apply extra var files to all relevant commands**

```hcl
terraform {
  extra_arguments "common_vars" {
    commands           = get_terraform_commands_that_need_vars()
    required_var_files = ["${get_terragrunt_dir()}/common.tfvars"]
  }
}
```

---

### get_terraform_commands_that_need_input

Returns a list of OpenTofu/Terraform commands that accept the `-input` parameter.

**Signature:**

```
get_terraform_commands_that_need_input()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of command names.

**Example 1: Disable interactive input for all relevant commands**

```hcl
terraform {
  extra_arguments "disable_input" {
    commands  = get_terraform_commands_that_need_input()
    arguments = ["-input=false"]
  }
}
```

---

### get_terraform_commands_that_need_locking

Returns a list of OpenTofu/Terraform commands that accept the `-lock-timeout` parameter.

**Signature:**

```
get_terraform_commands_that_need_locking()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of command names.

**Example 1: Set lock timeout for all relevant commands**

```hcl
terraform {
  extra_arguments "lock_timeout" {
    commands  = get_terraform_commands_that_need_locking()
    arguments = ["-lock-timeout=20m"]
  }
}
```

---

### get_terraform_commands_that_need_parallelism

Returns a list of OpenTofu/Terraform commands that accept the `-parallelism` parameter.

**Signature:**

```
get_terraform_commands_that_need_parallelism()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of command names.

**Example 1: Limit parallelism for resource-constrained environments**

```hcl
terraform {
  extra_arguments "limit_parallelism" {
    commands  = get_terraform_commands_that_need_parallelism()
    arguments = ["-parallelism=5"]
  }
}
```

---

### get_terraform_command

Returns the name of the current OpenTofu/Terraform command being executed.

**Signature:**

```
get_terraform_command()
```

**Parameters:** None.

**Return value:** `string` -- the command name (e.g., `"plan"`, `"apply"`, `"destroy"`).

**Example 1: Conditional behavior based on command**

```hcl
terraform {
  before_hook "warn_destroy" {
    commands = ["destroy"]
    execute  = ["echo", "WARNING: Destroying resources in ${get_terragrunt_dir()}"]
  }
}
```

---

### get_terraform_cli_args

Returns the list of CLI arguments being passed to the current OpenTofu/Terraform command.

**Signature:**

```
get_terraform_cli_args()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of CLI argument strings.

**Example 1: Log CLI arguments**

```hcl
terraform {
  before_hook "log_args" {
    commands = ["plan", "apply"]
    execute  = ["echo", "Running with args: ${join(" ", get_terraform_cli_args())}"]
  }
}
```

---

## AWS Identity Functions

These functions call the AWS STS API to retrieve information about the current AWS identity. They require valid AWS credentials.

### get_aws_account_id

Returns the AWS account ID of the currently authenticated identity.

**Signature:**

```
get_aws_account_id()
```

**Parameters:** None.

**Return value:** `string` -- the 12-digit AWS account ID (e.g., `"123456789012"`).

**Important:** The returned value may change after `iam_role` evaluation, since assuming a role can switch the effective account.

**Example 1: Dynamic state bucket name**

```hcl
remote_state {
  config = {
    bucket = "tf-state-${get_aws_account_id()}"
  }
}
```

**Example 2: Account-specific inputs**

```hcl
inputs = {
  account_id   = get_aws_account_id()
  state_bucket = "company-${get_aws_account_id()}-terraform-state"
}
```

---

### get_aws_account_alias

Returns the AWS account alias for the current account, or an empty string if no alias is configured.

**Signature:**

```
get_aws_account_alias()
```

**Parameters:** None.

**Return value:** `string` -- the account alias or `""`.

**Example 1: Use alias in tagging**

```hcl
inputs = {
  account_alias = get_aws_account_alias()
}
```

---

### get_aws_caller_identity_arn

Returns the ARN of the currently authenticated AWS identity.

**Signature:**

```
get_aws_caller_identity_arn()
```

**Parameters:** None.

**Return value:** `string` -- the caller identity ARN (e.g., `"arn:aws:iam::123456789012:user/myuser"` or `"arn:aws:sts::123456789012:assumed-role/myrole/session"`).

**Example 1: Log who is running Terragrunt**

```hcl
locals {
  caller_arn = get_aws_caller_identity_arn()
}
```

---

### get_aws_caller_identity_user_id

Returns the unique user ID of the currently authenticated AWS identity.

**Signature:**

```
get_aws_caller_identity_user_id()
```

**Parameters:** None.

**Return value:** `string` -- the UserId (e.g., `"AIDAJDPLRKLG7EXAMPLE"` for IAM users, or `"AROA3XFRBF23:session-name"` for assumed roles).

**Example 1: Use as an identifier**

```hcl
locals {
  user_id = get_aws_caller_identity_user_id()
}
```

---

## Execution Functions

### run_cmd

Executes a shell command and returns its stdout as a string. The command runs in the same directory as the `terragrunt.hcl` file containing the call.

**Signature:**

```
run_cmd(command, arg1, arg2, ...)
run_cmd(flag1, flag2, ..., command, arg1, arg2, ...)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command` | string | Yes | The command to execute. |
| `arg1, arg2, ...` | string | No | Arguments passed to the command. |

**Flags** (must appear BEFORE the command):

| Flag | Description |
|------|-------------|
| `--terragrunt-quiet` | Redacts command output from Terragrunt logs. Use for secrets. |
| `--terragrunt-global-cache` | Caches result globally (not per-directory). Use for commands whose output is directory-independent. |
| `--terragrunt-no-cache` | Disables caching entirely. Command runs fresh every time. |

**Caching behavior:** By default, results are cached per directory + command + arguments combination within a single Terragrunt run. The same `run_cmd` call in the same directory returns the cached result.

**Return value:** `string` -- the stdout of the command, with trailing newline stripped.

**Example 1: Get a value from a script**

```hcl
locals {
  bucket_name = run_cmd("./get_names.sh", "bucket")
}
```

**Example 2: Fetch a secret (redacted from logs)**

```hcl
locals {
  db_password = run_cmd("--terragrunt-quiet", "aws", "secretsmanager",
    "get-secret-value", "--secret-id", "db-password",
    "--query", "SecretString", "--output", "text")
}
```

**Example 3: Globally cached command**

```hcl
locals {
  account_id = run_cmd("--terragrunt-global-cache",
    "aws", "sts", "get-caller-identity",
    "--query", "Account", "--output", "text")
}
```

**Example 4: Uncached command (fresh every time)**

```hcl
locals {
  build_id = run_cmd("--terragrunt-no-cache", "uuidgen")
}
```

---

## Configuration Reading Functions

### read_terragrunt_config

Parses another Terragrunt configuration file and returns its contents as a map. This function fully renders the target config, including resolving dependency outputs.

**Signature:**

```
read_terragrunt_config(config_path)
read_terragrunt_config(config_path, default_val)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `config_path` | string | Yes | Absolute or relative path to the config file to read. Can be `terragrunt.hcl`, `terragrunt.stack.hcl`, or `terragrunt.values.hcl`. |
| `default_val` | any | No | Value to return if the config file cannot be read or parsed. Without this, errors are fatal. |

**Return value:** `map` -- a map containing all blocks and attributes from the parsed config. Access locals via `.locals`, inputs via `.inputs`, dependencies via `.dependency.<name>.outputs.<key>`.

**Example 1: Read shared variables from a parent config**

```hcl
locals {
  common     = read_terragrunt_config(find_in_parent_folders("common.hcl"))
  account    = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  region     = read_terragrunt_config(find_in_parent_folders("region.hcl"))
  env_name   = local.account.locals.env_name
  aws_region = local.region.locals.aws_region
}

inputs = merge(
  local.common.inputs,
  {
    env    = local.env_name
    region = local.aws_region
  }
)
```

**Example 2: Read optional config with default**

```hcl
locals {
  overrides = read_terragrunt_config(
    find_in_parent_folders("overrides.hcl", "${get_terragrunt_dir()}/empty.hcl"),
    { inputs = {} }
  )
}

inputs = merge(
  { instance_type = "t3.micro" },
  local.overrides.inputs
)
```

---

### read_tfvars_file

Reads a `.tfvars` or `.tfvars.json` file and returns its contents as a map of variable values.

**Signature:**

```
read_tfvars_file(file_path)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_path` | string | Yes | Path to the `.tfvars` or `.tfvars.json` file. |

**Return value:** `map` -- a map of variable names to their values.

**Example 1: Read and merge tfvars**

```hcl
locals {
  env_vars = read_tfvars_file("${get_terragrunt_dir()}/env.tfvars")
}

inputs = merge(local.env_vars, {
  extra_tag = "managed-by-terragrunt"
})
```

**Example 2: Read JSON tfvars**

```hcl
locals {
  config = read_tfvars_file("${get_terragrunt_dir()}/config.tfvars.json")
}
```

---

## Encryption and Secrets

### sops_decrypt_file

Decrypts a file encrypted with [Mozilla SOPS](https://github.com/getsops/sops) and returns the decrypted content as a string. Supports YAML, JSON, ENV, INI, and raw binary formats.

**Signature:**

```
sops_decrypt_file(file_path)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_path` | string | Yes | Path to the SOPS-encrypted file. |

**Supported encryption backends:** AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault Transit, age, PGP.

**Return value:** `string` -- the decrypted file contents.

**Example 1: Decrypt YAML secrets**

```hcl
locals {
  secrets = yamldecode(sops_decrypt_file(find_in_parent_folders("secrets.yaml")))
}

inputs = {
  db_password   = local.secrets.db_password
  api_key       = local.secrets.api_key
}
```

**Example 2: Decrypt JSON secrets with fallback**

```hcl
locals {
  secrets = try(jsondecode(sops_decrypt_file("${get_terragrunt_dir()}/secrets.json")), {})
}
```

---

## Data Processing Functions

### deep_merge

Deeply merges two or more maps. Unlike the built-in `merge()` which performs a shallow merge (overwriting nested maps entirely), `deep_merge()` recursively merges nested maps. Later arguments override earlier ones for conflicting keys. Lists are appended. Null values are ignored.

**Signature:**

```
deep_merge(map1, map2, ...)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `map1, map2, ...` | map | Yes (at least 2) | Maps to merge recursively. |

**Requires:** The `deep-merge` experiment must be enabled:

```hcl
experiments = ["deep-merge"]
```

**Return value:** `map` -- the deeply merged result.

**Example 1: Merge default and environment-specific config**

```hcl
experiments = ["deep-merge"]

locals {
  defaults = {
    service = {
      retries     = 1
      mode        = "safe"
      healthcheck = "/health"
    }
    tags = {
      managed_by = "terragrunt"
    }
  }

  env_config = {
    service = {
      retries = 3
    }
    tags = {
      env = "production"
    }
  }

  final = deep_merge(local.defaults, local.env_config)
  # Result:
  # {
  #   service = {
  #     retries     = 3
  #     mode        = "safe"
  #     healthcheck = "/health"
  #   }
  #   tags = {
  #     managed_by = "terragrunt"
  #     env        = "production"
  #   }
  # }
}
```

---

## Special Functions

### get_terragrunt_source_cli_flag

Returns the value passed via the `--source` CLI flag or the `TG_SOURCE` environment variable. Returns an empty string if neither is set.

**Signature:**

```
get_terragrunt_source_cli_flag()
```

**Parameters:** None.

**Return value:** `string` -- the source override value, or `""`.

**Example 1: Local development override**

```hcl
terraform {
  source = get_terragrunt_source_cli_flag() != "" ? get_terragrunt_source_cli_flag() : "git::git@github.com:acme/modules.git//vpc?ref=v1.0.0"
}
```

---

### get_default_retryable_errors

Returns the default list of retryable error patterns (as regex strings) that Terragrunt recognizes. Use this to seed your retry configuration while adding custom patterns.

**Signature:**

```
get_default_retryable_errors()
```

**Parameters:** None.

**Return value:** `list(string)` -- list of regex patterns matching transient errors.

**Example 1: Extend default retryable errors**

```hcl
errors {
  retry {
    retryable_errors = concat(
      get_default_retryable_errors(),
      ["(?s).*Error: Provider produced inconsistent result.*"]
    )
    max_attempts = 3
    sleep_interval_sec = 30
  }
}
```

---

### mark_as_read

Marks a file as "read" for the `--queue-include-units-reading` CLI flag. This allows Terragrunt to determine which units read which files, enabling selective execution when files change. Must be called in a `locals` block with an absolute file path.

**Signature:**

```
mark_as_read(file_path)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `file_path` | string | Yes | Absolute path to the file to mark as read. |

**Return value:** `string` -- the same file path passed in.

**Example 1: Track a dependency file**

```hcl
locals {
  config_file = mark_as_read("${get_repo_root()}/config/shared.yaml")
  config      = yamldecode(file(local.config_file))
}
```

---

### mark_glob_as_read

Marks multiple files matching a glob pattern as "read" for the `--queue-include-units-reading` CLI flag. Requires the `mark-many-as-read` experiment to be enabled.

**Signature:**

```
mark_glob_as_read(pattern)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `pattern` | string | Yes | Glob pattern using forward slashes. Supports `*`, `**`, `?`, `[abc]`, `{a,b}`. |

**Requires:** The `mark-many-as-read` experiment must be enabled.

**Return value:** `string` -- the pattern passed in.

**Example 1: Track all YAML files in a config directory**

```hcl
experiments = ["mark-many-as-read"]

locals {
  _ = mark_glob_as_read("${get_repo_root()}/config/**/*.yaml")
}
```

---

### constraint_check

Checks whether a version string satisfies a version constraint. Uses semantic versioning comparison.

**Signature:**

```
constraint_check(version, constraint)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `version` | string | Yes | The version string to check (e.g., `"1.5.3"`). |
| `constraint` | string | Yes | The version constraint (e.g., `">= 2.0.0"`, `"~> 1.5"`, `"< 3.0.0"`). |

**Return value:** `bool` -- `true` if the version satisfies the constraint, `false` otherwise.

**Example 1: Feature gating based on module version**

```hcl
locals {
  needs_v2 = constraint_check(feature.module_version.value, ">= 2.0.0")
}

inputs = {
  enable_new_feature = local.needs_v2
}
```

**Example 2: Conditional configuration**

```hcl
locals {
  tg_version   = "0.78.0"
  has_deep_merge = constraint_check(local.tg_version, ">= 0.67.0")
}
```
