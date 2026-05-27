# Terragrunt Remote State Reference

## Table of Contents

- [remote\_state Block](#remote_state-block)
  - [Attributes](#attributes)
- [S3 Backend (AWS)](#s3-backend-aws)
  - [Full Configuration](#full-configuration)
  - [All S3 Config Keys](#all-s3-config-keys)
  - [S3 Native Locking (No DynamoDB)](#s3-native-locking-no-dynamodb)
  - [Dual Locking (Migration Period)](#dual-locking-migration-period)
  - [Windows Path Compatibility](#windows-path-compatibility)
- [GCS Backend (Google Cloud)](#gcs-backend-google-cloud)
  - [All GCS Config Keys](#all-gcs-config-keys)
  - [GCS Example with Labels](#gcs-example-with-labels)
- [Azure Backend (Experimental)](#azure-backend-experimental)
- [Automatic Resource Creation](#automatic-resource-creation)
  - [Disabling Auto-Creation](#disabling-auto-creation)
  - [Skipping Backend Init Entirely](#skipping-backend-init-entirely)
- [DRY Remote State Pattern](#dry-remote-state-pattern)
  - [Dynamic Remote State from Config](#dynamic-remote-state-from-config)
- [OpenTofu State Encryption](#opentofu-state-encryption)
- [State Migration](#state-migration)

---

## remote_state Block

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
```

### Attributes

- `backend` (required): string -- backend type ("s3", "gcs", "azurerm", "local", etc.)
- `generate` (optional): map with `path` (filename to generate) and `if_exists` ("overwrite", "overwrite_terragrunt", "skip", "error")
- `config` (required): map -- backend-specific configuration
- `disable_init` (optional): boolean -- skip automatic resource creation
- `encryption` (optional): map -- OpenTofu state encryption

---

## S3 Backend (AWS)

### Full Configuration

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
```

### All S3 Config Keys

- `bucket`: S3 bucket name
- `key`: state file path in bucket
- `region`: AWS region
- `encrypt`: boolean -- enable SSE
- `dynamodb_table`: DynamoDB table for locking
- `profile`: AWS profile
- `role_arn`: IAM role to assume
- `endpoint`, `endpoints`: custom endpoints
- `shared_credentials_file`: path to credentials
- `external_id`, `session_name`: for role assumption
- `use_lockfile`: boolean -- S3 native locking (OpenTofu >= 1.10, no DynamoDB needed)
- `skip_bucket_versioning`: boolean
- `skip_bucket_ssencryption`: boolean
- `skip_bucket_root_access`: boolean
- `skip_bucket_enforced_tls`: boolean
- `skip_bucket_public_access_blocking`: boolean
- `skip_credentials_validation`: boolean
- `skip_metadata_api_check`: boolean
- `force_path_style`: boolean
- `enable_lock_table_ssencryption`: boolean
- `s3_bucket_tags`: map of tags
- `dynamodb_table_tags`: map of tags
- `accesslogging_bucket_name`: string
- `accesslogging_bucket_tags`: map
- `accesslogging_target_prefix`: string
- `bucket_sse_algorithm`: string
- `bucket_sse_kms_key_id`: string
- `assume_role`: map (roleArn, sessionName, etc.)
- `assume_role_with_web_identity`: map

### S3 Native Locking (No DynamoDB)

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket       = "my-tofu-state"
    key          = "${path_relative_to_include()}/tofu.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

Requires OpenTofu >= 1.10. Uses S3 conditional writes for locking.

### Dual Locking (Migration Period)

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-tofu-state"
    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "my-lock-table"
    use_lockfile   = true
  }
}
```

Both locks must be acquired. Remove dynamodb_table after migration.

### Windows Path Compatibility

```hcl
key = "${replace(path_relative_to_include(), "\\", "/")}/tofu.tfstate"
```

---

## GCS Backend (Google Cloud)

```hcl
remote_state {
  backend = "gcs"
  config = {
    bucket      = "my-gcs-bucket"
    prefix      = "tofu"
    project     = "my-gcp-project"
    location    = "us"
    credentials = "path/to/credentials.json"
  }
}
```

### All GCS Config Keys

- `bucket`: GCS bucket name
- `prefix`: state file prefix
- `project`: GCP project ID
- `location`: bucket location
- `credentials`: path to service account JSON
- `access_token`: OAuth2 token
- `skip_bucket_creation`: boolean
- `skip_bucket_versioning`: boolean
- `enable_bucket_policy_only`: boolean
- `encryption_key`: customer-managed encryption key
- `gcs_bucket_labels`: map of labels

### GCS Example with Labels

```hcl
remote_state {
  backend = "gcs"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket   = "my-tofu-state"
    prefix   = path_relative_to_include()
    project  = "my-gcp-project"
    location = "eu"
    gcs_bucket_labels = {
      owner = "terragrunt"
      env   = "shared"
    }
  }
}
```

---

## Azure Backend (Experimental)

Requires `azure-backend` experiment.

```hcl
remote_state {
  backend = "azurerm"
  config = {
    storage_account_name = "myterragruntstate"
    container_name       = "tfstate"
    key                  = "${path_relative_to_include()}/terraform.tfstate"
    resource_group_name  = "terraform-rg"
    subscription_id      = "00000000-0000-0000-0000-000000000000"
    use_azuread_auth     = true
  }
}
```

Note: Terragrunt does not auto-bootstrap Azure resources. The storage account and container must exist.

---

## Automatic Resource Creation

Terragrunt automatically creates remote state resources when they don't exist:

**S3**: Creates bucket with versioning, encryption, access logging, and DynamoDB table for locking.

**GCS**: Creates bucket with versioning and labels.

**Azure**: Does NOT auto-create (experimental).

### Disabling Auto-Creation

```hcl
remote_state {
  disable_init = tobool(get_env("TG_DISABLE_INIT", "false"))
  backend = "s3"
  config = { ... }
}
```

Or use `--backend-bootstrap` CLI flag for explicit bootstrap.

### Skipping Backend Init Entirely

```hcl
terraform {
  extra_arguments "no_backend_init" {
    commands  = ["init"]
    arguments = ["-backend=false"]
  }
}
```

---

## DRY Remote State Pattern

Place in root.hcl, included by all units:

```hcl
# root.hcl
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
```

`path_relative_to_include()` generates unique keys:
- `dev/us-east-1/vpc` -> `dev/us-east-1/vpc/tofu.tfstate`
- `prod/us-east-1/app` -> `prod/us-east-1/app/tofu.tfstate`

### Dynamic Remote State from Config

```hcl
locals {
  remote_state_config = read_terragrunt_config(find_in_parent_folders("common.hcl"))
}

remote_state = local.remote_state_config.remote_state
```

---

## OpenTofu State Encryption

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket = "my-bucket"
    key    = "path/to/key"
    region = "us-east-1"
  }
  encryption = {
    key_provider = "pbkdf2"
    passphrase   = get_env("PBKDF2_PASSPHRASE")
  }
}
```

Supported key providers: pbkdf2, aws_kms, gcp_kms, openbao.

---

## State Migration

When moving units (changing filesystem path), the state key changes:

```bash
# Migrate state to new location
terragrunt init -migrate-state

# Across all units
terragrunt run --all -- init -migrate-state

# Manual state operations
terragrunt state pull
terragrunt state push
```
