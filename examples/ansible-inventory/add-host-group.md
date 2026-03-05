---
name: add-host-group
description: Add a new host or host group to an Ansible-managed infrastructure repo. Detects current progress and routes to the appropriate phase.
metadata:
  author: platform-team
  version: "2.0"
compatibility: Works with any AI IDE that supports file references. Requires ansible-lint and yamllint for local validation. No SSH access needed -- the workflow produces inventory and playbook files validated locally.
---

# Add Host Group

Walk through the complete process of adding a new host or host group to your Ansible infrastructure repo. This workflow produces small, focused pull requests -- one per phase.

## Design Principles

- **Ownership**: The ops team creates inventory and playbook configuration; the platform lead reviews and merges. The agent assists the contributor, it does not act as the approver.
- **Source of truth**: Use existing inventory files and `group_vars` as reference. Do not invent structure -- derive it from what already exists in the repo.
- **Ask less, infer more**: Derive group names from the environment, reuse existing vault password file paths, and infer SSH defaults from other hosts in the same environment.
- **Prefer simple**: Skip vault encryption for non-sensitive variables. Default to minimal playbooks; add pre-tasks and handlers only when the user needs them.

## Prerequisites

- **Start from an up-to-date primary branch.** Sync your fork and rebase onto the latest upstream `main` before invoking this workflow (or `git pull` if working on a direct clone). Progress detection relies on files merged to the upstream primary branch -- not local commits, uncommitted changes, or prior session output. If you generated files in a previous session that haven't been merged yet, they don't count as completed work.
- **Start with a clean chat session.** Prior conversation history causes the agent to conflate previous output with actual repository state. If you have an existing session about this host, start a new one.

## Entry Point

**Ask** (all questions upfront):
1. "What is the hostname or group name you're adding?" (e.g., `web-servers`, `db-primary`)
2. "What environment is this for?" (dev / staging / production)

## Detect Current Progress

After getting the group name and environment, inspect the repository to determine what's already been completed. **Detection must be based on files merged to the primary branch** -- not on local uncommitted files, unmerged feature branches, or content from prior chat sessions. If a file was generated in a previous session but its PR hasn't merged, that phase is not complete.

### Progress Detection

| Check | What to Look For | Indicates |
|-------|-----------------|-----------|
| Inventory entry | `inventory/{environment}/hosts.yml` contains the group | Phase 1 complete |
| Group vars | `inventory/{environment}/group_vars/{group-name}.yml` | Phase 2 complete |
| Playbook | `playbooks/{group-name}.yml` | Phase 3 complete |

### Determine Phase

| Detected State | Recommended Action |
|----------------|-------------------|
| Group not in inventory | Start at **Phase 1** |
| Group in inventory, no group_vars | Continue at **Phase 2** |
| Group vars exist, no playbook | Continue at **Phase 3** |
| Playbook exists | Setup complete |

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
| Single host | 1 → 2 → 3 |
| Group of hosts | 1 → 2 → 3 (multiple hosts added in Phase 1) |

## Phase Files

| Phase | File | Description | PR |
|-------|------|-------------|-----|
| 1 | `@add-host-group/phase-1-inventory.md` | Add host(s) to environment inventory | 1 |
| 2 | `@add-host-group/phase-2-group-vars.md` | Group variables, vault-encrypted secrets | 1 |
| 3 | `@add-host-group/phase-3-playbook.md` | Playbook targeting the group with roles | 1 |
