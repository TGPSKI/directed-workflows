---
name: grant-access
description: "Grant access to a new contributor. Detects current progress and routes to the appropriate phase."
metadata:
  author: platform-team
  version: "2.0"
compatibility: Works with any AI IDE that supports file references. No external tooling required.
---

# Grant Contributor Access

Add a new contributor to the repository with the correct identity and access configuration.

## Design Principles

- **Ownership**: The contributor submits the access request; the maintainer reviews and merges. The agent assists the requester, it does not act as the approver.
- **Source of truth**: Use existing user files and team definitions as reference. Match the structure and conventions already in the repo.
- **Ask less, infer more**: Derive team membership from existing files, infer access patterns from similar contributors, and verify paths before asking.
- **Prefer simple**: Default to read access; offer write or admin only when the user explicitly needs it.

## Prerequisites

- **Start from an up-to-date primary branch.** Sync your fork and rebase onto the latest upstream `main` before invoking this workflow (or `git pull` if working on a direct clone). Progress detection relies on files merged to the upstream primary branch -- not local commits, uncommitted changes, or prior session output.
- **Start with a clean chat session.** Prior conversation history causes the agent to conflate previous output with actual repository state.

## Entry Point

**Ask**:
1. "What is the contributor's full name?"
2. "What is their username?" (e.g., GitHub handle)

## Detect Current Progress

After getting the username, inspect the repository to determine what's already been completed. **Detection must be based on files merged to the primary branch** -- not on local uncommitted files, unmerged feature branches, or content from prior chat sessions.

### Progress Detection

| Check | What to Look For | Indicates |
|-------|-----------------|-----------|
| User file | `team/{username}.yml` | Phase 1 complete |
| Role binding | `access/{username}-roles.yml` | Phase 2 complete |

### Determine Phase

| Detected State | Recommended Action |
|----------------|-------------------|
| No user file | Start at **Phase 1** |
| User file exists, no role binding | Continue at **Phase 2** |
| Both exist | Access configured |

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

## Phase Files

| Phase | File | Description | PR |
|-------|------|-------------|-----|
| 1 | `references/phase-01-identity.md` | User file, team membership | 1 |
| 2 | `references/phase-02-access.md` | Role assignment, repo permissions | 1 |
