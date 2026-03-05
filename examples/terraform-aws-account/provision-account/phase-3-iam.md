---
name: provision-account-phase-3
description: "Phase 3: IAM Baseline -- Admin and read-only roles, password policy, outputs"
metadata:
  author: platform-team
  version: "2.0"
parent: provision-account.md
---

# Phase 3: IAM Baseline

Create the baseline IAM resources: an admin role, a read-only role, and a password/MFA policy. Export the role ARNs as outputs for downstream consumers.

**Typical PR**: 1
**Files produced**: iam-baseline.tf, outputs.tf

## Prerequisites

- Phase 1 complete (provider.tf merged)
- Phase 2 complete or skipped (backend.tf merged, or shared bucket in use)

**Carry forward from prior phases**: The agent already knows the account ID, account alias, environment, region, provider config, and backend configuration from the router and Phases 1–2. Use these values directly.

---

## Step 1: IAM Roles

**Decide**:
1. "What should the admin role be named?" (default: `{account-alias}-admin`)
2. "What should the read-only role be named?" (default: `{account-alias}-readonly`)
3. "Is this a cross-account setup? If so, what is the trusted account ID that can assume these roles?" (e.g., the management account ID)
4. "Should MFA be required for role assumption?" (default: yes)

**Inspect**:
- Verify Phase 1 provider.tf exists: search `accounts/{account-alias}/provider.tf`
- Check for existing IAM baseline: search `accounts/{account-alias}/iam-baseline.tf`

| Status | Action |
|--------|--------|
| Provider file missing | Stop. Phase 1 must be completed first. |
| IAM baseline already exists | Show existing config, ask if update is needed |
| Ready to create | Continue with generation |

**Generate** (cross-account with MFA):

Search for existing files matching this pattern in the repo before generating from scratch.

```hcl
data "aws_iam_policy_document" "trust_policy" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::{trusted-account-id}:root"]
    }

    condition {
      test     = "Bool"
      variable = "aws:MultiFactorAuthPresent"
      values   = ["true"]
    }
  }
}

resource "aws_iam_role" "admin" {
  name               = "{admin-role-name}"
  assume_role_policy = data.aws_iam_policy_document.trust_policy.json

  tags = {
    Purpose = "Administrative access"
  }
}

resource "aws_iam_role_policy_attachment" "admin" {
  role       = aws_iam_role.admin.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

resource "aws_iam_role" "readonly" {
  name               = "{readonly-role-name}"
  assume_role_policy = data.aws_iam_policy_document.trust_policy.json

  tags = {
    Purpose = "Read-only access"
  }
}

resource "aws_iam_role_policy_attachment" "readonly" {
  role       = aws_iam_role.readonly.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

Write to: `accounts/{account-alias}/iam-baseline.tf`

---

## Step 2: Password Policy

No additional questions needed -- this applies a sensible default.

**Inspect**:
- Check if a password policy resource already exists in `accounts/{account-alias}/iam-baseline.tf`

| Status | Action |
|--------|--------|
| Password policy already defined | Skip, show existing config |
| Not defined | Append to iam-baseline.tf |

**Generate** (append to iam-baseline.tf):

Search for existing password policy resources in the repo before generating from scratch.

```hcl
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_uppercase_characters   = true
  require_numbers                = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 12
}
```

---

## Step 3: Outputs

No additional questions needed -- outputs are derived from the roles created above.

**Generate**:

Search for existing outputs.tf files in the repo before generating from scratch.

```hcl
output "admin_role_arn" {
  description = "ARN of the admin IAM role"
  value       = aws_iam_role.admin.arn
}

output "readonly_role_arn" {
  description = "ARN of the read-only IAM role"
  value       = aws_iam_role.readonly.arn
}

output "account_alias" {
  description = "Account alias for reference"
  value       = "{account-alias}"
}
```

Write to: `accounts/{account-alias}/outputs.tf`

---

## Validate (optional)

```bash
cd accounts/{account-alias}
terraform fmt -check
terraform validate
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Formatting passes, valid HCL | Continue to PR Checkpoint |
| Formatting issues | Run `terraform fmt` to auto-fix, then re-check |
| Validation error | Verify IAM resource references and policy document syntax |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Provision] Add {account-alias} IAM baseline -- Phase 3`

**Files to include**:
- `accounts/{account-alias}/iam-baseline.tf`
- `accounts/{account-alias}/outputs.tf`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Provisioning Complete

The account is now configured in Terraform. After this PR merges, your CI/CD pipeline will plan and apply the resources.

| Component | Location |
|-----------|----------|
| Provider + versions | `accounts/{account-alias}/provider.tf`, `versions.tf` |
| State backend | `accounts/{account-alias}/backend.tf` |
| IAM baseline | `accounts/{account-alias}/iam-baseline.tf` |
| Outputs | `accounts/{account-alias}/outputs.tf` |
