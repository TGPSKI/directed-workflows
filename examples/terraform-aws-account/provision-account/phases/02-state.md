---
name: provision-account-phase-2
description: "Phase 2: State Backend -- S3 remote state with DynamoDB locking"
metadata:
  author: platform-team
  version: "2.0"
parent: provision-account
---

# Phase 2: State Backend

Configure the remote state backend so Terraform state is stored in S3 with DynamoDB-based locking.

**This phase is skippable.** If your organization uses a shared state bucket that already exists, skip to Phase 3.

**Typical PR**: 1
**Files produced**: backend.tf

## Prerequisites

- Phase 1 complete (provider.tf and versions.tf merged)
- S3 bucket and DynamoDB table either already exist or will be created out-of-band

**Carry forward from prior phases**: The agent already knows the account ID, account alias, environment, region, and provider configuration from the router and Phase 1. Use these values directly.

---

## Should This Phase Run?

**Ask**: "Does a shared state bucket already exist for this organization, or do you need to configure a new backend?"

| Answer | Action |
|--------|--------|
| Shared bucket exists, already configured | Skip this phase. Continue with `@provision-account/phases/03-iam.md` |
| Need to configure backend for this account | Continue below |

---

## Step 1: Backend Configuration

**Ask** (all questions upfront):
1. "What is the S3 bucket name for Terraform state?" (e.g., `acme-terraform-state`)
2. "What key path should this account's state use?" (e.g., `accounts/{account-alias}/terraform.tfstate`)
3. "What is the DynamoDB table name for state locking?" (e.g., `terraform-lock`)
4. "What region is the state bucket in?" (default: same as provider region)
5. "Should state be encrypted at rest?" (default: yes)

**Inspect**:
- Verify Phase 1 provider.tf exists: search `accounts/{account-alias}/provider.tf`
- Check for existing backend config: search `accounts/{account-alias}/backend.tf`

| Status | Action |
|--------|--------|
| Provider file missing | Stop. Phase 1 must be completed first. |
| Backend file already exists | Show existing config, ask if update is needed |
| Ready to create | Continue with generation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```hcl
terraform {
  backend "s3" {
    bucket         = "{state-bucket-name}"
    key            = "{state-key-path}"
    region         = "{state-region}"
    dynamodb_table = "{dynamodb-table-name}"
    encrypt        = {encrypt}
  }
}
```

Write to: `accounts/{account-alias}/backend.tf`

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
| Validation error | Verify backend block syntax and required fields |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Provision] Add {account-alias} state backend -- Phase 2`

**Files to include**:
- `accounts/{account-alias}/backend.tf`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with IAM baseline:

`@provision-account/phases/03-iam.md`
