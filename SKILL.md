---
name: directed-workflows
description: "Creates directed workflow files for any repository. Generates structured markdown that AI agents execute interactively to guide users through multi-step configuration processes like service onboarding, environment setup, and infrastructure provisioning. Use when asked to create a directed workflow, encode a process, or build an agent-guided configuration flow."
---

# Directed Workflows -- Skill Instructions

Your job is to help the user create directed workflows for the repository they currently have open.

References to examples and templates below are relative to the skill directory where this file lives.

## Your Task

1. **Understand their process** -- what multi-step task they want to encode
2. **Analyze their workspace** -- file structure, naming conventions, validation tools
3. **Generate workflow files** -- in the current workspace, following the conventions below

---

## Output Structure

### Single-File Workflow

Use when the process completes in one session and one PR.

```
.agents/skills/{workflow-name}/
└── SKILL.md
```

### Multi-Phase Router

Use when the process spans multiple sessions or produces multiple PRs.

```
.agents/skills/{workflow-name}/
├── SKILL.md                        # Router (entry point, no generation)
└── references/
    ├── 01-{name}.md                # Phase module
    ├── 02-{name}.md                # Phase module
    └── 03-{name}.md                # Phase module
```

| Characteristic | Single-file | Multi-phase router |
|---------------|-------------|-------------------|
| Completes in one session | Yes | Overkill |
| Spans days/weeks | No | Yes |
| Produces 1 PR | Yes | No -- one PR per phase |
| Has 5+ steps with branching | Possible but fragile | Yes |

---

## Conventions

### Frontmatter

Every workflow file starts with YAML frontmatter:

```yaml
---
name: workflow-name
description: "What this workflow does and when to use it"
metadata:
  author: team-name
  version: "1.0"
parent: workflow-name              # Phase files only -- matches the router's name field
compatibility: "Required tools"    # Router and single-file only
---
```

The `parent` field on phase files references the `name` from the router's frontmatter, not a filename. This lets the agent trace any phase back to its router unambiguously.

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

**Decide** -- Present the user with options and defaults. Ask only for decisions that can't be derived from the repo or prior phases. Never guess a value -- but don't ask for values the agent can look up.

**Generate** -- Produce config using existing files as reference. Prefer schemas, examples, and prior configurations over hard-coded templates. Specify the output path.

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

Encode ALL conditional logic as tables, never as prose.

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

### Design Principles (multi-phase routers)

Every router should include a `## Design Principles` section defining the behavioral contract:

1. **Ownership** -- Who builds the configuration, who reviews it.
2. **Source of truth** -- Point the agent to existing files in the repo, not hard-coded templates.
3. **Ask less, infer more** -- Carry forward data from prior phases. Derive values from the repository. Only ask for decisions that can't be observed from disk.
4. **Prefer simple** -- Default to the simplest option. Offer complexity only when needed.

### Carry-Forward Data

Each phase declares what it inherits from prior phases in a `**Carry forward from prior phases**` block. This prevents re-asking for identifiers, paths, or environment names already collected.

### Router Rules

The router:
- Collects ONE key identifier (service name, account alias, hostname, etc.)
- Detects progress by inspecting the repo (which files exist = which phases done)
- Routes to the correct phase
- **Never generates files itself**

The router MUST include:
1. **Prerequisites** -- fork/rebase instructions, clean session warning
2. **Entry Point** -- the one Ask
3. **Progress Detection table** -- maps file existence to phase completion
4. **Determine Phase table** -- maps detected state to recommended action
5. **Route to Phase** -- presents options to the user
6. **Phase Files table** -- lists all phases with `@` references

### Phase Module Rules

Each phase:
- Declares `parent:` in frontmatter (matching the router's `name` field)
- Lives in the `references/` subdirectory of the workflow
- Is independently invocable
- Follows Inspect-Decide-Generate for every step
- Ends with a **PR Checkpoint** listing exact files to include
- Links to the **Next Phase**

### Progress Detection

Progress is detected from the **merged upstream primary branch**:
- Sync fork and rebase (or pull) before each session
- Local uncommitted files don't count
- Unmerged feature branches don't count
- Prior LLM session history doesn't count
- A PR merge advances the state machine

### Validation

Workflows produce files. CI/CD pipelines are the actual gate. Phases may include an optional local validation step before the PR Checkpoint:

- YAML: `yamllint`
- Kubernetes: `kubeconform`
- Terraform: `terraform fmt -check`, `terraform validate`
- Ansible: `ansible-lint`, `ansible-inventory --list`
- Helm: `helm lint`, `helm template`

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

### Step 1: Understand the Process

Ask the user:
1. What process do you want to encode?
2. How long does it typically take? (minutes → single-file, days/weeks → multi-phase)
3. How many PRs should it produce? (1 → single-file, multiple → multi-phase)
4. What validation tools does your repo use?

### Step 2: Analyze the Workspace

Inspect the workspace:
- Directory structure and naming conventions
- File formats (YAML, HCL, JSON, etc.)
- Cross-referencing patterns (`$ref`, imports, etc.)
- Existing validation (Makefile targets, CI checks, linters)
- Placement conventions for new files

### Step 3: Design Phase Boundaries (multi-phase only)

Each phase should:
- Produce a coherent, independently reviewable PR
- Have a clear progress indicator (a file whose existence proves completion)
- Group logically related steps

### Step 4: Generate the Workflow Files

Write files to `.agents/skills/{workflow-name}/` in the user's workspace. For multi-phase routers, place phase files in a `references/` subdirectory. Study the examples in the skill directory:

| Example | Phases | Good reference for |
|---------|--------|--------------------|
| `examples/kubernetes-onboarding/` | 4 | Full router, conditional skipping, YAML generation |
| `examples/terraform-aws-account/` | 3 | HCL generation, skippable phases |
| `examples/ansible-inventory/` | 3 | Inventory updates, vault secrets |
| `examples/contributor-access/` | 2 | Minimal router, non-infrastructure use case |

Use `templates/single-file-workflow.md` or `templates/multi-phase-router/` as the starting skeleton.

### Step 5: Enable IDE Discovery

Create symlinks from IDE-specific directories to the canonical location. Adapt the target path to whatever IDE the user has:

```bash
ln -s ../../.agents/skills/{workflow-name} .cursor/skills/{workflow-name}
ln -s ../../.agents/skills/{workflow-name} .claude/skills/{workflow-name}
```

---

## What NOT to Do

- **Never guess values.** Add a Decide directive instead.
- **Never use prose for conditional logic.** Use status-action tables.
- **Never put generation logic in the router.** The router detects and routes. Phases generate.
- **Never reference live infrastructure.** Workflows produce files. Validation is local.
- **Never modify the skill directory.** Output goes in the user's workspace.
