---
name: your-workflow-name              # Unique identifier, used as the skill name
description: "Short description. This is the router -- it detects progress and routes to phases."
metadata:
  author: your-team
  version: "1.0"
compatibility: "List any required tools"
---

# Your Workflow Name

One-line summary. This workflow produces small, focused PRs -- one per phase.

## Design Principles

<!-- Define the behavioral contract for this workflow. Remove or replace these examples. -->

- **[Ownership]**: Who builds the configuration, who reviews it, who approves it.
- **[Source of truth]**: Where the agent should look for schemas, examples, and valid values -- existing files in the repo, not hard-coded templates in this workflow.
- **[Ask less, infer more]**: Carry forward data from prior phases. Derive values from the repository. Only ask the user for decisions and information that can't be observed from disk.
- **[Prefer simple]**: Default to the simplest option. Offer complexity only when the user needs it.

## Prerequisites

- **Start from an up-to-date primary branch.** Sync your fork and rebase onto the latest upstream default branch (or `git pull` if working on a direct clone) before invoking this workflow. Progress detection relies on files merged to the upstream primary branch -- not local commits, uncommitted changes, or prior session output.
- **Start with a clean chat session.** Prior conversation history causes the agent to conflate previous output with actual repository state. Each phase should be a new session.

## State Machine

The PR merge is the event that advances the state machine. Each phase produces exactly one PR. The workflow cycle is:

1. **Detect** -- Read the filesystem to determine the current phase
2. **Execute** -- Run the phase (Inspect → Decide → Generate)
3. **Checkpoint** -- STOP. Commit, push, create PR. Do NOT proceed to the next phase.
4. **Merge** -- PR is reviewed and merged (outside this workflow)
5. **Resume** -- User starts a new session, invokes the workflow again, detection advances to the next phase

Local uncommitted files, unmerged feature branches, and prior LLM session history all exist outside this state machine. Only merged files on the primary branch advance progress.

## Entry Point

**Ask**: "What is the [key identifier]?" (e.g., service name, project name, username)

## Detect Current Progress

After getting the identifier, inspect the repository to determine what's already been completed. **Detection must be based on files that exist on disk from the primary branch** -- not on local uncommitted files, unmerged feature branches, or content from prior chat sessions. Use file search tools (Glob, Read) to check the actual filesystem.

### Progress Detection

| Check | What to Look For | Indicates |
|-------|-----------------|-----------|
| [Phase 1 artifact] | `path/to/{identifier}/file.yml` | Phase 1 complete |
| [Phase 2 artifact] | `path/to/{identifier}/other-file.yml` | Phase 2 complete |

### Determine Phase

| Detected State | Recommended Action |
|----------------|-------------------|
| No files found | Start at **Phase 1** |
| Phase 1 artifact exists, Phase 2 missing | Continue at **Phase 2** |
| All artifacts exist | Workflow complete |

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

## Variant Paths (if applicable)

| Scenario | Phases |
|----------|--------|
| Standard | 1 → 2 |
| [Variant] | 1 → 2 (modified) |

## Phase Files

| Phase | File | Description | PR |
|-------|------|-------------|-----|
| 1 | `@your-workflow-name/phase-1-name.md` | [What Phase 1 does] | 1 |
| 2 | `@your-workflow-name/phase-2-name.md` | [What Phase 2 does] | 1 |
