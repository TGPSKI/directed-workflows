# Directed Workflows

**Directed Workflows** is a **command pattern** for encoding complex infrastructure procedures as **interactive, validated, agent-driven workflows** using structured prompts and repository context.

It turns brittle documentation, tribal knowledge, and error-prone manual processes into **guided execution paths** that scale expertise without freezing it into scripts or UIs.

---

## Why Directed Workflows Exist

Modern infrastructure tasks often fail not because they are impossible—but because they sit behind a **knowledge barrier**.

Common symptoms:

* Complex, multi-step procedures that require deep system knowledge
* Long time-to-completion for first-time or infrequent users
* High error rates due to missing context or undocumented conventions
* Dependency on a small set of experts to unblock progress
* Documentation that is outdated, incomplete, or assumes prior familiarity
* Scripts that are brittle and fail outside the “happy path”

Directed Workflows address the gap between:

> **What a user needs to accomplish**
> and
> **The knowledge required to do it correctly**

---

## What This Pattern Is (and Is Not)

### ✅ What it is

* A **workflow command pattern**, not a product or platform
* **Agent-first**, designed to run inside tools like Cursor, Claude, or other agentic IDEs
* **Repository-aware**, using real schemas, examples, and conventions
* **Human-guided**, not fully autonomous
* **Continuously validated**, not “run it and hope CI catches it”

### ❌ What it is not

* Not a wizard-based UI
* Not a custom CLI tool
* Not a chatbot with canned responses
* Not a replacement for documentation (docs are inputs, not outputs)

---

## Core Idea

Encode domain expertise directly into **workflow commands** that guide a human through complex tasks step-by-step, using:

* Interactive prompting
* Progressive disclosure
* Repository context exploration
* Continuous validation
* Optional integration with external systems

The agent becomes a **guided execution partner**, not a guesser.

---

## How It Works

### High-Level Flow

1. User invokes a workflow command
2. Agent loads the command definition
3. Agent asks a simple entry-point question
4. Agent explores repository context (schemas, examples, patterns)
5. Agent guides the user through each step
6. Files are created or updated incrementally
7. Validation occurs at each stage
8. Final validation confirms completion

---

## Command Definition Format

Workflow commands are defined as **Markdown files with YAML frontmatter**.

They are:

* Human-readable
* Version-controlled
* Easy to edit and review
* Discoverable by agents

### Example Structure

```markdown
---
description: Brief description of the command purpose
globs: [optional file patterns to scope command]
alwaysApply: false
---
# Command Title

## Entry Point
**Ask user first**: Simple question that unlocks maximum context

## Workflow Steps
### Step 1: Step name
Instructions, templates, examples, and validation criteria

### Step 2: Step name
...

## Output
Files or artifacts produced by the workflow

## Validation
How correctness is validated (schemas, commands, checks)

## Examples
References to existing examples in the repository
```

---

## Design Principles

### 1. Interactive Prompting

Commands explicitly request key information instead of assuming it.

**Result:** Fewer errors, higher confidence, better outcomes.

---

### 2. Entry-Point Simplicity

Each workflow starts with the *smallest possible question* that unlocks maximum context.

**Result:** Low cognitive load, immediate traction.

---

### 3. Progressive Disclosure

Complexity is revealed only when needed.

**Result:** Users aren’t overwhelmed, regardless of experience level.

---

### 4. Context-Driven Guidance

The agent explores the repository to find:

* Existing examples
* Schemas
* Conventions
* Related files and dependencies

**Result:** Output matches real patterns, not idealized ones.

---

### 5. Validation at Every Step

Validation happens continuously, not only in CI.

**Result:** Errors are caught early, when they’re cheapest to fix.

---

## Execution Model

Directed Workflows follow a **turn-based interaction loop**:

1. Agent asks for information or presents options
2. User responds
3. Agent processes input and explores context
4. Agent performs actions and validates results
5. Agent confirms completion and moves forward

Errors trigger clarification, examples, or fallback guidance—not failure.

---

## Example Use Cases

* RBAC configuration
* Alert routing setup
* Service onboarding
* Infrastructure migrations
* Compliance-sensitive changes
* Cross-system coordination (Git, Jira, Vault, etc.)

<!-- Concrete examples live in the `/examples` directory. -->

---

## Tool Compatibility

This pattern is designed to work with **existing agentic tools**, including:

* IDE-embedded agents
* Repository-aware assistants
* MCP-enabled integrations (Git, issue trackers, CI, etc.)

No custom runtime required.

---

## Why This Works

Directed Workflows succeed where docs, scripts, and wizards fail because they:

* Preserve human judgment
* Encode expertise without freezing it
* Adapt to context and variation
* Validate continuously
* Live alongside the code they operate on

They scale *understanding*, not just execution.

---

## Provenance

This pattern was developed through real-world use in complex infrastructure environments and is published here for community use, iteration, and extension.
