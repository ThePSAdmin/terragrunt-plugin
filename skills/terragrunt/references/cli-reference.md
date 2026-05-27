# Terragrunt CLI Reference

Comprehensive reference for all Terragrunt commands, flags, and environment variables. Covers current implementation only.

## Table of Contents

- [Commands](#commands)
  - [terragrunt run](#terragrunt-run)
  - [terragrunt exec](#terragrunt-exec)
  - [terragrunt stack generate](#terragrunt-stack-generate)
  - [terragrunt stack output](#terragrunt-stack-output)
  - [terragrunt stack run](#terragrunt-stack-run)
  - [terragrunt stack clean](#terragrunt-stack-clean)
  - [terragrunt backend bootstrap](#terragrunt-backend-bootstrap)
  - [terragrunt backend delete](#terragrunt-backend-delete)
  - [terragrunt backend migrate](#terragrunt-backend-migrate)
  - [terragrunt catalog](#terragrunt-catalog)
  - [terragrunt scaffold](#terragrunt-scaffold)
  - [terragrunt find](#terragrunt-find)
  - [terragrunt list](#terragrunt-list)
  - [terragrunt dag graph](#terragrunt-dag-graph)
  - [terragrunt hcl fmt](#terragrunt-hcl-fmt)
  - [terragrunt hcl validate](#terragrunt-hcl-validate)
  - [terragrunt info print](#terragrunt-info-print)
  - [terragrunt info strict](#terragrunt-info-strict)
  - [terragrunt render](#terragrunt-render)
- [Global Flags](#global-flags)
- [Key Environment Variables](#key-environment-variables)
- [Common Usage Examples](#common-usage-examples)

---

## Commands

### terragrunt run

The primary command. Runs OpenTofu/Terraform commands on units.

```bash
terragrunt run [flags] [--] <tofu-command> [tofu-args]
terragrunt run --all [flags] [--] <tofu-command> [tofu-args]
terragrunt run --graph [flags] [--] <tofu-command> [tofu-args]
```

**Modes:**

- Without `--all` or `--graph`: runs on the current unit (equivalent to `terragrunt plan`, `terragrunt apply`, etc.)
- `--all`: runs on all units in the directory tree, respecting DAG order
- `--graph`: runs on all units in the DAG of the current unit

**Flags:**

| Flag | Description |
|------|-------------|
| `--all` | Run on all units in the directory tree, respecting DAG order |
| `--graph` | Run on all units in the DAG of the current unit |
| `--filter <expr>` | Filter units (glob, name, path, git, graph expressions). Repeatable |
| `--filters-file <path>` | Load filters from a file |
| `--filter-affected` | Only units affected by git changes |
| `--filter-allow-destroy` | Allow destroy on filtered units |
| `--parallelism <n>` | Max concurrent runs (default varies) |
| `--queue-construct-as <cmd>` | Build queue as if running a different command |
| `--queue-ignore-dag-order` | Run concurrently ignoring dependencies (dangerous for apply/destroy) |
| `--queue-ignore-errors` | Continue on failures |
| `--queue-include-units-reading <file>` | Include units that read a specific file |
| `--queue-exclude-dir <path>` | Exclude directory from the queue |
| `--queue-include-dir <path>` | Include only this directory in the queue |
| `--fail-fast` | Stop on first error |
| `--no-auto-approve` | Don't auto-approve applies in `run --all` |
| `--no-auto-init` | Skip automatic `init` |
| `--no-auto-retry` | Skip automatic retry on transient errors |
| `--source <path>` | Override terraform source |
| `--source-update` | Force re-download of remote sources |
| `--source-map <old>=<new>` | Map old source to new source |
| `--out-dir <path>` | Directory for binary plan files |
| `--json-out-dir <path>` | Directory for JSON plan output |
| `--inputs-debug` | Generate `terragrunt-debug.tfvars.json` |
| `--tf-path <path>` | Path to tofu/terraform binary |
| `--tf-forward-stdout` | Forward terraform stdout directly |
| `--config <path>` | Path to `terragrunt.hcl` |
| `--download-dir <path>` | Custom download directory |
| `--dependency-fetch-output-from-state` | Fetch dependency outputs from state |
| `--provider-cache` | Enable provider caching |
| `--provider-cache-dir <path>` | Custom provider cache directory |
| `--provider-cache-hostname <host>` | Cache server hostname (default: `localhost`) |
| `--provider-cache-port <port>` | Cache server port |
| `--provider-cache-token <token>` | Cache auth token |
| `--provider-cache-registry-names <names>` | Registries to cache |
| `--report-file <path>` | Write run report (csv or json) |
| `--report-format <fmt>` | Report format: `csv` or `json` |
| `--report-schema-file <path>` | Generate JSON schema for report |
| `--summary-per-unit` | Show per-unit durations |
| `--summary-disable` | Disable run summary |
| `--destroy-dependencies-check` | Check for destroy dependencies |
| `--iam-assume-role <arn>` | IAM role to assume |
| `--iam-assume-role-duration <seconds>` | STS session duration |
| `--iam-assume-role-session-name <name>` | Session name |
| `--iam-assume-role-web-identity-token <token>` | Web identity token |
| `--feature <name>=<value>` | Set feature flag override |

---

### terragrunt exec

Execute arbitrary commands wrapped by Terragrunt. The command receives all unit inputs as `TF_VAR_*` environment variables.

```bash
terragrunt exec [flags] -- <command> [args]
```

**Example:**

```bash
terragrunt exec -- bash -c 'aws s3 ls s3://$TF_VAR_bucket_name'
```

---

### terragrunt stack generate

Generate units from `terragrunt.stack.hcl` files.

```bash
terragrunt stack generate [flags]
```

---

### terragrunt stack output

Get aggregated outputs from all units in a stack.

```bash
terragrunt stack output [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--format json` | JSON output |
| `--parallelism <n>` | Limit concurrent output fetches |

---

### terragrunt stack run

Run a command against a stack.

```bash
terragrunt stack run [flags] -- <command>
```

---

### terragrunt stack clean

Remove `.terragrunt-stack` directories.

```bash
terragrunt stack clean [flags]
```

---

### terragrunt backend bootstrap

Create backend infrastructure (S3 bucket, DynamoDB table, etc.).

```bash
terragrunt backend bootstrap [flags]
```

---

### terragrunt backend delete

Delete backend state for a unit.

```bash
terragrunt backend delete [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--force` | Skip confirmation prompt |

---

### terragrunt backend migrate

Migrate state from one unit to another.

```bash
terragrunt backend migrate [flags]
```

---

### terragrunt catalog

Launch the TUI to browse and scaffold modules.

```bash
terragrunt catalog [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--no-shell` | Disable shell commands |
| `--no-hooks` | Disable hooks |

---

### terragrunt scaffold

Generate a `terragrunt.hcl` from a module.

```bash
terragrunt scaffold <MODULE_URL> [TEMPLATE_URL] [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--var key=value` | Pass variables |
| `--var-file <path>` | Variables from file |
| `--no-include-root` | Don't include root config |
| `--root-file-name <name>` | Name of root config to include |
| `--no-dependency-prompt` | Don't prompt for dependencies |
| `--no-shell` | Disable shell execution |
| `--no-hooks` | Disable hooks |

**Example:**

```bash
terragrunt scaffold github.com/acme/modules//networking/vpc
```

---

### terragrunt find

Find units and stacks with filtering.

```bash
terragrunt find [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--filter <expr>` | Filter expression |
| `--json` | JSON output |
| `--dag` | Show DAG info |

---

### terragrunt list

List units with status info.

```bash
terragrunt list [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--filter <expr>` | Filter expression |
| `--queue-construct-as <cmd>` | Show order for specific command (alias: `--as`) |
| `-l` | Long format with type info |
| `--tree` | Tree view |
| `--json` | JSON output |
| `--format dot` | DOT format for graphviz |
| `--dependencies` | Show dependency info |
| `--external` | Include external dependencies |

---

### terragrunt dag graph

Output the DAG in DOT language.

```bash
terragrunt dag graph [flags]
```

**Example (pipe to graphviz):**

```bash
terragrunt dag graph | dot -Tsvg > graph.svg
```

---

### terragrunt hcl fmt

Format HCL files.

```bash
terragrunt hcl fmt [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--check` | Verify formatting without rewriting |
| `--diff` | Show unified diff |
| `--file <path>` | Format a single file |
| `--exclude-dir <path>` | Exclude directory |

---

### terragrunt hcl validate

Validate HCL files.

```bash
terragrunt hcl validate [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--json` | JSON output |
| `--show-config-path` | Show config path |

---

### terragrunt info print

Print Terragrunt context info.

```bash
terragrunt info print [flags]
```

---

### terragrunt info strict

List and inspect strict controls.

```bash
terragrunt info strict [flags]
```

---

### terragrunt render

Render merged Terragrunt configuration.

```bash
terragrunt render [flags]
```

**Flags:**

| Flag | Description |
|------|-------------|
| `--json` | JSON output |
| `-w` | Write to file |

---

## Global Flags

These flags can be used with any command.

| Flag | Env Var | Description |
|------|---------|-------------|
| `--working-dir <path>` | `TG_WORKING_DIR` | Run from a different directory |
| `--config <path>` | `TG_CONFIG` | Path to config file |
| `--log-level <level>` | `TG_LOG_LEVEL` | Log level: `debug`, `info`, `warn`, `error` |
| `--log-format <fmt>` | `TG_LOG_FORMAT` | Log output format |
| `--log-custom-format <fmt>` | `TG_LOG_CUSTOM_FORMAT` | Custom log format string |
| `--log-disable` | `TG_LOG_DISABLE` | Disable logging |
| `--log-show-abs-paths` | `TG_LOG_SHOW_ABS_PATHS` | Show absolute paths in logs |
| `--no-color` | `TG_NO_COLOR` | Disable color output |
| `--non-interactive` | `TG_NON_INTERACTIVE` | Assume yes to all prompts |
| `--strict-mode` | `TG_STRICT_MODE` | Enable all strict controls |
| `--strict-control <name>` | `TG_STRICT_CONTROL` | Enable a specific strict control |
| `--experiment <name>` | `TG_EXPERIMENT` | Enable an experiment |
| `--experiment-mode` | `TG_EXPERIMENT_MODE` | Enable experiment mode |
| `--no-tip <name>` | `TG_NO_TIP` | Disable a specific tip |
| `--no-tips` | `TG_NO_TIPS` | Disable all tips |
| `--help` | | Show help |
| `--version` | | Show version |

---

## Key Environment Variables

| Variable | Description |
|----------|-------------|
| `TG_IAM_ASSUME_ROLE` | IAM role ARN to assume |
| `TG_IAM_ASSUME_ROLE_DURATION` | STS session duration (seconds) |
| `TG_IAM_ASSUME_ROLE_SESSION_NAME` | STS session name |
| `TG_IAM_ASSUME_ROLE_WEB_IDENTITY_TOKEN` | Web identity token |
| `TG_TF_PATH` | Path to tofu/terraform binary |
| `TG_SOURCE` | Override terraform source |
| `TG_DOWNLOAD_DIR` | Custom download directory |
| `TG_NO_AUTO_INIT` | Disable auto-init |
| `TG_NO_AUTO_RETRY` | Disable auto-retry |
| `TG_PARALLELISM` | Max concurrent runs |
| `TG_FEATURE` | Feature flag overrides (`name=value`) |
| `TG_INPUTS_DEBUG` | Generate debug tfvars |
| `TG_PROVIDER_CACHE` | Enable provider caching |
| `TG_PROVIDER_CACHE_DIR` | Provider cache directory |
| `TG_TF_REGISTRY_TOKEN` | Private registry auth token |
| `TG_TF_DEFAULT_REGISTRY_HOST` | Custom default registry |
| `TG_AUTH_PROVIDER_CMD` | Auth provider command path |
| `TG_ENGINE_CACHE_PATH` | Engine cache path |
| `TG_ENGINE_LOG_LEVEL` | Engine log level |
| `TG_ENGINE_SKIP_CHECK` | Skip engine checksum validation |

---

## Common Usage Examples

```bash
# Plan all units
terragrunt run --all plan

# Apply with parallelism limit
terragrunt run --all --parallelism 4 -- apply

# Plan only prod units
terragrunt run --all --filter './prod/**' -- plan

# Plan affected units (git changes)
terragrunt run --all --filter '[main...HEAD]' -- plan

# Destroy with filter
terragrunt run --all --filter 'vpc' --filter-allow-destroy -- destroy

# Generate and deploy stack
terragrunt stack generate && terragrunt stack run -- apply

# Show dependency graph
terragrunt dag graph | dot -Tsvg > graph.svg

# List units in destroy order
terragrunt list --as destroy -l

# Debug configuration
terragrunt render --json

# Format all HCL files
terragrunt hcl fmt

# Scaffold new unit from module
terragrunt scaffold github.com/acme/modules//networking/vpc

# Run with report
terragrunt run --all --report-file report.json -- plan

# Override source for local dev
terragrunt run --all --source /local/modules -- plan
```
