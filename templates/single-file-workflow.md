---
name: your-workflow-name              # Unique identifier, used as the skill name
description: "Short description of what this workflow does and when to use it"
metadata:
  author: your-team
  version: "1.0"
compatibility: "List any required tools (e.g., kubectl, terraform, helm)"
---

# Your Workflow Name

One-line summary of what this workflow produces.

## Prerequisites

- List what must be true before running this workflow
- Tools, access, prior steps

---

## Step 1: [Step Name]

**Inspect**:
- Read the repo for existing state, schemas, and examples
- Derive values before involving the user

| Status | Action |
|--------|--------|
| [Condition A] | [What to do] |
| [Condition B] | [What to do] |

**Decide**:
1. "Question for the user -- only what can't be derived from the repo"

**Generate**:

Prefer referencing existing files in the repo (schemas, examples, prior configurations) over hard-coded templates. Inline markup drifts from the actual source of truth over time. Use it only as minimal annotation or when no existing reference exists.

```yaml
# Your generated configuration here -- or reference an existing file:
# Use `schemas/path/to/schema.yml` as the source of truth for required fields.
# Search `path/to/existing/examples/` for reference files.
```

Write to: `path/to/output-file.yml`

---

## Step 2: [Step Name]

Repeat the Inspect-Decide-Generate cycle for each step.

---

## Validate (optional)

```bash
# Optional local validation command -- CI/CD is the actual gate
```

## PR Checkpoint

**Title**: `[Descriptive prefix] Short description of changes`

**Files to include**:
- `path/to/file-1.yml`
- `path/to/file-2.yml`
