---
name: provision-account-phase-1
description: "Phase 1: Provider -- AWS provider configuration and Terraform version constraints"
metadata:
  author: platform-team
  version: "2.0"
parent: provision-account
---

# Phase 1: Provider

Configure the AWS provider and pin Terraform and provider versions for the new account.

**Typical PR**: 1
**Files produced**: provider.tf, versions.tf

## Prerequisites

- Account ID, alias, and environment from the router

**Carry forward from prior phases**: The agent already knows the account ID, account alias, and environment from the router. Use these values directly.

---

## Step 1: Terraform Version Constraints

**Decide**:
1. "What minimum Terraform version should be required?" (default: `>= 1.5`)
2. "What AWS provider version constraint?" (default: `~> 5.0`)

**Inspect**:
- Check if the account directory already exists: search `accounts/{account-alias}/`

| Status | Action |
|--------|--------|
| Directory exists with `versions.tf` | Show existing constraints, ask if update is needed |
| Directory exists without `versions.tf` | Continue with creation |
| Directory does not exist | Create it, continue with creation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```hcl
terraform {
  required_version = "{terraform-version-constraint}"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "{aws-provider-version-constraint}"
    }
  }
}
```

Write to: `accounts/{account-alias}/versions.tf`

---

## Step 2: AWS Provider

**Ask** (all questions upfront):
1. "What AWS region?" (e.g., `us-east-1`)
2. "How does Terraform authenticate to this account?"
   - `assume_role` -- Cross-account role assumption (recommended for multi-account)
   - `default` -- Use ambient credentials (environment variables, shared config)
3. If assume_role: "What is the role ARN to assume?" (e.g., `arn:aws:iam::{account-id}:role/TerraformAdmin`)

**Inspect**:
- Check for existing provider config: search `accounts/{account-alias}/provider.tf`

| Status | Action |
|--------|--------|
| Provider file exists | Show existing config, ask if update is needed |
| Provider file missing | Continue with creation |

**Generate** (assume_role variant):

Search for existing provider configs in the repo before generating from scratch.

```hcl
provider "aws" {
  region = "{region}"

  assume_role {
    role_arn = "{assume-role-arn}"
  }

  default_tags {
    tags = {
      Account     = "{account-alias}"
      Environment = "{environment}"
      ManagedBy   = "terraform"
    }
  }
}
```

**Generate** (default credentials variant):

```hcl
provider "aws" {
  region = "{region}"

  default_tags {
    tags = {
      Account     = "{account-alias}"
      Environment = "{environment}"
      ManagedBy   = "terraform"
    }
  }
}
```

Write to: `accounts/{account-alias}/provider.tf`

---

## Validate (optional)

```bash
cd accounts/{account-alias}
terraform fmt -check
terraform validate
```

| Validation Result | Action |
|-------------------|--------|
| Formatting passes, valid HCL | Continue to PR Checkpoint |
| Formatting issues | Run `terraform fmt` to auto-fix, then re-check |
| Validation error | Verify `required_providers` source and version syntax |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Provision] Add {account-alias} provider config -- Phase 1`

**Files to include**:
- `accounts/{account-alias}/versions.tf`
- `accounts/{account-alias}/provider.tf`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with state backend configuration:

`@provision-account/references/02-state.md`

If the state bucket already exists and is shared across accounts, skip to IAM baseline:

`@provision-account/references/03-iam.md`
