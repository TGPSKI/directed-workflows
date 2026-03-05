---
name: onboard-service-phase-3
description: "Phase 3: Exposure -- Ingress and NetworkPolicy for external HTTP access (skippable)"
metadata:
  author: platform-team
  version: "2.0"
parent: onboard-service.md
---

# Phase 3: Exposure

Configure external HTTP access via Ingress and a NetworkPolicy for the ingress controller.

**This phase is skippable.** Internal-only services should proceed directly to Phase 4.

**Typical PR**: 1
**Files produced**: ingress.yml, networkpolicy-ingress.yml

## Prerequisites

- Phase 2 complete (Deployment and Service files merged)
- DNS record or ability to create one for the desired hostname

**Carry forward from prior phases**: The agent already knows the service name, namespace name, environment, container port, and Service configuration from the router and Phases 1–2. Use these values directly.

---

## Should This Phase Run?

**Decide**: "Does this service need external HTTP access?"

| Answer | Action |
|--------|--------|
| No -- internal only | Skip this phase. Continue with `@onboard-service/phase-4-observability.md` |
| Yes -- external HTTP | Continue below |

---

## Step 1: Ingress

**Decide**:
1. "What hostname should route to this service?" (e.g., `payment-gateway.example.com`)
2. "How should TLS be handled?" (cert-manager / custom certificate / none for dev)

**Inspect**:
- Verify the Service file from Phase 2 exists: search `apps/{service-name}/service.yml`
- Check for existing Ingress file: search `apps/{service-name}/ingress.yml`

| Status | Action |
|--------|--------|
| Service file missing | Stop. Phase 2 must be completed first. |
| Ingress file already exists | Show existing config, ask if update is needed |
| Ready to create | Continue with generation |

**Generate** (cert-manager TLS):

Search for existing files matching this pattern in the repo before generating from scratch.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {service-name}
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/name: {service-name}
    app.kubernetes.io/part-of: {service-name}
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - {hostname}
      secretName: {service-name}-tls
  rules:
    - host: {hostname}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {service-name}
                port:
                  number: {port}
```

Write to: `apps/{service-name}/ingress.yml`

---

## Step 2: NetworkPolicy

No additional questions needed -- the policy allows traffic from the ingress controller namespace.

**Generate**:

Search for existing NetworkPolicy files in the repo before generating from scratch.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: {namespace-name}
  labels:
    app.kubernetes.io/part-of: {service-name}
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {service-name}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
  policyTypes:
    - Ingress
```

Write to: `apps/{service-name}/networkpolicy-ingress.yml`

---

## Validate (optional)

```bash
yamllint apps/{service-name}/ingress.yml apps/{service-name}/networkpolicy-ingress.yml
kubeconform apps/{service-name}/ingress.yml apps/{service-name}/networkpolicy-ingress.yml
# Optional -- CI/CD is the actual gate
```

| Validation Result | Action |
|-------------------|--------|
| Valid YAML, passes schema check | Continue to PR Checkpoint |
| YAML syntax error | Fix indentation or structure |
| Schema validation error | Verify `ingressClassName`, `tls`, and `rules` fields |

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Onboarding] Add {service-name} Ingress and NetworkPolicy -- Phase 3`

**Files to include**:
- `apps/{service-name}/ingress.yml`
- `apps/{service-name}/networkpolicy-ingress.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

Continue with observability:

`@onboard-service/phase-4-observability.md`
