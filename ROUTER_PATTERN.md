# The Multi-Phase Router Pattern

A single-file workflow works for focused tasks that complete in one session. But real-world processes -- onboarding a service, setting up a new environment, migrating to a new deployment model -- span weeks, involve multiple pull requests, and require picking up where you left off.

The single-file approach breaks down at scale:

- **Context window limits**: A multi-step onboarding process with YAML templates, validation steps, and branching logic exceeds what an agent can hold in context at once.
- **No resumability**: Close your IDE and come back tomorrow -- the agent has no idea what you already completed.
- **All-or-nothing execution**: You can't skip to the monitoring step or run just the RBAC setup independently.
- **Pull request sprawl**: A single workflow that generates 15+ files encourages monolithic PRs that are impossible to review.

## The Pattern

The Multi-Phase Router splits a directed workflow into three structural components:

```
.agents/commands/
├── onboard-service.md                 # Router (entry point)
└── onboard-service/
    ├── phase-1-foundation.md          # Phase module
    ├── phase-2-workload.md            # Phase module
    ├── phase-3-exposure.md            # Phase module (skippable)
    └── phase-4-observability.md       # Phase module
```

### The Router

The single entry point. It does four things:

1. **Defines design principles** -- the behavioral contract (ownership, source of truth, ask less/infer more, prefer simple)
2. **Collects a key identifier** (service name, project name, etc.)
3. **Detects current progress** by inspecting the repository
4. **Routes to the correct phase** based on what it finds

```markdown
## Entry Point

**Ask**: "What is the name of your service?"

## Detect Current Progress

| Check | Files to Search | Status Indicator |
|-------|-----------------|------------------|
| Namespace exists | `namespaces/{service}/namespace.yml` | Phase 1 complete |
| Deployment exists | `apps/{service}/deployment.yml` | Phase 2 complete |
| Ingress exists | `apps/{service}/ingress.yml` | Phase 3 complete |
| ServiceMonitor exists | `monitoring/{service}/` | Phase 4 complete |

## Route to Phase

**Ask**: "Based on my analysis, you appear to be at [detected phase]. Would you like to:
1. Continue from [detected phase]
2. Start from a different phase
3. Review what's already configured"
```

The router never generates files itself. It's pure orchestration.

### Phase Modules

Each phase is a self-contained workflow file that:

- **Declares its parent** via frontmatter (`parent: onboard-service.md`)
- **Declares carry-forward data** -- what it inherits from prior phases (identifiers, paths, environment names) so the agent doesn't re-ask
- **Covers a logical grouping** of steps (a coherent unit of work, not a single step)
- **Ends with a PR checkpoint** specifying exactly which files to include
- **Links to the next phase** so the agent can continue if the user wants

Each phase is independently invocable. A user who already has their service deployed can jump straight to `@phase-4-observability.md` without touching the router.

### The Progress Detection Table

This is what makes resumability work. The router maps **repository state** to **workflow phase**:

```markdown
| Detected State | Recommended Action |
|----------------|-------------------|
| No service directory | Start at Phase 1 |
| Namespace exists, no Deployment | Continue at Phase 2 |
| Deployment exists, no Ingress | Continue at Phase 3 |
| ServiceMonitor exists | Onboarding complete |
```

The agent checks the primary branch, matches against this table, and proposes where to resume. No session state, no database, no external tracking -- the merged default branch *is* the state.

## Why This Works

### Codebase-as-State

Traditional workflow tools track progress in a database or session store. When the session dies, progress is lost. The router pattern inverts this: progress is detected from artifacts that already exist in the repository. If `namespace.yml` exists but `deployment.yml` doesn't, you're at Phase 2. It doesn't matter if you completed Phase 1 yesterday, last week, or in a different IDE.

**The source of truth is the upstream primary branch.** Progress detection must run against an up-to-date view of the default branch -- sync your fork and rebase, or pull if working on a direct clone. Local uncommitted files, unmerged feature branches, and content recalled from a prior LLM session are all excluded. Files generated in a previous session that haven't been merged via PR don't count as completed work. This is what makes the pattern durable: it's immune to stale local state, abandoned sessions, and diverged forks. The PR checkpoint at the end of each phase is the mechanism that advances the state machine -- until a phase's PR merges, that phase isn't done.

The detection is also **self-healing**. If someone manually creates a file outside the workflow, the router picks up the correct state. If a PR gets reverted, the router detects the regression.

### Context Window Efficiency

A single phase file is typically 200-400 lines -- well within any model's effective context window. The agent loads only the phase it needs, keeping templates, validation steps, and branching logic in focus. Compare this to a monolithic 2000-line workflow where the agent is working from a summary of instructions it read 1500 lines ago.

### PR Checkpoints

Each phase ends with an explicit checkpoint:

```markdown
## PR Checkpoint

Title: `[Onboarding] Add {service-name} monitoring - Phase 4`

**Files to include**:
- ServiceMonitor
- PrometheusRules
- PrometheusRule tests
- Updated monitoring namespace reference
- NetworkPolicy for metrics scraping
```

Phase boundaries are drawn where the work forms a coherent, independently reviewable unit.

### Conditional Phase Skipping

Not every workflow is linear. The router supports variant paths:

```markdown
| Service Type | Phases Needed |
|--------------|---------------|
| Internal service (no HTTP) | 1 → 2 → 4 |
| Full production service | 1 → 2 → 3 → 4 |
| CronJob / batch workload | 1 → 2 → 4 (modified) |
```

## Structural Conventions

### Frontmatter Schema

```yaml
---
name: workflow-name           # Unique identifier
description: "..."            # Human-readable purpose
metadata:
  author: team-name
  version: "1.0"
parent: parent-workflow.md    # Only on phase files
compatibility: "..."          # Tool/dependency requirements
---
```

The `parent` field creates a navigable hierarchy. An agent encountering a phase file in isolation can trace back to the router.

### Inspect-Decide-Generate Cycle

Each step follows a consistent pattern:

1. **Inspect** -- Read the repo: schemas, existing examples, prior phase outputs, carry-forward data. Derive everything you can before involving the user.
2. **Decide** -- Present the user with options and defaults. Ask only for decisions and information that can't be derived from the repo or prior phases.
3. **Generate** -- Produce the config using existing files as reference. Prefer schemas, examples, and prior configurations over hard-coded templates.

The workflow produces files. CI/CD pipelines validate and deploy them.

### Status-Action Tables

Conditional logic is encoded as tables, not prose:

```markdown
| Status | Action |
|--------|--------|
| Namespace exists | Show existing config, skip to RBAC verification |
| Namespace missing | Continue with namespace creation |
```

Tables are unambiguous. An agent reading "if the namespace exists, you might want to skip ahead, unless the labels are wrong" will sometimes get confused. A two-row table does not leave room for interpretation.

## Implementing the Pattern

### 1. Identify Phase Boundaries

Map your process to logical units that each produce a reviewable PR:

- **Foundation** -- Identity, namespace, permissions, quotas
- **Workload** -- Deployment, Service, ConfigMap, Secrets
- **Exposure** -- Ingress/Gateway, TLS, DNS
- **Observability** -- Monitoring, alerting, SLOs, dashboards
- **Production** -- Promotion, handoff, operational readiness

### 2. Define Progress Indicators

For each phase, identify a file whose existence proves the phase is complete. Good indicators are files that are always created (not optional), have distinctive paths, and are created last in the phase.

### 3. Write the Router

Keep it short (under 150 lines). Start with a Design Principles section that defines the behavioral contract: who owns the configuration, where the source of truth lives, what the agent should derive vs ask, and whether to default to simple or complex. Then collect one identifier, run detection, route. No generation logic.

### 4. Write Phase Modules

Each module should declare what data it carries forward from prior phases, be independently invocable, derive values from the repo before asking the user, prefer referencing existing files over hard-coded templates, include local validation commands (`yamllint`, `kubeconform`, `terraform validate`, etc.), end with a PR checkpoint (STOP -- do not proceed until merged), and link to the next phase.

### 5. Add Tool Integration

Create symlinks from tool-specific directories to the canonical files:

```
.cursor/commands/onboard-service.md → ../../.agents/commands/onboard-service.md
.claude/commands/onboard-service.md → ../../.agents/commands/onboard-service.md
```

One source of truth, multiple AI IDE entry points.

## Comparison to Alternatives

| Approach | Resumable | Modular | Fits Context | Small PRs |
|----------|-----------|---------|--------------|-----------|
| Single monolithic file | No | No | No | No |
| Numbered step files (no router) | Manual | Yes | Yes | Maybe |
| External state tracking (DB, issue tracker) | Yes | Depends | Depends | Depends |
| Backstage scaffolder | No | Partial | N/A | No |
| **Multi-Phase Router** | **Yes** | **Yes** | **Yes** | **Yes** |

Numbered steps without a router get close, but without progress detection the user must manually remember where they left off. External state tracking adds infrastructure dependencies and breaks when the state store disagrees with reality. Scaffolders are one-shot and can't resume.

## Applying This to Your Platform

The pattern works for any system where:

1. **Configuration is file-based** -- YAML, JSON, HCL, or any structured format in a git repo
2. **Files follow placement conventions** -- Predictable directory structures enable progress detection
3. **A local validation toolchain exists** -- `yamllint`, `terraform validate`, `helm lint`, `ansible-lint`, or equivalent
4. **The process spans multiple sessions** -- If it completes in 15 minutes, use a single-file workflow instead

| Platform | Example Process | Progress Indicators |
|----------|----------------|---------------------|
| **Kubernetes** | Service setup: namespace, RBAC, deployment, ingress, monitoring | Directory per service, resource files |
| **Terraform** | Account provisioning: provider config, state backend, IAM baseline | `.tf` files per module |
| **Ansible** | Inventory management: hosts, group_vars, playbooks | Inventory entries, group_vars files |
| **ArgoCD** | Application setup: AppProject, sync policies, notifications | Application YAML in apps/ directory |
| **Crossplane** | Resource authoring: XRD, Composition, Claim, ProviderConfig | CRD files per resource type |
| **Helm** | Chart scaffolding: values per env, CI pipeline, release | `Chart.yaml`, `values-{env}.yaml` |
