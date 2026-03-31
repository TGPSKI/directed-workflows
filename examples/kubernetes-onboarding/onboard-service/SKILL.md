---
name: onboard-service
description: Onboard a new service to a Kubernetes platform. Detects current onboarding progress and routes to the appropriate phase.
metadata:
  author: platform-team
  version: "2.0"
compatibility: Works with any AI IDE that supports file references. No cluster access required -- the workflow produces config files validated locally.
---

# Onboard Service

Walk through the complete process of onboarding a new service to the Kubernetes platform. This workflow produces small, focused pull requests -- one per phase.

## Design Principles

- **Ownership**: The service team creates the configuration; the platform team reviews and merges. The agent assists the team but does not bypass review.
- **Source of truth**: Use existing namespace, RBAC, and workload patterns in the repo. Search for similar services before generating from scratch.
- **Ask less, infer more**: Derive namespace names from the service name (e.g., `{service-name}` or `{service-name}-staging`). Verify state from the filesystem before asking the user.
- **Prefer simple**: Skip Ingress for internal-only services. Skip NetworkPolicy when the default policy allows. Default to the minimal viable configuration.

## Prerequisites

- **Start from an up-to-date primary branch.** Sync your fork and rebase onto the latest upstream `main` before invoking this workflow (or `git pull` if working on a direct clone). Progress detection relies on files merged to the upstream primary branch -- not local commits, uncommitted changes, or prior session output. If you generated files in a previous session that haven't been merged yet, they don't count as completed work.
- **Start with a clean chat session.** Prior conversation history causes the agent to conflate previous output with actual repository state. If you have an existing session about this service, start a new one.

## Entry Point

**Ask**: "What is the name of your service? (used for naming resources, e.g., `payment-gateway`)"

## Detect Current Progress

After getting the service name, inspect the repository to determine what's already been completed. **Detection must be based on files merged to the primary branch** -- not on local uncommitted files, unmerged feature branches, or content from prior chat sessions. If a file was generated in a previous session but its PR hasn't merged, that phase is not complete.

### Progress Detection

| Check | What to Look For | Indicates |
|-------|-----------------|-----------|
| Namespace | `namespaces/{service}/namespace.yml` | Phase 1 complete |
| Deployment | `apps/{service}/deployment.yml` | Phase 2 complete |
| Ingress | `apps/{service}/ingress.yml` | Phase 3 complete (or skipped) |
| ServiceMonitor | `monitoring/{service}/servicemonitor.yml` | Phase 4 complete |

### Determine Phase

| Detected State | Recommended Action |
|----------------|-------------------|
| No service directory | Start at **Phase 1** |
| Namespace exists, no Deployment | Continue at **Phase 2** |
| Deployment exists, no Ingress or ServiceMonitor | Continue at **Phase 3** or **Phase 4** |
| ServiceMonitor exists | Onboarding complete |

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

| Service Type | Phases |
|--------------|--------|
| Internal service (no HTTP) | 1 → 2 → 4 |
| External service (HTTP) | 1 → 2 → 3 → 4 |

## Phase Files

| Phase | File | Description | PR |
|-------|------|-------------|-----|
| 1 | `@onboard-service/phases/01-foundation.md` | Namespace, RBAC | 1 |
| 2 | `@onboard-service/phases/02-workload.md` | Deployment, Service | 1 |
| 3 | `@onboard-service/phases/03-exposure.md` | Ingress, NetworkPolicy (skippable) | 1 |
| 4 | `@onboard-service/phases/04-observability.md` | ServiceMonitor for metrics scraping | 1 |
