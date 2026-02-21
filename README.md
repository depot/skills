# Depot AI Agent Skills & Plugins

Skills and plugins that teach AI coding agents how to use [Depot](https://depot.dev). Install them into your agent of choice so it can generate correct Depot CLI commands, configurations, and workflows.

## What's in this repo

### Skills

Individual SKILL.md files covering each Depot product area:

| Skill | Directory | Description |
|-------|-----------|-------------|
| **Container Builds** | [`depot-container-builds/`](depot-container-builds/SKILL.md) | `depot build`, `depot bake`, multi-platform builds, caching, Depot Registry, migration from Docker |
| **GitHub Actions Runners** | [`depot-github-runners/`](depot-github-runners/SKILL.md) | Managed GHA runners, runner labels/sizes, caching, Dagger, Dependabot, egress filtering |
| **Depot CI** | [`depot-ci/`](depot-ci/SKILL.md) | Depot CI beta, `depot ci migrate`, secrets/vars, running workflows, GHA compatibility |
| **General** | [`depot-general/`](depot-general/SKILL.md) | CLI installation, authentication (tokens, OIDC), project setup, org management, API, pricing |

### Plugins

Packaged plugins that bundle skills for specific agent platforms:

| Plugin | Directory | Description |
|--------|-----------|-------------|
| **Container Build Best Practices** | [`plugins/container-build-best-practices-plugin/`](plugins/container-build-best-practices-plugin/) | Comprehensive container build guide — Dockerfiles, CI/CD, troubleshooting, observability |

## Installation

### Claude Code

Install the container build best practices plugin directly:

```bash
claude plugin add --from depot/skills/plugins/container-build-best-practices-plugin
```

Or copy individual skill folders into your project:

```bash
mkdir -p .claude/skills
cp -r depot-container-builds .claude/skills/
cp -r depot-github-runners .claude/skills/
cp -r depot-ci .claude/skills/
cp -r depot-general .claude/skills/
```

You can also upload individual SKILL.md files as user skills in Claude.ai project settings.

### Cursor

Copy SKILL.md files into `.cursor/rules/` as `.mdc` files:

```bash
mkdir -p .cursor/rules
cp depot-container-builds/SKILL.md .cursor/rules/depot-container-builds.mdc
cp depot-github-runners/SKILL.md .cursor/rules/depot-github-runners.mdc
cp depot-ci/SKILL.md .cursor/rules/depot-ci.mdc
cp depot-general/SKILL.md .cursor/rules/depot-general.mdc
```

### AGENTS.md

Reference skills from your repository's `AGENTS.md` file:

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
