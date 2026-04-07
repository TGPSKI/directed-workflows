# The Multi-Phase Router Pattern

A single-file workflow works for focused tasks that complete in one session. But real-world processes -- onboarding a service, setting up a new environment, migrating to a new deployment model -- span weeks, involve multiple pull requests, and require picking up where you left off.

The single-file approach breaks down:

- **Context window limits**: A multi-step onboarding process with templates, validation steps, and branching logic exceeds what an agent can hold in context at once.
- **No resumability**: Close your IDE and come back tomorrow -- the agent has no idea what you already completed.
- **All-or-nothing execution**: You can't skip to the monitoring step or run just the RBAC setup independently.
- **PR sprawl**: A single workflow that generates 15+ files encourages monolithic PRs that are impossible to review.

## The Pattern

The Multi-Phase Router splits a workflow into a router and phase modules:

```
.agents/skills/onboard-service/
├── SKILL.md                           # Router (entry point)
└── references/
    ├── phase-01-foundation.md         # Phase module
    ├── phase-02-workload.md           # Phase module
    ├── phase-03-exposure.md           # Phase module (skippable)
    └── phase-04-observability.md      # Phase module
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

The router never generates files itself. Pure orchestration.

### Phase Modules

Each phase is a self-contained workflow file in the `references/` subdirectory that:

- **Declares its parent** via frontmatter (`parent: onboard-service` -- the router's `name` field)
- **Declares carry-forward data** -- what it inherits from prior phases so the agent doesn't re-ask
- **Covers a logical grouping** of steps (a coherent unit of work, not a single step)
- **Ends with a PR checkpoint** specifying exactly which files to include
- **Links to the next phase**

Each phase is independently invocable. A user who already has their service deployed can jump straight to `references/phase-04-observability.md` without touching the router.

### Progress Detection

This is what makes resumability work. The router maps **repository state** to **workflow phase**:

```markdown
| Detected State | Recommended Action |
|----------------|-------------------|
| No service directory | Start at Phase 1 |
| Namespace exists, no Deployment | Continue at Phase 2 |
| Deployment exists, no Ingress | Continue at Phase 3 |
| ServiceMonitor exists | Onboarding complete |
```

The agent checks the primary branch, matches against this table, and proposes where to resume. No session state, no database, no external tracking.

## Codebase-as-State

This is the key idea. Traditional workflow tools track progress in a database or session store. When the session dies, progress is lost.

The router pattern inverts this: progress is detected from artifacts that already exist in the repository. If `namespace.yml` exists but `deployment.yml` doesn't, you're at Phase 2. It doesn't matter if you completed Phase 1 yesterday, last week, or in a different IDE.

**The source of truth is the upstream primary branch.** Sync your fork and rebase (or pull) before each session. Local uncommitted files, unmerged feature branches, and content recalled from a prior LLM session are all excluded. Files generated in a previous session that haven't been merged via PR don't count as completed work.

The PR checkpoint at the end of each phase is the mechanism that advances the state machine -- until a phase's PR merges, that phase isn't done.

The detection is also **self-healing**. If someone manually creates a file outside the workflow, the router picks up the correct state. If a PR gets reverted, the router detects the regression.

## Structural Conventions

### Frontmatter

```yaml
---
name: onboard-service            # Unique identifier (router)
description: "..."               # Human-readable purpose
metadata:
  author: team-name
  version: "1.0"
compatibility: "..."             # Tool/dependency requirements
---
```

Phase files reference their router by name, not filename:

```yaml
---
name: onboard-service-phase-01
description: "..."
parent: onboard-service          # Matches the router's name field
---
```

This lets the agent trace any phase back to its router unambiguously, even when every router is named `SKILL.md`.

### Inspect-Decide-Generate

Each step follows a consistent cycle:

1. **Inspect** -- Read the repo: schemas, existing examples, prior phase outputs, carry-forward data. Derive everything you can before involving the user.
2. **Decide** -- Present the user with options and defaults. Ask only for decisions that can't be derived from the repo or prior phases.
3. **Generate** -- Produce config using existing files as reference. Prefer schemas, examples, and prior configurations over hard-coded templates.

### Status-Action Tables

Conditional logic is encoded as tables, not prose:

```markdown
| Status | Action |
|--------|--------|
| Namespace exists | Show existing config, skip to RBAC verification |
| Namespace missing | Continue with namespace creation |
```

An agent reading "if the namespace exists, you might want to skip ahead, unless the labels are wrong" will sometimes get confused. A two-row table does not leave room for interpretation.

### PR Checkpoints

Each phase ends with an explicit checkpoint:

```markdown
## PR Checkpoint

Title: `[Onboarding] Add {service-name} monitoring - Phase 4`

**Files to include**:
- ServiceMonitor
- PrometheusRules
- PrometheusRule tests
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

Keep it short (under 150 lines). Start with a Design Principles section defining the behavioral contract: who owns the configuration, where the source of truth lives, what the agent should derive vs ask, and whether to default to simple or complex. Then collect one identifier, run detection, route. No generation logic.

### 4. Write Phase Modules

Place phase files in the `references/` subdirectory. Each module should:
- Declare what data it carries forward from prior phases
- Be independently invocable
- Derive values from the repo before asking the user
- Prefer referencing existing files over hard-coded templates
- Include local validation (`yamllint`, `kubeconform`, `terraform validate`, etc.)
- End with a PR checkpoint -- STOP, do not proceed until merged
- Link to the next phase

### 5. Enable IDE Discovery

Create symlinks from IDE-specific directories to the canonical location:

```
.cursor/skills/onboard-service → ../../.agents/skills/onboard-service
.claude/skills/onboard-service → ../../.agents/skills/onboard-service
```

One source of truth in `.agents/skills/`, multiple IDE entry points.

## When to Use This

The pattern works for any system where:

1. **Configuration is file-based** -- YAML, JSON, HCL, or any structured format in a git repo
2. **Files follow placement conventions** -- predictable directory structures enable progress detection
3. **The process spans multiple sessions** -- if it completes in 15 minutes, use a single-file workflow instead

## Comparison to Alternatives

| Approach | Resumable | Modular | Fits Context | Small PRs |
|----------|-----------|---------|--------------|-----------|
| Single monolithic file | No | No | No | No |
| Numbered step files (no router) | Manual | Yes | Yes | Maybe |
| External state tracking (DB, issue tracker) | Yes | Depends | Depends | Depends |
| Backstage scaffolder | No | Partial | N/A | No |
| **Multi-Phase Router** | **Yes** | **Yes** | **Yes** | **Yes** |

Numbered steps without a router get close, but without progress detection the user must manually remember where they left off. External state tracking adds infrastructure dependencies and breaks when the state store disagrees with reality.
