---
name: add-host-group-phase-2
description: "Phase 2: Group Variables -- Plain-text config and vault-encrypted secrets"
metadata:
  author: platform-team
  version: "2.0"
parent: add-host-group.md
---

# Phase 2: Group Variables

Configure group-level variables for the host group and set up vault-encrypted secrets.

**Typical PR**: 1
**Files produced**: {group-name}.yml, {group-name}/vault.yml

## Prerequisites

- Phase 1 complete (hosts added to inventory and PR merged)

---

## Step 1: Group Variables

**Decide** (all questions upfront):
1. "What application port does this group serve?" (e.g., `8080`, `443`)
2. "What log level should be configured?" (debug / info / warn / error, default: `info`)
3. "Are there any additional environment-specific variables?" (e.g., `app_version`, `feature_flags`, `max_connections`)

**Inspect**:
- Verify the group exists in inventory: search `inventory/{environment}/hosts.yml` for `{group-name}`
- Check for existing group_vars: search `inventory/{environment}/group_vars/{group-name}.yml`

| Status | Action |
|--------|--------|
| Group not in inventory | Stop. Phase 1 must be completed first. |
| Group vars file already exists | Show existing vars, ask if update is needed |
| Ready to create | Continue with generation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
---
app_port: {app-port}
log_level: {log-level}

# Environment-specific configuration
{additional-var-1}: {value-1}
{additional-var-2}: {value-2}
```

Write to: `inventory/{environment}/group_vars/{group-name}.yml`

---

## Step 2: Vault-Encrypted Secrets

**Decide**:
1. "Does this group need any secrets?" (e.g., API keys, database passwords, TLS certificates)
2. If yes: "What secret variables are needed?" (e.g., `db_password`, `api_key`, `tls_cert`)

| Answer | Action |
|--------|--------|
| No secrets needed | Skip to Validate (optional) |
| Yes, secrets needed | Continue below |

**Inspect**:
- Check for existing vault file: search `inventory/{environment}/group_vars/{group-name}/vault.yml`

| Status | Action |
|--------|--------|
| Vault file already exists | Show existing structure (encrypted), ask if update is needed |
| Ready to create | Continue with generation |

**Generate** (plaintext structure -- user encrypts after):

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
---
vault_{secret-var-1}: "REPLACE_WITH_ACTUAL_VALUE"
vault_{secret-var-2}: "REPLACE_WITH_ACTUAL_VALUE"
```

Write to: `inventory/{environment}/group_vars/{group-name}/vault.yml`

Then reference the vault variables from the plain-text group_vars file:

```yaml
{secret-var-1}: "{{ vault_{secret-var-1} }}"
{secret-var-2}: "{{ vault_{secret-var-2} }}"
```

Append to: `inventory/{environment}/group_vars/{group-name}.yml`

**Instruct the user**:

> Before committing, encrypt the vault file:
> ```bash
> ansible-vault encrypt inventory/{environment}/group_vars/{group-name}/vault.yml
> ```
> Never commit plaintext secrets. The PR should contain the encrypted vault file only.

---

## Validate (optional)

```bash
yamllint inventory/{environment}/group_vars/{group-name}.yml
# Optional -- CI/CD is the actual gate
```

If vault file was created, verify it's encrypted (the file should start with `$ANSIBLE_VAULT;`).

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, vault encrypted | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or quoting |
| Vault file is plaintext | Run `ansible-vault encrypt` before committing |

## PR Checkpoint

**Title**: `[Inventory] Add {group-name} group variables -- Phase 2`

**Files to include**:
- `inventory/{environment}/group_vars/{group-name}.yml`
- `inventory/{environment}/group_vars/{group-name}/vault.yml` (encrypted)

---

## Next Phase

Continue with the playbook:

`@add-host-group/phase-3-playbook.md`
