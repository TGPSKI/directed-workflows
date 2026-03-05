---
name: onboard-service-phase-4
description: "Phase 4: Observability -- ServiceMonitor for Prometheus metrics scraping"
metadata:
  author: platform-team
  version: "2.0"
parent: onboard-service.md
---

# Phase 4: Observability

Add monitoring configuration so Prometheus scrapes your service's metrics.

**Typical PR**: 1
**Files produced**: servicemonitor.yml

## Prerequisites

- Phase 2 complete (Deployment and Service files merged)
- Phase 3 complete or skipped
- Application exposes a Prometheus-compatible metrics endpoint

**Carry forward from prior phases**: The agent already knows the service name, namespace name, port configuration, and Service labels from the router and Phases 1–3. Use these values directly.

---

## Step 1: ServiceMonitor

**Decide**:
1. "What path does your application expose metrics on?" (default: `/metrics`)
2. "What port name or number serves metrics?" (default: same as application port)
3. "What scrape interval?" (default: `30s`)

**Inspect**:
- Check for existing ServiceMonitor in the repo: search `monitoring/{service-name}/`
- Verify the Service file from Phase 2 exists: search `apps/{service-name}/service.yml`

| Status | Action |
|--------|--------|
| ServiceMonitor already exists | Show existing config, ask if update is needed |
| Service file missing | Stop. Phase 2 must be completed first. |
| No metrics endpoint | Skip this phase. Onboarding is complete without monitoring. |
| Ready to create | Continue with generation |

**Generate**:

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {service-name}
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/name: {service-name}
    app.kubernetes.io/part-of: {service-name}
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: {service-name}
  endpoints:
    - port: {metrics-port}
      path: {metrics-path}
      interval: {scrape-interval}
```

Write to: `monitoring/{service-name}/servicemonitor.yml`

---

## Validate (optional)

Verify the generated YAML is well-formed:

```bash
yamllint monitoring/{service-name}/servicemonitor.yml
kubeconform monitoring/{service-name}/servicemonitor.yml
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, passes schema check | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or structure |
| Schema validation error | Verify `apiVersion` and `spec` fields match ServiceMonitor CRD |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Onboarding] Add {service-name} monitoring -- Phase 4`

**Files to include**:
- `monitoring/{service-name}/servicemonitor.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Onboarding Complete

The service configuration is now complete. After this PR merges, your CI/CD pipeline will validate and deploy the resources.

| Component | Location |
|-----------|----------|
| Namespace + RBAC | `namespaces/{service-name}/` |
| Deployment + Service | `apps/{service-name}/` |
| Ingress (if applicable) | `apps/{service-name}/ingress.yml` |
| Monitoring | `monitoring/{service-name}/` |
