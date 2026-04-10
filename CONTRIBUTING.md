# Contributing to directed-workflows

Thank you for your interest in contributing to directed-workflows.

## How to Contribute

### Reporting Issues

Open a GitHub issue describing the problem, including:
- What you expected to happen
- What actually happened
- Steps to reproduce

### Proposing Changes

1. Fork the repository
2. Create a feature branch from `main`
3. Make your changes
4. Submit a pull request against `main`

### Agent Skills Specification

All `SKILL.md` files must conform to the [Agent Skills specification](https://github.com/agentskills/agentskills/blob/1eeb1aab054a20e9b8508887e82bd911a29235c8/docs/specification.mdx).

Required frontmatter fields:

| Field | Constraints |
|-------|-------------|
| `name` | 1-64 chars, lowercase alphanumeric + hyphens, must match parent directory name |
| `description` | 1-1024 chars, describes what the skill does and when to use it |

Optional frontmatter fields: `license`, `compatibility`, `metadata`, `allowed-tools`.

### Workflow File Conventions

When adding or modifying workflow examples:

- Entry points are always `SKILL.md` with valid YAML frontmatter per the spec above
- Phase files live under `references/` and use `phase-0N-name.md` naming
- The `parent` field references the router's name, not its filename
- Follow the Inspect-Decide-Generate cycle in every step
- Use Status-Action tables for conditional logic, not prose

### Adding Examples

1. Create a directory under `examples/<domain>/<workflow-name>/`
2. Add a `SKILL.md` router as the entry point
3. Add phase files under `references/`
4. Update the examples table in `README.md`

### Adding Templates

1. Place new templates under `templates/`
2. Single-session workflows: one `.md` file
3. Multi-session workflows: a directory with `router.md` and `phase-template.md`

### Code of Conduct

Be respectful. Constructive feedback is welcome; personal attacks are not.

### Quality Gates

Every PR must:

1. Not break existing workflow file structure
2. Follow the naming conventions documented in `ROUTER_PATTERN.md`
3. Conform to the [Agent Skills specification](https://github.com/agentskills/agentskills/blob/1eeb1aab054a20e9b8508887e82bd911a29235c8/docs/specification.mdx) — all `SKILL.md` files must have valid frontmatter with required `name` and `description` fields
4. Include a clear description of what the change does and why

### Running Validation

There is no build step. Review your markdown renders correctly and all internal `@`-references resolve to existing files.

### License

By contributing, you agree that your contributions will be licensed under the GNU General Public License v3.0.
