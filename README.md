# Directed Workflows

**Markdown files that turn AI coding agents into interactive platform experts.**

Directed workflows encode multi-step configuration processes -- onboarding a service, setting up monitoring, migrating environments -- as markdown files that AI agents execute interactively. The agent asks questions, inspects your repo, and generates valid configuration. The human provides the inputs that require judgment.

```
User: @onboard-service.md

Agent: "What is the name of your service?"
User: "payment-gateway"

Agent: [inspects repo]
       "Namespace exists, no Deployment found. You're at Phase 2. Continue?"
User: "Yes"

Agent: [generates Deployment, Service, Ingress YAML; validates locally]
       "Resources created and validated. Ready for PR."
```

## Quick Start

### 1. Clone and add to your workspace

```bash
git clone https://github.com/tgpski/directed-workflows.git
```

Open your AI IDE (Cursor, VS Code with Claude/Copilot, Windsurf) and add **both** this repo and your target repo to the same workspace.

### 2. Ask the agent to create a workflow

Start a new agent session and describe the process you want to encode:

```
I want a directed workflow for onboarding new Helm charts to our platform.
The process takes about a week, produces 3 PRs, and involves creating a
Chart.yaml, values files per environment, and a CI pipeline config.
Use the directed-workflows repo as the pattern reference.
```

The agent reads `AGENTS.md`, studies the examples and templates, analyzes your repo's structure, and generates workflow files tailored to your conventions.

### 3. Review the output

The agent creates files in your target repo:

```
your-repo/.agents/commands/
├── onboard-chart.md                # Router (entry point)
└── onboard-chart/
    ├── phase-1-scaffold.md         # Chart.yaml, base values
    ├── phase-2-environments.md     # Per-env values files
    └── phase-3-pipeline.md         # CI config
```

Review the generated workflows like any other PR. The workflow files are version-controlled alongside the config they produce.

### 4. Use the workflow

Invoke the workflow from your target repo:

```
@onboard-chart.md
```

The agent asks questions, inspects your repo, and generates valid configuration. Come back next week, invoke the same file, and it picks up where you left off.

---

## Why

Every mature platform develops a configuration layer that's too complex for casual contributors. The documentation exists. The gap between reading it and producing valid configuration is where mistakes happen.

| Existing Tool | Limitation |
|---------------|-----------|
| Developer portals (Backstage) | Require building/maintaining a web app. Good at creation-time, not multi-step. |
| Template generators (Cookiecutter) | One-shot. No conditional logic, no existing-state inspection. |
| Documentation | Tells you what to do but can't do it with you. |
| Raw AI assistants | Hallucinate resource names, invent conventions, guess values. |

Directed workflows close this gap. The workflow file is the program. The AI agent is the runtime.

## Key Concepts

### Design Principles

Every workflow defines a behavioral contract: who owns the configuration, where the source of truth lives, what the agent should derive vs ask, and whether to default to simple or complex. This prevents the agent from guessing values, deferring work to the wrong party, or over-engineering the output.

#### Inspect-Decide-Generate

Every step follows the same cycle: inspect the repo and prior phase outputs to derive what you can, present the user with options and defaults for decisions that can't be derived, generate config using existing files as reference over hard-coded templates.

#### Carry-Forward Data

Each phase declares what it inherits from prior phases -- identifiers, paths, environment names. The agent uses these directly instead of re-asking. As phases progress, the carry-forward list grows.

#### Status-Action Tables

Conditional logic encoded as lookup tables, not prose. Unambiguous for both humans and agents.

### Multi-Phase Router

For processes that span multiple sessions and PRs, a router file detects progress from the **merged upstream primary branch** and routes to the correct phase. No external state tracking -- the merged default branch *is* the state. Before each session, sync your fork and rebase onto upstream `main` (or pull if on a direct clone). Local uncommitted files, unmerged branches, and prior session history are explicitly excluded from detection. A PR merge is the event that advances the state machine.

### The Substitution Property

Workflow files specify *what information is needed*, not *who answers*. The same file works whether a human, an orchestrating agent, or a policy engine provides the inputs.

## Repository Structure

```
directed-workflows/
├── README.md                          # You are here (includes router pattern deep-dive)
├── AGENTS.md                          # Agent instructions for generating workflows
├── examples/
│   ├── kubernetes-onboarding/         # 4-phase example (full router demo)
│   │   ├── onboard-service.md         # Router (entry point)
│   │   └── onboard-service/
│   │       ├── phase-1-foundation.md  # Namespace, RBAC
│   │       ├── phase-2-workload.md    # Deployment, Service
│   │       ├── phase-3-exposure.md    # Ingress, NetworkPolicy (skippable)
│   │       └── phase-4-observability.md # ServiceMonitor
│   ├── terraform-aws-account/         # 3-phase example (HCL generation)
│   │   ├── provision-account.md       # Router (entry point)
│   │   └── provision-account/
│   │       ├── phase-1-provider.md    # AWS provider, version constraints
│   │       ├── phase-2-state.md       # S3 backend, DynamoDB lock (skippable)
│   │       └── phase-3-iam.md         # IAM baseline roles, outputs
│   ├── ansible-inventory/             # 3-phase example (inventory + playbook)
│   │   ├── add-host-group.md          # Router (entry point)
│   │   └── add-host-group/
│   │       ├── phase-1-inventory.md   # Add hosts to environment inventory
│   │       ├── phase-2-group-vars.md  # Group variables, vault secrets
│   │       └── phase-3-playbook.md    # Playbook with roles
│   └── contributor-access/            # 2-phase example (minimal router)
│       ├── grant-access.md            # Router (entry point)
│       └── grant-access/
│           ├── phase-1-identity.md    # User file, team membership
│           └── phase-2-access.md      # Roles, repo permissions
└── templates/
    ├── single-file-workflow.md        # Starter template for simple workflows
    └── multi-phase-router/            # Starter template for router workflows
        ├── router.md
        └── phase-template.md
```

### Try the examples

To see the pattern in action before creating your own, run any of the included examples:

| Example | Command | Phases |
|---------|---------|--------|
| Kubernetes service onboarding | `@examples/kubernetes-onboarding/onboard-service.md` | 4 |
| Terraform AWS account provisioning | `@examples/terraform-aws-account/provision-account.md` | 3 |
| Ansible inventory setup | `@examples/ansible-inventory/add-host-group.md` | 3 |
| Contributor access management | `@examples/contributor-access/grant-access.md` | 2 |

### Write one manually

If you prefer to write workflows by hand:

1. Copy `templates/single-file-workflow.md` (one session) or `templates/multi-phase-router/` (multi-session)
2. Fill in the Inspect/Decide/Generate steps for your process
3. Drop it in your repo at `.agents/commands/`

## IDE Integration

Directed workflows work with any AI IDE that supports referencing files. Place workflows in `.agents/commands/` and symlink to tool-specific directories:

```bash
# Cursor
ln -s ../../.agents/commands/onboard-service.md .cursor/commands/onboard-service.md

# Claude Code
ln -s ../../.agents/commands/onboard-service.md .claude/commands/onboard-service.md
```

One source of truth, multiple entry points.

## Works With

| Platform | Example Use Case |
|----------|-----------------|
| **Kubernetes** | Service onboarding, namespace setup, monitoring ([example](examples/kubernetes-onboarding/onboard-service.md)) |
| **Terraform** | AWS account provisioning, module creation, environment setup ([example](examples/terraform-aws-account/provision-account.md)) |
| **Ansible** | Inventory management, host group setup, playbook wiring ([example](examples/ansible-inventory/add-host-group.md)) |
| **ArgoCD** | Application onboarding, sync policy setup |
| **Crossplane** | XRD/Composition authoring |
| **Helm** | Chart scaffolding, multi-env values |
| **Any GitOps repo** | Any file-based config with placement conventions |

## How It Compares

| Approach | Resumable | Modular | Fits Context Window | Small PRs | Zero Infra |
|----------|-----------|---------|---------------------|-----------|------------|
| Single monolithic prompt | No | No | No | No | Yes |
| Backstage scaffolder | No | Partial | N/A | No | No |
| External workflow engine | Yes | Depends | Depends | Depends | No |
| **Directed Workflows** | **Yes** | **Yes** | **Yes** | **Yes** | **Yes** |

## Further Reading

- [The Multi-Phase Router Pattern](./ROUTER_PATTERN.md)
- [AGENTS.md](AGENTS.md) -- instructions the agent reads when generating workflows for your repo
- [Agent Skills Standard](https://agentskills.io/) -- the adjacent open standard for single-invocation agent capabilities

## License

Apache 2.0

## Author

Tyler Pate ([@TGPSKI(https://github.com/tgpski)), 2026
