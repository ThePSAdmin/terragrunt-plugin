# Debugging and Observability

Comprehensive reference for Terragrunt debugging, logging, tracing, error handling, and performance diagnostics.

## Table of Contents

- [Log Levels](#log-levels)
- [Log Formatting](#log-formatting)
  - [Format Options](#format-options)
  - [Placeholders](#placeholders)
  - [Presets](#presets)
  - [Show Absolute Paths](#show-absolute-paths)
  - [Disable Logging](#disable-logging)
- [Strict Controls](#strict-controls)
  - [Enable All](#enable-all)
  - [Enable Specific Control](#enable-specific-control)
  - [List Available Controls](#list-available-controls)
  - [Key Controls](#key-controls)
- [OpenTelemetry Integration](#opentelemetry-integration)
  - [High-Level Overview](#high-level-overview)
  - [Example: Jaeger](#example-jaeger)
  - [Example: Grafana Tempo](#example-grafana-tempo)
  - [Console Output](#console-output)
  - [Metrics with Prometheus](#metrics-with-prometheus)
- [Error Handling](#error-handling)
  - [errors Block](#errors-block)
  - [retry Sub-Block](#retry-sub-block)
  - [ignore Sub-Block](#ignore-sub-block)
  - [Default Retryable Errors](#default-retryable-errors)
  - [Combining Default and Custom Errors](#combining-default-and-custom-errors)
  - [Dynamic Error Handling with Feature Flags](#dynamic-error-handling-with-feature-flags)
- [Feature Flags](#feature-flags)
- [Debugging Commands](#debugging-commands)
  - [terragrunt info print](#terragrunt-info-print)
  - [terragrunt render](#terragrunt-render)
  - [inputs-debug](#inputs-debug)
  - [Debugging OpenTofu/Terraform Behavior](#debugging-opentofuterraform-behavior)
- [Run Summary](#run-summary)
  - [Per-Unit Durations](#per-unit-durations)
  - [Disable Summary](#disable-summary)
- [Run Report](#run-report)
  - [Report Fields](#report-fields)
  - [CSV Example](#csv-example)
- [Performance Tips](#performance-tips)

---

## Log Levels

```bash
# Set log level
terragrunt run --log-level debug -- plan

# Environment variable
export TG_LOG_LEVEL=debug
```

Available levels: `debug`, `info`, `warn`, `error`

- `debug`: most verbose, shows all internal operations
- `info`: default, shows key operations
- `warn`: only warnings and errors
- `error`: only errors

## Log Formatting

### Format Options

```bash
# Set log format
terragrunt run --log-format json -- plan

# Custom format
terragrunt run --log-custom-format '%(time) [%(level)] %(msg)' -- plan
```

### Placeholders

- `%(time)`: timestamp
- `%(level)`: log level
- `%(msg)`: log message
- `%(prefix)`: unit prefix
- `%(tf-path)`: path to tofu/terraform binary
- `%(working-dir)`: working directory
- `%(tf-command)`: terraform command
- `%(tf-command-args)`: terraform command arguments

### Presets

Format presets available via `--log-format`:
- Default format
- JSON format
- Custom via `--log-custom-format`

### Show Absolute Paths

```bash
terragrunt run --log-show-abs-paths -- plan
```

### Disable Logging

```bash
terragrunt run --log-disable -- plan
export TG_LOG_DISABLE=true
```

## Strict Controls

Strict controls enforce best practices and prepare for breaking changes.

### Enable All

```bash
terragrunt run --strict-mode -- plan
export TG_STRICT_MODE=true
```

### Enable Specific Control

```bash
terragrunt run --strict-control root-terragrunt-hcl -- plan
export TG_STRICT_CONTROL=root-terragrunt-hcl
```

### List Available Controls

```bash
terragrunt info strict
```

### Key Controls

- `root-terragrunt-hcl`: use `root.hcl` instead of `terragrunt.hcl` at repo root

## OpenTelemetry Integration

Terragrunt supports OpenTelemetry for distributed tracing and metrics collection.

### High-Level Overview

- Traces: each Terragrunt run creates spans for major operations
- Metrics: token usage, duration, command execution counts
- Supports any OpenTelemetry-compatible collector

### Example: Jaeger

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="http://localhost:4317"
export OTEL_SERVICE_NAME="terragrunt"
terragrunt run --all plan
```

### Example: Grafana Tempo

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://tempo.example.com:4317"
export OTEL_SERVICE_NAME="terragrunt"
terragrunt run --all plan
```

### Console Output

```bash
export OTEL_TRACES_EXPORTER="console"
terragrunt run plan
```

### Metrics with Prometheus

Configure OpenTelemetry collector to scrape Terragrunt metrics and forward to Prometheus.

## Error Handling

### errors Block

```hcl
errors {
  retry "transient_errors" {
    retryable_errors   = [".*Error: transient network issue.*"]
    max_attempts       = 3
    sleep_interval_sec = 5
  }

  ignore "known_safe_errors" {
    ignorable_errors = [
      ".*Error: safe warning.*",
      "!.*Error: do not ignore.*"
    ]
    message = "Ignoring safe warning errors"
    signals = {
      alert_team = false
    }
  }
}
```

### retry Sub-Block

- `retryable_errors` (required): list of regex patterns matching retriable error output
- `max_attempts` (required): maximum number of retries
- `sleep_interval_sec` (required): delay between retries in seconds

### ignore Sub-Block

- `ignorable_errors` (required): list of regex patterns; prefix with `!` to exclude from ignoring
- `message` (optional): string logged when error is ignored
- `signals` (optional): map of key-value pairs for downstream processing

### Default Retryable Errors

```hcl
errors {
  retry "default_errors" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }
}
```

`get_default_retryable_errors()` returns Terragrunt's built-in list of known transient errors (network timeouts, rate limits, etc.).

### Combining Default and Custom Errors

```hcl
errors {
  retry "default_errors" {
    retryable_errors   = get_default_retryable_errors()
    max_attempts       = 3
    sleep_interval_sec = 5
  }

  retry "custom_errors" {
    retryable_errors   = [".*my custom transient error.*"]
    max_attempts       = 5
    sleep_interval_sec = 10
  }
}
```

### Dynamic Error Handling with Feature Flags

```hcl
feature "enable_flaky_module" {
  default = false
}

errors {
  ignore "flaky_module_errors" {
    ignorable_errors = feature.enable_flaky_module.value ? [
      ".*Error: flaky module error.*"
    ] : []
    message = "Ignoring flaky module error"
    signals = {
      send_notification = true
    }
  }
}
```

## Feature Flags

```hcl
feature "s3_version" {
  default = "v1.0.0"
}

terraform {
  source = "git::git@github.com:acme/modules.git//s3?ref=${feature.s3_version.value}"
}
```

Override at runtime:

```bash
# CLI flag
terragrunt run --feature s3_version=v1.1.0 -- plan

# Environment variable
export TG_FEATURE="s3_version=v1.1.0"
terragrunt run plan
```

## Debugging Commands

### terragrunt info print

Shows resolved Terragrunt context including all evaluated locals, inputs, and configuration:

```bash
terragrunt info print
```

### terragrunt render

Outputs the fully merged configuration (all includes resolved):

```bash
# Human-readable
terragrunt render

# JSON format
terragrunt render --json

# Write to file
terragrunt render --json -w
```

### inputs-debug

Generate a debug tfvars file showing resolved input values:

```bash
terragrunt run --inputs-debug -- plan
# Creates terragrunt-debug.tfvars.json in working directory
```

### Debugging OpenTofu/Terraform Behavior

```bash
# Enable OpenTofu/Terraform debug logging
export TF_LOG=DEBUG
terragrunt run plan

# Trace level
export TF_LOG=TRACE
terragrunt run plan
```

## Run Summary

Displayed at the end of `run --all` commands:

```
Run Summary  3 units  62ms
────────────────────────────
Succeeded    3
```

### Per-Unit Durations

```bash
terragrunt run --all plan --summary-per-unit
```

Shows each unit sorted by duration (longest first).

### Disable Summary

```bash
terragrunt run --all plan --summary-disable
```

## Run Report

Generate detailed reports of multi-unit runs:

```bash
# CSV report
terragrunt run --all plan --report-file report.csv

# JSON report
terragrunt run --all plan --report-file report.json --report-format json

# Generate JSON schema
terragrunt run --all plan --report-schema-file report.schema.json
```

### Report Fields

- `Name`: unit name
- `Started`: ISO timestamp with timezone
- `Ended`: ISO timestamp with timezone
- `Result`: `excluded`, `failed`, `succeeded`, or `early exit`
- `Reason`: why the result occurred
- `Cause`: underlying cause

### CSV Example

```csv
Name,Started,Ended,Result,Reason,Cause
vpc,2025-06-05T16:28:41-04:00,2025-06-05T16:28:41-04:00,succeeded,,
app,2025-06-05T16:28:41-04:00,2025-06-05T16:28:42-04:00,failed,run error,
dependent,2025-06-05T16:28:42-04:00,2025-06-05T16:28:42-04:00,early exit,run error,
```

## Performance Tips

1. **Use Provider Cache Server** instead of `TF_PLUGIN_CACHE_DIR` with `run --all`
2. **Enable CAS** (`--experiment cas`) for stacks with remote sources
3. **Limit parallelism** if hitting API rate limits: `--parallelism 4`
4. **Use `--queue-ignore-dag-order`** for stateless commands like `validate` or `plan` (faster but unsafe for apply)
5. **Use filtering** to run only affected units: `--filter '[main...HEAD]'`
6. **Auto Provider Cache Dir** (OpenTofu >= 1.10) -- Terragrunt auto-configures shared cache
7. **Source caching**: Terragrunt caches downloaded sources; force refresh with `--source-update`
