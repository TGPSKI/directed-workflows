---
name: grant-access-phase-2
description: "Phase 2: Access -- Role assignment and repository permissions"
metadata:
  author: platform-team
  version: "2.0"
parent: grant-access
---

# Phase 2: Access

Assign roles and configure repository permissions for the contributor.

**Typical PR**: 1
**Files produced**: {username}-roles.yml, updated CODEOWNERS

## Prerequisites

- Phase 1 complete (user file and team membership merged)

**Carry forward from prior phases**: The agent already knows the contributor's name, username, email, GitHub handle, Slack handle, and team from the router and Phase 1. Use these values directly.

---

## Step 1: Role Assignment

**Decide** (all questions upfront):
1. "What access level should this contributor have?"
   - `read-only` -- Can view all config, cannot modify
   - `contributor` -- Can modify config in their team's directories
   - `admin` -- Full write access across the repository
2. "Are there specific directories this contributor should own?" (optional, e.g., `apps/payment-gateway/`)

**Inspect**:
- Check if a roles file already exists: search `access/{username}-roles.yml`
- Verify the user file from Phase 1 exists: search `team/{username}.yml`

| Status | Action |
|--------|--------|
| User file missing | Stop. Phase 1 must be completed first. |
| Roles file already exists | Show existing roles, ask if update is needed |
| No existing roles | Continue with creation |

| Access Level | Permissions |
|-------------|------------|
| read-only | View all directories, no write access |
| contributor | Read all, write to `apps/{team}/**`, `monitoring/{team}/**` |
| admin | Read and write to all directories |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
username: {username}
user_ref: team/{username}.yml
access_level: {access-level}
permissions:
  - path: {directory-scope}
    actions: [read, write]
granted_by: {current-user}
granted_date: {today's date}
```

Write to: `access/{username}-roles.yml`

---

## Step 2: Repository Permissions

**Decide**:
1. "Should this contributor be added as a code owner for any paths?" (e.g., `apps/payment-gateway/`)

| Answer | Action |
|--------|--------|
| No code ownership needed | Skip to Validate (optional) |
| Yes, specific paths | Continue with CODEOWNERS update |

**Inspect**:
- Check if CODEOWNERS file exists at repo root
- Check if the contributor is already listed for the requested paths

| Status | Action |
|--------|--------|
| CODEOWNERS missing | Create the file |
| Contributor already listed for path | Show existing entry, skip |
| Ready to add | Continue with update |

**Generate**: Append to CODEOWNERS:

Search for existing files matching this pattern in the repo before generating from scratch.

```
# {full-name} ({team-name})
{directory-path}  @{github-username}
```

Update: `CODEOWNERS`

---

## Validate (optional)

<!-- Optional -- CI/CD is the actual gate -->

Verify the roles file is valid YAML and the user reference resolves.

| Validation Result | Action |
|-------------------|--------|
| Files are valid | Continue to PR Checkpoint |
| User ref doesn't resolve | Verify Phase 1 PR was merged and path is correct |
| CODEOWNERS syntax error | Verify path patterns and GitHub username format |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Access] Add {username} permissions -- Phase 2`

**Files to include**:
- `access/{username}-roles.yml`
- `CODEOWNERS` (if updated)

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, the access configuration is complete. The workflow will detect the merged files and confirm the contributor is fully configured.

---

## Access Configured

The contributor now has:

| Component | Location |
|-----------|----------|
| User identity | `team/{username}.yml` |
| Team membership | `team/teams/{team-name}.yml` |
| Access roles | `access/{username}-roles.yml` |
| Code ownership | `CODEOWNERS` (if applicable) |
