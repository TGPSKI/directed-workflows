---
name: your-workflow-name-phase-NN
description: "Phase N: [Phase Name] -- Short description of what this phase produces"
metadata:
  author: your-team
  version: "1.0"
parent: your-workflow-name           # Matches the router's name field
---

# Phase N: [Phase Name]

One-line summary of what this phase produces.

**Typical PR**: 1
**Files produced**: file-1.yml, file-2.yml

## Prerequisites

- Previous phase complete (artifacts merged)
- Any tools or access needed for this phase

**Carry forward from prior phases**: List what the agent already knows from earlier phases (e.g., identifiers, paths, environment names, cluster names). The agent should use these values directly and not re-ask for them.

---

## Step 1: [Step Name]

**Inspect**:
- Read the repo for existing state, schemas, and examples
- Check carry-forward data from prior phases
- Derive values before involving the user

| Status | Action |
|--------|--------|
| [Condition A] | [What to do] |
| [Condition B] | [What to do] |

**Decide**:
1. "Question for the user -- only what can't be derived from the repo"

**Generate**:

Prefer referencing existing files in the repo (schemas, examples, prior configurations) over hard-coded templates.

```yaml
# Your generated configuration here -- or reference an existing file:
# Use `schemas/path/to/schema.yml` as the source of truth for required fields.
# Search `path/to/existing/examples/` for reference files.
```

Write to: `path/to/{identifier}/file.yml`

---

## Step 2: [Step Name]

Repeat the Inspect-Decide-Generate cycle for each step in this phase.

---

## Validate (optional)

```bash
# Optional local validation command -- CI/CD is the actual gate
```

## PR Checkpoint

**STOP. This phase is complete. Create the PR now. Do NOT proceed to the next phase.**

The next phase cannot begin until this PR is merged into the primary branch. Merging is the event that advances the workflow state machine. Starting the next phase before this PR is merged will cause progress detection to fail in the next session.

**Title**: `[Descriptive prefix] Short description -- Phase N`

**Files to include**:
- `path/to/file-1.yml`
- `path/to/file-2.yml`

**After creating the PR**, inform the user:

> PR created. This phase is done. After the PR is reviewed and merged, start a new chat session and invoke the workflow again -- it will detect the merged files and route to the next phase.

---

## Next Phase

After the PR is merged, start a new session and invoke the router. It will detect Phase N as complete and route to:

`@your-workflow-name/references/NN+1-name.md`
