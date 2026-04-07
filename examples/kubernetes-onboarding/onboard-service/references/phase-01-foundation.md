---
name: onboard-service-phase-1
description: "Phase 1: Foundation -- Namespace and RBAC setup for a new service"
metadata:
  author: platform-team
  version: "2.0"
parent: onboard-service
---

# Phase 1: Foundation

Create the namespace and RBAC bindings for a new service.

**Typical PR**: 1
**Files produced**: namespace.yml, resourcequota.yml, rolebinding.yml

## Prerequisites

- Phase 0: Service name determined via the router

**Carry forward from prior phases**: The agent already knows the service name from the router. Use it directly for namespace naming and file paths.

---

## Step 1: Create Namespace

**Decide**:
1. "What namespace name should be used?" (e.g., `{service-name}`, `{service-name}-staging`)
2. "What environment is this for?" (staging / production)
3. "What cost-center or team label should be applied?"

**Inspect**:
- Check if a namespace file already exists in the repo: search `namespaces/{service}/`

| Status | Action |
|--------|--------|
| Namespace file exists | Show existing config, skip to Step 2 |
| Namespace file missing | Continue with creation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: {namespace-name}
  labels:
    app.kubernetes.io/part-of: {service-name}
    environment: {environment}
    team: {team-label}
  annotations:
    description: "{service-name} {environment} namespace"
```

**Generate** ResourceQuota:

Search for existing ResourceQuota files in the repo before generating from scratch.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {service-name}-quota
  namespace: {namespace-name}
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

Write files to:
- `namespaces/{service-name}/namespace.yml`
- `namespaces/{service-name}/resourcequota.yml`

---

## Step 2: RBAC

**Decide**:
1. "What is the team or group name that should have access?" (e.g., `payments-team`)
2. "What level of access?" (edit / view / admin)

| Access Level | ClusterRole to Bind |
|-------------|-------------------|
| view | `view` |
| edit | `edit` |
| admin | `admin` |

**Inspect**:
- Check for existing RoleBinding file: search `namespaces/{service-name}/rolebinding.yml`

| Status | Action |
|--------|--------|
| RoleBinding file exists | Show existing binding, confirm it's correct |
| No existing RoleBinding file | Continue with creation |

**Generate**:

Search for existing RoleBinding files in the repo before generating from scratch.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {team-name}-{access-level}
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/part-of: {service-name}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {access-level}
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: {team-name}
```

Write to: `namespaces/{service-name}/rolebinding.yml`

---

## Validate (optional)

```bash
yamllint namespaces/{service-name}/
kubeconform namespaces/{service-name}/
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, passes schema check | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or structure |
| Schema validation error | Verify `apiVersion` and required fields |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Onboarding] Add {service-name} namespace and RBAC -- Phase 1`

**Files to include**:
- `namespaces/{service-name}/namespace.yml`
- `namespaces/{service-name}/resourcequota.yml`
- `namespaces/{service-name}/rolebinding.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with workload deployment:

`references/phase-02-workload.md`
