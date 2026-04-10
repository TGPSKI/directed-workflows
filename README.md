# Directed Workflows

Structured markdown files that AI agents execute interactively. The agent asks questions, inspects your repo, and generates valid configuration. You provide the decisions.

```
User: @onboard-service/SKILL.md

Agent: "What is the name of your service?"
User: "payment-gateway"

Agent: [inspects repo]
       "Namespace exists, no Deployment found. Phase 2. Continue?"
User: "Yes"

Agent: [generates Deployment, Service, Ingress YAML; validates locally]
       "Files created and validated. Ready for PR."
```

The workflow file is the program. The AI agent is the runtime.

## Quick Start

### 1. Get the skill into your agent's scope

Clone this repo and add it to your workspace alongside your target repo:

```bash
git clone https://github.com/tgpski/directed-workflows.git
```

The agent reads `AGENTS.md` → `SKILL.md` for instructions. Most AI IDEs (Cursor, Claude Code, Windsurf, etc.) pick up `AGENTS.md` or `SKILL.md` automatically when the repo is in the workspace.

If your IDE supports global skill installation (e.g. `~/.cursor/skills/`, `~/.claude/skills/`), you can clone there instead for availability across all workspaces.

### 2. Ask the agent to create a workflow

Start a new agent session in your target repo:

```
I want a directed workflow for onboarding new Helm charts to our platform.
The process takes about a week, produces 3 PRs, and involves creating a
Chart.yaml, values files per environment, and a CI pipeline config.
```

The agent studies the examples and templates, analyzes your repo, and generates workflow files matching your conventions.

### 3. Review and use

The agent writes files to `.agents/skills/` in your repo:

```
your-repo/.agents/skills/onboard-chart/
├── SKILL.md                        # Router (entry point)
└── references/
    ├── 01-scaffold.md              # Chart.yaml, base values
    ├── 02-environments.md          # Per-env values files
    └── 03-pipeline.md              # CI config
```

Symlink for IDE auto-discovery (adapt the target path to your IDE):

```bash
ln -s ../../.agents/skills/onboard-chart .cursor/skills/onboard-chart
ln -s ../../.agents/skills/onboard-chart .claude/skills/onboard-chart
```

Invoke the workflow: `@onboard-chart/SKILL.md`. Come back next week, invoke the same file, and it picks up where you left off.

---

## How It Works

### Inspect-Decide-Generate

Every step follows the same cycle:

1. **Inspect** the repo -- schemas, existing files, prior phase outputs. Derive everything possible before involving the user.
2. **Decide** -- present options and defaults. Ask only for information that can't be derived.
3. **Generate** config using existing files as reference. Prefer repo patterns over hard-coded templates.

### Status-Action Tables

Conditional logic is encoded as lookup tables, not prose. Tables are unambiguous for both humans and agents.

```markdown
| Status | Action |
|--------|--------|
| Namespace exists, labels match | Skip to next step |
| Namespace exists, labels outdated | Update labels |
| Namespace missing | Create namespace |
```

### Multi-Phase Router

For processes that span multiple sessions and PRs, a router file detects progress from the **merged primary branch** and routes to the correct phase. The merged default branch *is* the state -- no database, no session store, no external tracking. A PR merge advances the state machine.

See [The Multi-Phase Router Pattern](./ROUTER_PATTERN.md) for the full deep-dive.

### The Substitution Property

Workflow files specify *what information is needed*, not *who answers*. The same file works whether a human, an orchestrating agent, or a policy engine provides the inputs.

## Repository Structure

```
directed-workflows/
├── SKILL.md                           # Agent instructions for generating workflows
├── AGENTS.md                          # Redirect to SKILL.md
├── README.md                          # You are here
├── ROUTER_PATTERN.md                  # Multi-phase router deep-dive
├── examples/
│   ├── kubernetes-onboarding/
│   │   └── onboard-service/           # 4-phase: namespace, deployment, ingress, monitoring
│   │       ├── SKILL.md               # Router
│   │       └── references/
│   ├── terraform-aws-account/
│   │   └── provision-account/         # 3-phase: provider, state backend, IAM
│   │       ├── SKILL.md
│   │       └── references/
│   ├── ansible-inventory/
│   │   └── add-host-group/            # 3-phase: hosts, group_vars, playbook
│   │       ├── SKILL.md
│   │       └── references/
│   └── contributor-access/
│       └── grant-access/              # 2-phase: identity, permissions
│           ├── SKILL.md
│           └── references/
└── templates/
    ├── single-file-workflow.md        # One-session template
    └── multi-phase-router/            # Multi-session template (router + phase)
```

### Try the examples

| Example | Entry point | Phases |
|---------|-------------|--------|
| Kubernetes service onboarding | `@examples/kubernetes-onboarding/onboard-service/SKILL.md` | 4 |
| Terraform AWS account | `@examples/terraform-aws-account/provision-account/SKILL.md` | 3 |
| Ansible inventory | `@examples/ansible-inventory/add-host-group/SKILL.md` | 3 |
| Contributor access | `@examples/contributor-access/grant-access/SKILL.md` | 2 |

### Write one manually

1. Copy `templates/single-file-workflow.md` (one session) or `templates/multi-phase-router/` (multi-session)
2. Fill in the Inspect/Decide/Generate steps
3. Place in your repo at `.agents/skills/{workflow-name}/SKILL.md`

## File Layout Convention

Workflows live in `.agents/skills/` -- tool-agnostic and version-controlled. Symlink into IDE-specific directories for auto-discovery:

```bash
# Adapt paths to your IDE's skill/agent directory
ln -s ../../.agents/skills/onboard-service .cursor/skills/onboard-service
ln -s ../../.agents/skills/onboard-service .claude/skills/onboard-service
```

## Works With

Any system where configuration is file-based and follows placement conventions:

| Platform | Example |
|----------|---------|
| **Kubernetes** | Service onboarding, namespace setup, monitoring ([example](examples/kubernetes-onboarding/onboard-service/SKILL.md)) |
| **Terraform** | Account provisioning, module creation ([example](examples/terraform-aws-account/provision-account/SKILL.md)) |
| **Ansible** | Inventory management, playbook wiring ([example](examples/ansible-inventory/add-host-group/SKILL.md)) |
| **Helm / ArgoCD / Crossplane** | Chart scaffolding, app onboarding, XRD authoring |
| **Any GitOps repo** | File-based config with predictable directory structures |

## Further Reading

- [The Multi-Phase Router Pattern](./ROUTER_PATTERN.md) -- codebase-as-state, progress detection, phase boundaries
- [SKILL.md](SKILL.md) -- the instructions the agent reads when generating workflows

## License

[GNU General Public License v3.0](./LICENSE)

## Author

Tyler Pate ([@TGPSKI](https://github.com/tgpski)), 2026
