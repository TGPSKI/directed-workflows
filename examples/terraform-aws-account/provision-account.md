---
name: provision-account
description: Provision a new AWS account in a Terraform-managed infrastructure repo. Detects current progress and routes to the appropriate phase.
metadata:
  author: platform-team
  version: "2.0"
compatibility: Works with any AI IDE that supports file references. Requires Terraform CLI for local validation (terraform fmt, terraform validate). No AWS credentials needed -- the workflow produces .tf files validated locally.
---

# Provision AWS Account

Walk through the complete process of provisioning a new AWS account in your Terraform infrastructure repo. This workflow produces small, focused pull requests -- one per phase.

## Design Principles

- **Ownership**: The team provisions the account; security reviews the IAM baseline. The agent assists with provisioning but does not bypass review.
- **Source of truth**: Use existing provider configs and module patterns in the repo. Search for similar accounts before generating from scratch.
- **Ask less, infer more**: Derive bucket names from the account alias (e.g., `{org}-terraform-state`). Verify state from the filesystem before asking the user.
- **Prefer simple**: Skip state locking if single-user; skip backend creation if a shared bucket exists. Default to the minimal viable configuration.

## Prerequisites

- **Start from an up-to-date primary branch.** Sync your fork and rebase onto the latest upstream `main` before invoking this workflow (or `git pull` if working on a direct clone). Progress detection relies on files merged to the upstream primary branch -- not local commits, uncommitted changes, or prior session output. If you generated files in a previous session that haven't been merged yet, they don't count as completed work.
- **Start with a clean chat session.** Prior conversation history causes the agent to conflate previous output with actual repository state. If you have an existing session about this account, start a new one.

## Entry Point

**Ask** (all questions upfront):
1. "What is the AWS account ID?" (e.g., `123456789012`)
2. "What alias or short name should identify this account?" (e.g., `acme-staging`, used for directory naming)
3. "What environment is this?" (dev / staging / production)

## Detect Current Progress

After getting the account alias, inspect the repository to determine what's already been completed. **Detection must be based on files merged to the primary branch** -- not on local uncommitted files, unmerged feature branches, or content from prior chat sessions. If a file was generated in a previous session but its PR hasn't merged, that phase is not complete.

### Progress Detection

| Check | What to Look For | Indicates |
|-------|-----------------|-----------|
| Provider config | `accounts/{account-alias}/provider.tf` | Phase 1 complete |
| State backend | `accounts/{account-alias}/backend.tf` | Phase 2 complete |
| IAM baseline | `accounts/{account-alias}/iam-baseline.tf` | Phase 3 complete |

### Determine Phase

| Detected State | Recommended Action |
|----------------|-------------------|
| No account directory | Start at **Phase 1** |
| Provider exists, no backend | Continue at **Phase 2** |
| Backend exists, no IAM baseline | Continue at **Phase 3** |
| IAM baseline exists | Provisioning complete |

## Route to Phase

**Ask**: "Based on my analysis, you appear to be at [detected phase]. Would you like to:
1. Continue from [detected phase]
2. Start from a different phase
3. Review what's already configured"

| User Choice | Action |
|-------------|--------|
| Continue detected phase | Load the corresponding phase file |
| Different phase | Ask which phase, then load that file |
| Review configuration | Summarize existing files, then ask which phase |

## Variant Paths

| Scenario | Phases |
|----------|--------|
| Standard (new account, new state bucket) | 1 → 2 → 3 |
| Existing state bucket | 1 → 3 (skip Phase 2 backend creation if bucket already exists) |

## Phase Files

| Phase | File | Description | PR |
|-------|------|-------------|-----|
| 1 | `@provision-account/phase-1-provider.md` | AWS provider, version constraints | 1 |
| 2 | `@provision-account/phase-2-state.md` | S3 backend, DynamoDB lock table | 1 |
| 3 | `@provision-account/phase-3-iam.md` | IAM baseline roles, outputs | 1 |
