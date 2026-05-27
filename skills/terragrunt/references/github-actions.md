# Terragrunt GitHub Actions and CI/CD

## Table of Contents

- [Good IaC CI Properties](#good-iac-ci-properties)
- [Terragrunt Scale](#terragrunt-scale)
- [OIDC Authentication Flow](#oidc-authentication-flow)
- [IAM Policies](#iam-policies)
- [Bootstrap Stack Configuration](#bootstrap-stack-configuration)
- [GitOps Workflow](#gitops-workflow)
- [Gruntwork GitHub App](#gruntwork-github-app)
- [terragrunt-action (Official GitHub Action)](#terragrunt-action-official-github-action)
- [Infrastructure Repository Patterns](#infrastructure-repository-patterns)
- [SSH Key Setup for Private Repos in CI](#ssh-key-setup-for-private-repos-in-ci)
- [Report Generation in CI](#report-generation-in-ci)

---

## Good IaC CI Properties

A well-designed infrastructure CI/CD pipeline exhibits four key properties:

1. **Plan on every pull request** -- Every PR triggers a plan so reviewers can examine both the code diff and the computed dry-run output showing what will actually change. This catches misconfigurations before they reach production.

2. **Apply on merge to main** -- The main (or deploy) branch is the single source of truth for deployed infrastructure. Merging a PR triggers apply, ensuring the live environment always matches what is checked in.

3. **DAG-aware orchestration** -- Terragrunt understands the dependency graph between units. CI must respect this ordering: dependent resources plan/apply after their dependencies, and destroy operations run in reverse order. Using `terragrunt run --all` handles this automatically.

4. **OIDC-based authentication** -- CI runners authenticate to cloud providers using OpenID Connect tokens, which are short-lived and scoped. This eliminates the need for long-lived IAM access keys stored as repository secrets. GitHub Actions natively supports OIDC token issuance.

---

## Terragrunt Scale

Terragrunt Scale is Gruntwork's managed CI/CD platform purpose-built for Terragrunt. It provides out-of-the-box plan-on-PR and apply-on-merge workflows without requiring manual GitHub Actions authoring.

- Free tier available at https://app.gruntwork.io/signup/terragrunt-scale-free-tier
- Supported platform: GitHub
- Includes:
  - **Pipelines** -- CI/CD orchestration for plan and apply
  - **Drift Detection** -- Scheduled checks for infrastructure drift
  - **Patcher** -- Automated dependency update PRs for module versions

### What You Get

- Automatic plan on PR creation and every subsequent push to the PR branch
- Plan results posted as PR comments via the Gruntwork GitHub App
- Automatic apply on merge to the deploy branch (typically `main`)
- DAG-aware execution that respects inter-unit dependencies
- Drift detection on a configurable schedule
- Separate IAM roles for plan (read-only) and apply (read-write) operations
- Support for multi-account AWS deployments

---

## OIDC Authentication Flow

OIDC (OpenID Connect) eliminates long-lived cloud credentials in CI. The flow works as follows:

1. The GitHub Actions runner requests a short-lived JWT from GitHub's built-in OIDC provider (`token.actions.githubusercontent.com`).
2. The runner presents this JWT to AWS STS via `sts:AssumeRoleWithWebIdentity`.
3. AWS validates the JWT signature against GitHub's OIDC provider, checks the trust policy conditions (audience, subject claims), and issues temporary AWS credentials.
4. Separate IAM roles are used for plan and apply operations, enforcing least-privilege access.

### IAM Trust Policy for GitHub Actions OIDC

This trust policy allows a GitHub Actions workflow in a specific repository to assume the IAM role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*"
        }
      }
    }
  ]
}
```

Key points:
- The `sub` claim can be scoped to specific branches (e.g., `repo:my-org/my-repo:ref:refs/heads/main`) or left as wildcard `*` for all branches.
- The OIDC provider must be registered as an identity provider in the AWS account before use.
- The `aud` claim must match `sts.amazonaws.com` when using the official `aws-actions/configure-aws-credentials` action.

---

## IAM Policies

### Plan Role (Read-Only)

The plan role needs read access to the state bucket, lock table, and the resources being managed. It must also be able to acquire and release DynamoDB locks (since `terraform plan` acquires a state lock).

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3StateAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketVersioning",
        "s3:GetObject",
        "s3:GetBucketLocation"
      ],
      "Resource": [
        "arn:aws:s3:::my-tofu-state",
        "arn:aws:s3:::my-tofu-state/*"
      ]
    },
    {
      "Sid": "DynamoDBLockAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:DescribeTable",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:*:123456789012:table/my-lock-table"
    }
  ]
}
```

Additional resource-read permissions (e.g., `ec2:Describe*`, `s3:Get*`, `iam:Get*`) should be added based on the infrastructure being managed. The plan role should never have write access to managed resources.

### Apply Role (Read + Write)

The apply role includes all plan role permissions plus write access to managed resources. It needs:

- All S3 state bucket permissions from plan, plus `s3:PutObject` and `s3:DeleteObject` for state writes
- Full DynamoDB lock table access
- Create, update, and delete permissions for all managed resource types

The apply role should be scoped as narrowly as possible. Only grant permissions for resource types that are actually managed by the Terragrunt configurations.

---

## Bootstrap Stack Configuration

Terragrunt Scale provides a catalog stack that bootstraps the OIDC identity provider, plan role, and apply role in a single operation.

```hcl
stack "bootstrap" {
  source = "github.com/gruntwork-io/terragrunt-scale-catalog//stacks/aws/github/pipelines-bootstrap?ref=v1.10.1"
  path   = "bootstrap"

  values = {
    terragrunt_scale_catalog_ref = "v1.10.1"
    aws_account_id    = "123456789012"
    oidc_resource_prefix = "pipelines"
    github_org_name   = "my-org"
    github_repo_name  = "infrastructure-live"
    deploy_branch     = "main"
    state_bucket_name = "my-tofu-state"
    plan_iam_policy   = templatefile("${get_terragrunt_dir()}/plan_iam_policy.json", {
      state_bucket = "my-tofu-state"
      lock_table   = "my-lock-table"
      account_id   = "123456789012"
    })
    apply_iam_policy  = templatefile("${get_terragrunt_dir()}/apply_iam_policy.json", {
      state_bucket = "my-tofu-state"
      lock_table   = "my-lock-table"
      account_id   = "123456789012"
    })
  }
}
```

This creates:
- The GitHub OIDC identity provider in the AWS account (if not already present)
- `pipelines-plan` IAM role with OIDC trust and the supplied plan policy
- `pipelines-apply` IAM role with OIDC trust and the supplied apply policy

The `oidc_resource_prefix` controls the naming of the created IAM roles. The `deploy_branch` restricts which branch can assume the apply role.

---

## GitOps Workflow

The standard GitOps workflow for infrastructure changes:

1. Clone the infrastructure-live repository and create a feature branch.
2. Make infrastructure changes (add, modify, or remove Terragrunt units).
3. Commit and push the feature branch.
4. Create a pull request targeting the deploy branch (e.g., `main`).
5. GitHub Actions triggers and runs `terragrunt run --all plan` for affected units.
6. Plan results are posted as PR comments by the Gruntwork GitHub App.
7. Team reviews the code diff and plan output, then approves the PR.
8. Merge the PR to the deploy branch.
9. GitHub Actions triggers and runs `terragrunt run --all apply` for affected units.
10. Apply results are posted as a comment on the merge commit.

This workflow ensures that:
- No infrastructure change is applied without review.
- The deploy branch always reflects the current state of deployed infrastructure.
- Plan output is available alongside the code diff for informed review.

---

## Gruntwork GitHub App

The Gruntwork GitHub App is installed during Terragrunt Scale onboarding and serves as the interface between CI/CD pipelines and pull request feedback.

Capabilities:
- Posts plan output as PR comments
- Posts apply results as merge commit comments
- Accesses git repositories to read configuration
- Triggers CI runners for plan and apply operations

Security boundary:
- The GitHub App does NOT access cloud infrastructure directly.
- The GitHub App does NOT read or write Terraform/OpenTofu state.
- All cloud operations are performed by the CI runner using OIDC-assumed IAM roles.

---

## terragrunt-action (Official GitHub Action)

Repository: https://github.com/gruntwork-io/terragrunt-action

The official GitHub Action installs Terragrunt and OpenTofu (or Terraform) on the runner. It handles version management and binary caching.

### Basic Plan Workflow

Triggered on every pull request targeting `main`. Uses the plan IAM role via OIDC.

```yaml
name: Terragrunt Plan

on:
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/pipelines-plan
          aws-region: us-east-1

      - name: Setup Terragrunt
        uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"

      - name: Terragrunt Plan
        run: terragrunt run --all plan
```

Key permissions:
- `id-token: write` -- Required for OIDC token generation.
- `contents: read` -- Required to check out the repository.
- `pull-requests: write` -- Required if posting plan output as PR comments.

### Basic Apply Workflow

Triggered on push (merge) to `main`. Uses the apply IAM role via OIDC.

```yaml
name: Terragrunt Apply

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/pipelines-apply
          aws-region: us-east-1

      - name: Setup Terragrunt
        uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"

      - name: Terragrunt Apply
        run: terragrunt run --all apply
```

### Plan with Affected Units Only

Use the `--filter` flag with a git range to plan only units affected by the changes in the PR. This significantly reduces CI time in large repositories.

```yaml
      - name: Plan Affected
        run: |
          terragrunt run --all \
            --filter '[origin/main...HEAD]' \
            -- plan
```

The filter `[origin/main...HEAD]` computes the git diff between the base branch and the PR head, then selects only the Terragrunt units whose files were modified. Dependencies of affected units are also included.

### Multi-Account Deploy

Run plan or apply against multiple AWS accounts in parallel using separate jobs with different IAM roles.

```yaml
jobs:
  plan-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/pipelines-plan
          aws-region: us-east-1
      - uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"
      - run: terragrunt run --all --working-dir dev -- plan

  plan-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::222222222222:role/pipelines-plan
          aws-region: us-east-1
      - uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"
      - run: terragrunt run --all --working-dir prod -- plan
```

Each account has its own OIDC-trusted IAM roles. The `--working-dir` flag scopes Terragrunt to only the units under the specified directory.

### Progressive Rollout

Chain deployments across environments using job dependencies. Each environment applies only after the previous one succeeds.

```yaml
jobs:
  apply-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111111111111:role/pipelines-apply
          aws-region: us-east-1
      - uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"
      - run: terragrunt run --all --working-dir dev -- apply

  apply-staging:
    needs: apply-dev
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::333333333333:role/pipelines-apply
          aws-region: us-east-1
      - uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"
      - run: terragrunt run --all --working-dir staging -- apply

  apply-prod:
    needs: apply-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::222222222222:role/pipelines-apply
          aws-region: us-east-1
      - uses: gruntwork-io/terragrunt-action@v2
        with:
          tofu_version: "1.9.0"
          tg_version: "latest"
      - run: terragrunt run --all --working-dir prod -- apply
```

The `needs` keyword creates the sequential dependency: dev -> staging -> prod. If any environment fails, subsequent environments are skipped.

---

## Infrastructure Repository Patterns

### infrastructure-live Repository

The `infrastructure-live` repository contains Terragrunt configurations that represent deployed infrastructure. Key practices:

- **Trunk-based development** -- Short-lived feature branches merged frequently to `main`.
- **Plan via PR** -- Every pull request triggers a plan so reviewers see the exact changes.
- **Apply on merge** -- Merging to `main` triggers apply, making the branch the source of truth.
- **Environment directories** -- Separate top-level directories for each environment or account (e.g., `dev/`, `staging/`, `prod/`).
- **Default branch is source of truth** -- The `main` branch always reflects what is currently deployed.

### infrastructure-modules Repository

The `infrastructure-modules` repository contains reusable OpenTofu/Terraform modules consumed by `infrastructure-live`. Key practices:

- **Semantic versioning** -- Modules are tagged with semver (e.g., `v1.2.3`).
- **Pinned versions** -- Terragrunt units in `infrastructure-live` reference modules at specific versions, never `main` or `latest`.
- **Testing with Terratest** -- Modules include integration tests that provision real infrastructure, validate it, and tear it down.
- **Separate CI pipeline** -- Module CI runs tests on PR; tagging a release publishes the new version.

Example of a Terragrunt unit referencing a versioned module:

```hcl
terraform {
  source = "git::git@github.com:my-org/infrastructure-modules.git//modules/vpc?ref=v1.5.0"
}
```

---

## SSH Key Setup for Private Repos in CI

When Terragrunt units reference modules from private Git repositories, the CI runner needs SSH access. Use a deploy key scoped to the modules repository.

```yaml
      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.INFRA_MODULES_DEPLOY_KEY }}

      - name: Add GitHub to known hosts
        run: ssh -T -oStrictHostKeyChecking=accept-new git@github.com || true
```

Notes:
- The deploy key should be read-only and scoped to the `infrastructure-modules` repository.
- The `ssh -T` command with `StrictHostKeyChecking=accept-new` adds GitHub's host key to `~/.ssh/known_hosts` on first connection, preventing interactive prompts.
- Store the private key as a GitHub Actions encrypted secret. Never commit private keys to the repository.
- For multiple private module repos, add multiple deploy keys to the `ssh-agent` step or use a machine user SSH key with access to all required repositories.

---

## Report Generation in CI

Terragrunt can generate structured reports during plan and apply operations. These reports are useful for audit trails, compliance, and automated analysis.

```yaml
      - name: Plan with Report
        run: |
          terragrunt run --all \
            --report-file plan-report.json \
            --report-format json \
            -- plan

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: plan-report
          path: plan-report.json
```

The JSON report includes:
- List of units that were planned
- Status of each unit (success, failure, no changes)
- Execution order based on the dependency graph
- Timing information for each unit

Reports can also be generated during apply:

```yaml
      - name: Apply with Report
        run: |
          terragrunt run --all \
            --report-file apply-report.json \
            --report-format json \
            -- apply

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: apply-report
          path: apply-report.json
```

Uploaded artifacts are retained according to the repository's artifact retention policy and can be downloaded from the GitHub Actions run summary.
