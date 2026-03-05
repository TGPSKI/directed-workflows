# Directed Workflows -- Agent Instructions

You are working with the Directed Workflows pattern repository. Your job is to help the user create directed workflows for **their** codebase -- not to modify this repository.

## What This Repo Is

This repo contains the pattern definition, examples, and templates for directed workflows: structured markdown files that AI agents execute interactively to walk users through multi-step configuration processes.

## Your Task

When the user asks you to create a directed workflow, you will:

1. **Understand their process** -- what multi-step task they want to encode
2. **Analyze their target repo** -- file structure, naming conventions, validation tools
3. **Generate workflow files** -- in their repo, following the conventions below

Output goes in the **user's repo**, not here. This repo is reference material.

---

## The Pattern

### Single-File Workflow

Use when the process completes in one session and one PR.

**Structure:**

```
{user-repo}/.agents/commands/
└── {workflow-name}.md
```

### Multi-Phase Router

Use when the process spans multiple sessions or produces multiple PRs.

**Structure:**

```
{user-repo}/.agents/commands/
├── {workflow-name}.md              # Router (entry point, no generation)
└── {workflow-name}/
    ├── phase-1-{name}.md           # Phase module
    ├── phase-2-{name}.md           # Phase module
    └── phase-3-{name}.md           # Phase module
```

**When to use which:**

| Characteristic | Single-file | Multi-phase router |
|---------------|-------------|-------------------|
| Completes in one session | Yes | Overkill |
| Spans days/weeks | No | Yes |
| Produces 1 PR | Yes | No -- one PR per phase |
| Has 5+ steps with branching | Possible but fragile | Yes |

---

## Conventions (follow exactly)

### Frontmatter

Every workflow file starts with YAML frontmatter:

```yaml
---
name: workflow-name
description: "What this workflow does"
metadata:
  author: team-name
  version: "1.0"
parent: router-name.md          # Phase files only
compatibility: "Required tools"  # Router and single-file only
---
```

### Design Principles

Every multi-phase router should include a `## Design Principles` section after the summary. This section defines the behavioral contract for the workflow: who owns what, where the source of truth lives, and how the agent should behave. Four patterns to consider:

1. **Ownership** -- State who builds the configuration and who reviews it. This prevents the agent from deferring work to the wrong party.
2. **Source of truth** -- Direct the agent to existing files in the repo (schemas, examples, prior configurations) rather than relying on hard-coded templates in the workflow. Inline YAML drifts from the actual source of truth over time and becomes a maintenance burden.
3. **Ask less, infer more** -- The agent should carry forward data from prior phases, derive values from the repository, and verify state from the filesystem before asking the user. User questions should be reserved for decisions and information that can't be observed from disk.
4. **Prefer simple** -- Default to the simplest option and offer complexity only when the user needs it. Not every workflow invocation needs every feature.

These are starting points. Adapt them to the domain.

### Carry-Forward Data Between Phases

Each phase should declare what data it inherits from prior phases in a `**Carry forward from prior phases**` block after the prerequisites. This prevents the agent from re-asking for identifiers, paths, environment names, or other values it already collected. As phases progress, the carry-forward list grows.

### Inspect-Decide-Generate

Every step follows this cycle.

**Inspect** -- Read the repo: schemas, existing examples, prior phase outputs, carry-forward data. Derive everything you can before involving the user. Use a status-action table for branching:

```markdown
| Status | Action |
|--------|--------|
| File exists | Show existing config, ask if update needed |
| File missing, prerequisites met | Continue with generation |
| Prerequisites missing | Stop. Tell user what's needed first |
```

**Decide** -- Present the user with options and defaults. Ask only for decisions and information that can't be derived from the repo or prior phases. List numbered questions. Never let the agent guess a value -- but don't ask for values the agent can look up.

**Generate** -- Produce the config using existing files as reference. Prefer schemas, examples, and prior configurations over hard-coded templates. Inline markup drifts from the actual source of truth. Use it only as minimal annotation or when no existing reference exists. Specify the output path.

```markdown
**Generate**:

Use `schemas/path/to/schema.yml` as the source of truth for required fields.
Search `path/to/existing/examples/` for reference files.

\```yaml
# template with {placeholders} -- or reference existing files
\```

Write to: `path/to/{identifier}/file.yml`
```

### Status-Action Tables

Encode ALL conditional logic as tables, never as prose. This is the single most important convention -- it prevents the agent from misinterpreting branching.

Bad:
> "If the namespace exists, you should skip ahead unless the labels are wrong."

Good:

```markdown
| Status | Action |
|--------|--------|
| Namespace exists, labels match | Skip to next step |
| Namespace exists, labels outdated | Update labels |
| Namespace missing | Create namespace |
```

### Router Rules (multi-phase only)

The router:
- Collects ONE key identifier (service name, account alias, hostname, etc.)
- Detects progress by inspecting the repo (which files exist = which phases done)
- Routes to the correct phase
- **Never generates files itself** -- pure orchestration

The router MUST include:
1. **Prerequisites** -- fork/rebase instructions, clean session warning
2. **Entry Point** -- the one Ask
3. **Progress Detection table** -- maps file existence to phase completion
4. **Determine Phase table** -- maps detected state to recommended action
5. **Route to Phase** -- presents options to the user
6. **Phase Files table** -- lists all phases with `@` references

### Phase Module Rules

Each phase:
- Declares `parent:` in frontmatter linking back to the router
- Is independently invocable (user can jump straight to any phase)
- Follows Inspect-Decide-Generate for every step
- Ends with a **PR Checkpoint** listing exact files to include
- Links to the **Next Phase** at the bottom

### Progress Detection (critical)

Progress is detected from the **merged upstream primary branch**. The workflow must explicitly state:
- Sync fork and rebase (or pull) before each session
- Local uncommitted files don't count
- Unmerged feature branches don't count
- Prior LLM session history doesn't count
- A PR merge is the event that advances the state machine

### Validation (optional convenience)

Workflows produce files. CI/CD pipelines are the actual validation gate. Phases may include an optional validation step before the PR Checkpoint as a convenience for catching obvious errors early, but it is not part of the core Inspect-Decide-Generate cycle. Never interact with live infrastructure from within a workflow.

Common local validation tools by ecosystem:
- YAML: `yamllint`
- Kubernetes: `yamllint`, `kubeconform`
- Terraform: `terraform fmt -check`, `terraform validate`
- Ansible: `yamllint`, `ansible-lint`, `ansible-inventory --list`
- Helm: `helm lint`, `helm template`
- General: whatever linter/validator the target repo already uses

### PR Checkpoints

Every phase (or single-file workflow) ends with:

```markdown
## PR Checkpoint

**Title**: `[Prefix] Short description -- Phase N`

**Files to include**:
- `path/to/file-1.yml`
- `path/to/file-2.yml`
```

---

## How to Create a Workflow

When the user asks you to create a directed workflow, follow this process:

### Step 1: Understand the Process

Ask the user:
1. "What process do you want to encode?" (e.g., onboarding a service, setting up monitoring, creating a new environment)
2. "How long does it typically take?" (minutes → single-file, days/weeks → multi-phase)
3. "How many PRs should it produce?" (1 → single-file, multiple → multi-phase)
4. "What validation tools does your repo use?" (linters, schema validators, etc.)

### Step 2: Analyze the Target Repo

Inspect the user's repo to understand:
- **Directory structure** -- where config files live, naming conventions
- **File formats** -- YAML, HCL, JSON, etc.
- **Cross-referencing patterns** -- how files reference each other (`$ref`, imports, etc.)
- **Existing validation** -- Makefile targets, CI checks, linters
- **Placement conventions** -- where new files of each type should go

### Step 3: Design Phase Boundaries (multi-phase only)

Map the process to phases where each phase:
- Produces a coherent, independently reviewable PR
- Has a clear progress indicator (a file whose existence proves the phase is done)
- Groups logically related steps (not one step per phase, not everything in one phase)

### Step 4: Generate the Workflow Files

Write the files into the user's repo at `.agents/commands/`. Study the examples in this repo for structural reference:

| Example | Phases | Good reference for |
|---------|--------|--------------------|
| `examples/kubernetes-onboarding/` | 4 | Full router pattern, conditional phase skipping, YAML generation |
| `examples/terraform-aws-account/` | 3 | HCL generation, skippable phases, `terraform validate` |
| `examples/ansible-inventory/` | 3 | Inventory updates (append vs create), vault secrets, `ansible-lint` |
| `examples/contributor-access/` | 2 | Minimal router, non-infrastructure use case |

Use `templates/single-file-workflow.md` or `templates/multi-phase-router/` as your starting skeleton.

### Step 5: Suggest IDE Integration

After generating the workflow files, suggest symlinks:

```bash
# Cursor
ln -s ../../.agents/commands/{workflow}.md .cursor/commands/{workflow}.md

# Claude Code
ln -s ../../.agents/commands/{workflow}.md .claude/commands/{workflow}.md
```

---

## What NOT to Do

- **Never guess values.** If the workflow needs a cluster name, image registry, role ARN, or any environment-specific value, add a Decide directive. Do not invent values.
- **Never use prose for conditional logic.** Always use status-action tables.
- **Never put generation logic in the router.** The router detects and routes. Phases generate.
- **Never reference live infrastructure.** Workflows produce files. Validation is local.
- **Never create workflows in this repo.** Output goes in the user's target repo.
