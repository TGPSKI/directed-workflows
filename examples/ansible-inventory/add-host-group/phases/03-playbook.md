---
name: add-host-group-phase-3
description: "Phase 3: Playbook -- Create the playbook targeting the host group with roles"
metadata:
  author: platform-team
  version: "2.0"
parent: add-host-group
---

# Phase 3: Playbook

Create the playbook that targets the new host group, applying the appropriate roles.

**Typical PR**: 1
**Files produced**: {group-name}.yml

## Prerequisites

- Phase 1 complete (hosts in inventory, PR merged)
- Phase 2 complete (group_vars configured, PR merged)
- Roles referenced by the playbook must already exist under `roles/`

**Carry forward from prior phases**: The agent already knows the group name, environment, hostnames, inventory structure, group vars, and vault configuration from the router and Phases 1–2. Use these values directly.

---

## Step 1: Playbook

**Decide** (all questions upfront):
1. "Which roles should be applied to this group?" (e.g., `common`, `nginx`, `app-deploy`)
2. "Does the playbook need privilege escalation (become)?" (default: yes)
3. "Are there any pre-tasks to run before roles?" (e.g., update apt cache, check connectivity)
4. "Are there any post-tasks or handlers?" (e.g., restart a service, send a notification)

**Inspect**:
- Check if a playbook already exists: search `playbooks/{group-name}.yml`
- Verify each referenced role exists: search `roles/{role-name}/` for each role

| Status | Action |
|--------|--------|
| Playbook already exists | Show existing playbook, ask if update is needed |
| Referenced role missing | Warn the user. List missing roles and ask whether to proceed without them or stop. |
| Ready to create | Continue with generation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

**Generate** (with pre-tasks and handlers):

```yaml
---
- name: Configure {group-name}
  hosts: {group-name}
  become: {become}
  gather_facts: true

  pre_tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

  roles:
    - role: {role-1}
    - role: {role-2}
    - role: {role-3}

  handlers:
    - name: Restart {service-name}
      ansible.builtin.service:
        name: {service-name}
        state: restarted
```

**Generate** (minimal -- no pre-tasks or handlers):

```yaml
---
- name: Configure {group-name}
  hosts: {group-name}
  become: {become}
  gather_facts: true

  roles:
    - role: {role-1}
    - role: {role-2}
```

Write to: `playbooks/{group-name}.yml`

---

## Validate (optional)

```bash
yamllint playbooks/{group-name}.yml
ansible-lint playbooks/{group-name}.yml
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, no lint errors | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or quoting |
| Lint warning (e.g., FQCN) | Use fully-qualified collection names (e.g., `ansible.builtin.apt` not `apt`) |
| Lint error (e.g., missing role) | Verify role exists under `roles/` or is listed in `requirements.yml` |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Inventory] Add {group-name} playbook -- Phase 3`

**Files to include**:
- `playbooks/{group-name}.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, the setup is complete. The workflow will detect the merged files and confirm the host group is fully configured.

---

## Setup Complete

The host group is now fully configured. After this PR merges, you can run the playbook against the environment:

```bash
ansible-playbook -i inventory/{environment}/ playbooks/{group-name}.yml
```

| Component | Location |
|-----------|----------|
| Inventory | `inventory/{environment}/hosts.yml` |
| Group variables | `inventory/{environment}/group_vars/{group-name}.yml` |
| Vault secrets | `inventory/{environment}/group_vars/{group-name}/vault.yml` |
| Playbook | `playbooks/{group-name}.yml` |
