---
name: add-host-group-phase-1
description: "Phase 1: Inventory -- Add host(s) to the environment inventory"
metadata:
  author: platform-team
  version: "2.0"
parent: add-host-group
---

# Phase 1: Inventory

Add the host or host group to the appropriate environment inventory file.

**Typical PR**: 1
**Files produced**: updated hosts.yml (or created if new environment)

## Prerequisites

- Group name and environment from the router

**Carry forward from prior phases**: The agent already knows the group name and environment from the router. Use these values directly.

---

## Step 1: Add Hosts to Inventory

**Decide** (all questions upfront):
1. "How many hosts are you adding?" (1 for a single host, or a count for a group)
2. For each host:
   - "What is the hostname?" (e.g., `web-01.example.com`)
   - "What is the IP address or FQDN for SSH access?" (e.g., `10.0.1.50`)
3. "What SSH user should Ansible use?" (default: `ansible`)
4. "What SSH port?" (default: `22`)
5. "Does the host need a custom Python interpreter path?" (default: auto-detected)

**Inspect**:
- Check if the inventory file exists: search `inventory/{environment}/hosts.yml`
- If it exists, check if the group `{group-name}` is already defined
- Check if any of the hostnames are already listed

| Status | Action |
|--------|--------|
| Inventory file missing | Create a new inventory file with the group |
| Group already exists with these hosts | Show existing entries, ask if update is needed |
| Group exists but hosts are new | Append hosts to the existing group |
| Group does not exist | Add the group with hosts |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

**Generate** (new inventory file):

```yaml
all:
  children:
    {group-name}:
      hosts:
        {hostname-1}:
          ansible_host: {ip-or-fqdn-1}
          ansible_user: {ssh-user}
          ansible_port: {ssh-port}
        {hostname-2}:
          ansible_host: {ip-or-fqdn-2}
          ansible_user: {ssh-user}
          ansible_port: {ssh-port}
```

**Generate** (append to existing inventory -- add group under `all.children`):

```yaml
    {group-name}:
      hosts:
        {hostname-1}:
          ansible_host: {ip-or-fqdn-1}
          ansible_user: {ssh-user}
          ansible_port: {ssh-port}
```

Write to: `inventory/{environment}/hosts.yml`

---

## Validate (optional)

```bash
yamllint inventory/{environment}/hosts.yml
ansible-inventory -i inventory/{environment}/ --list > /dev/null
# Optional -- CI/CD is the actual gate
```

The `ansible-inventory --list` command parses the inventory and validates its structure without making any SSH connections.

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, inventory parses | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation (YAML inventory is indentation-sensitive) |
| Inventory parse error | Verify group/host nesting under `all.children` |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Inventory] Add {group-name} to {environment} inventory -- Phase 1`

**Files to include**:
- `inventory/{environment}/hosts.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with group variables and secrets:

`references/phase-02-group-vars.md`
