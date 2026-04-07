---
name: onboard-service-phase-2
description: "Phase 2: Workload -- Deployment and Service for a new service"
metadata:
  author: platform-team
  version: "2.0"
parent: onboard-service
---

# Phase 2: Workload

Create the Deployment and Service for your application.

**Typical PR**: 1
**Files produced**: deployment.yml, service.yml

## Prerequisites

- Phase 1 complete (namespace exists and PR merged)
- Container image built and pushed to a registry

**Carry forward from prior phases**: The agent already knows the service name, namespace name, and environment from the router and Phase 1. Use these values directly.

---

## Step 1: Deployment

**Decide**:
1. "What is the container image?" (e.g., `ghcr.io/acme/payment-gateway:latest`)
2. "What port does your application listen on?"
3. "How many replicas for this environment?" (default: 2)
4. "Does your app expose a health check endpoint?" (e.g., `/healthz`, `/ready`)

**Inspect**:
- Verify the namespace file from Phase 1 exists: search `namespaces/{service-name}/namespace.yml`
- Check for existing Deployment file: search `apps/{service-name}/deployment.yml`

| Status | Action |
|--------|--------|
| Namespace file missing | Stop. Phase 1 must be completed first. |
| Deployment file already exists | Show existing spec, ask if it should be updated or skipped |
| No existing Deployment file | Continue with creation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {service-name}
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/name: {service-name}
    app.kubernetes.io/part-of: {service-name}
spec:
  replicas: {replicas}
  selector:
    matchLabels:
      app.kubernetes.io/name: {service-name}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {service-name}
        app.kubernetes.io/part-of: {service-name}
    spec:
      containers:
        - name: {service-name}
          image: {container-image}
          ports:
            - containerPort: {port}
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: {health-endpoint}
              port: {port}
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: {health-endpoint}
              port: {port}
            initialDelaySeconds: 5
            periodSeconds: 10
```

Write to: `apps/{service-name}/deployment.yml`

---

## Step 2: Service

No additional questions needed -- the Service is derived from the Deployment.

**Inspect**:
- Check for existing Service file: search `apps/{service-name}/service.yml`

| Status | Action |
|--------|--------|
| Service file exists | Show existing spec, confirm ports match |
| Service file missing | Continue with creation |

**Generate**:

Search for existing Service files in the repo before generating from scratch.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {service-name}
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/name: {service-name}
    app.kubernetes.io/part-of: {service-name}
spec:
  selector:
    app.kubernetes.io/name: {service-name}
  ports:
    - port: {port}
      targetPort: {port}
      protocol: TCP
  type: ClusterIP
```

Write to: `apps/{service-name}/service.yml`

---

## Validate (optional)

```bash
yamllint apps/{service-name}/
kubeconform apps/{service-name}/
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, passes schema check | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or structure |
| Schema validation error | Verify `apiVersion`, `spec.selector`, and `spec.template` fields |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Onboarding] Add {service-name} Deployment and Service -- Phase 2`

**Files to include**:
- `apps/{service-name}/deployment.yml`
- `apps/{service-name}/service.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

If the service needs external HTTP access, continue with exposure:

`@onboard-service/references/03-exposure.md`

If the service is internal only, skip to observability:

`@onboard-service/references/04-observability.md`
