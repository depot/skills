# Depot AI Agent Skills

This repository contains SKILL.md files that teach AI coding agents how to use [Depot](https://depot.dev) products. Each skill follows the SKILL.md specification with YAML frontmatter and provides technical reference material that agents use to generate correct Depot CLI commands, configurations, and workflows.

## Skills

| Skill | Directory | Description |
|-------|-----------|-------------|
| **Container Builds** | `depot-container-builds/` | `depot build`, `depot bake`, multi-platform builds, caching, Depot Registry, migration from Docker |
| **GitHub Actions Runners** | `depot-github-runners/` | Managed GHA runners, runner labels/sizes, caching, Dagger, Dependabot, egress filtering |
| **Depot CI** | `depot-ci/` | Depot CI beta, `depot ci migrate`, secrets/vars, running workflows, GHA compatibility |
| **General** | `depot-general/` | CLI installation, authentication (tokens, OIDC), project setup, org management, API, pricing |

## Usage

### Claude Code / Claude.ai

Copy skill folders into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r depot-container-builds .claude/skills/
cp -r depot-github-runners .claude/skills/
cp -r depot-ci .claude/skills/
cp -r depot-general .claude/skills/
```

Or upload individual SKILL.md files as user skills in Claude.ai project settings.

### Cursor

Reference the SKILL.md content in `.cursor/rules/*.mdc` files:

```bash
mkdir -p .cursor/rules
cp depot-container-builds/SKILL.md .cursor/rules/depot-container-builds.mdc
cp depot-github-runners/SKILL.md .cursor/rules/depot-github-runners.mdc
cp depot-ci/SKILL.md .cursor/rules/depot-ci.mdc
cp depot-general/SKILL.md .cursor/rules/depot-general.mdc
```

### AGENTS.md

Reference these skills from your repository's `AGENTS.md` file:

```markdown
## Depot

This project uses Depot for container builds and CI. See the following skill files for reference:

- [Container Builds](./depot-container-builds/SKILL.md)
- [GitHub Actions Runners](./depot-github-runners/SKILL.md)
- [Depot CI](./depot-ci/SKILL.md)
- [General Setup](./depot-general/SKILL.md)
```

### OpenAI / Codex

Include the contents of relevant SKILL.md files in your custom instructions or system prompt. For Codex CLI, place skill files in your project and reference them in your configuration.

## Documentation

For full Depot documentation, visit [https://depot.dev/docs](https://depot.dev/docs).
