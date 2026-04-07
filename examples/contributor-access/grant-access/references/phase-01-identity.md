---
name: grant-access-phase-1
description: "Phase 1: Identity -- Create user file and add to team"
metadata:
  author: platform-team
  version: "2.0"
parent: grant-access
---

# Phase 1: Identity

Create the contributor's user file and add them to the appropriate team.

**Typical PR**: 1
**Files produced**: {username}.yml, updated team file

## Prerequisites

- Contributor's name and username from the router
- Knowledge of which team they belong to

**Carry forward from prior phases**: The agent already knows the contributor's name and username from the router. Use these values directly.

---

## Step 1: User File

**Decide** (all questions upfront):
1. "What is the contributor's email address?"
2. "What is their GitHub username?"
3. "What is their Slack handle?" (optional)

**Inspect**:
- Check if a user file already exists: search `team/{username}.yml`

| Status | Action |
|--------|--------|
| User file exists | Show existing file, ask if it needs updating |
| User file missing | Continue with creation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
name: {full-name}
username: {username}
email: {email}
github: {github-username}
slack: {slack-handle}
joined: {today's date}
```

Write to: `team/{username}.yml`

---

## Step 2: Team Membership

**Decide**:
1. "Which team should this contributor join?" (e.g., `backend`, `platform`, `frontend`)

**Inspect**:
- Check if the team file exists: search `team/teams/{team-name}.yml`
- Check if the user is already listed in the team file

| Status | Action |
|--------|--------|
| Team file missing | Stop. The team must be created first, or verify the team name. |
| User already in team | Show existing membership, skip to PR Checkpoint |
| Team exists, user not listed | Continue with update |

**Generate**: Add a reference to the user file in the team's members list:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
members:
  # ... existing members ...
  - $ref: team/{username}.yml
```

Update: `team/teams/{team-name}.yml`

---

## Validate (optional)

<!-- Optional -- CI/CD is the actual gate -->

Verify the user file is valid YAML and the team file reference resolves correctly.

| Validation Result | Action |
|-------------------|--------|
| Files are valid | Continue to PR Checkpoint |
| YAML syntax error | Fix and re-validate |
| Team ref doesn't resolve | Verify the user file path matches the `$ref` |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Access] Add contributor {username} -- Phase 1`

**Files to include**:
- `team/{username}.yml`
- `team/teams/{team-name}.yml` (updated)

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with access and permissions:

`references/phase-02-access.md`
