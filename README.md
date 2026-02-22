# Depot Skills

Skills that teach AI coding agents how to use [Depot](https://depot.dev). Install them into your agent of choice so it can generate correct Depot CLI commands, configurations, and workflows.

## What's in this repo

| Skill | Directory | Description |
|-------|-----------|-------------|
| **Container Builds** | [`skills/depot-container-builds/`](skills/depot-container-builds/SKILL.md) | `depot build`, `depot bake`, multi-platform builds, caching, Depot Registry, migration from Docker |
| **GitHub Actions Runners** | [`skills/depot-github-runners/`](skills/depot-github-runners/SKILL.md) | Managed GHA runners, runner labels/sizes, caching, Dagger, Dependabot, egress filtering |
| **Depot CI** | [`skills/depot-ci/`](skills/depot-ci/SKILL.md) | Depot CI beta, `depot ci migrate`, secrets/vars, running workflows, GHA compatibility |
| **General** | [`skills/depot-general/`](skills/depot-general/SKILL.md) | CLI installation, authentication (tokens, OIDC), project setup, org management, API, pricing |

## Installation

### Claude Code

Add the marketplace and install the Depot plugin:

```bash
claude plugin marketplace add depot/skills
claude plugin install depot@depot-skills
```

### Codex

Install all Depot skills into Codex's local skills directory (`$CODEX_HOME/skills`, usually `~/.codex/skills`):

```bash
mkdir -p "$HOME/.codex/skills"
cp -R skills/depot-container-builds "$HOME/.codex/skills/depot-container-builds"
cp -R skills/depot-github-runners "$HOME/.codex/skills/depot-github-runners"
cp -R skills/depot-ci "$HOME/.codex/skills/depot-ci"
cp -R skills/depot-general "$HOME/.codex/skills/depot-general"
```

Then reference them from your repo's `AGENTS.md` so Codex can discover and use them in this workspace:

```markdown
## Skills
- depot-container-builds: (file: ~/.codex/skills/depot-container-builds/SKILL.md)
- depot-github-runners: (file: ~/.codex/skills/depot-github-runners/SKILL.md)
- depot-ci: (file: ~/.codex/skills/depot-ci/SKILL.md)
- depot-general: (file: ~/.codex/skills/depot-general/SKILL.md)
```

### Cursor

Install the Depot plugin from the Cursor Marketplace:

1. In Cursor chat, run `/add-plugin`.
2. Search for **Depot Skills** (or enter `depot/skills`).
3. Install the plugin.

Manual fallback: install as project rules by copying each `SKILL.md` into `.cursor/rules/` as an `.mdc` file:

```bash
mkdir -p .cursor/rules
cp skills/depot-container-builds/SKILL.md .cursor/rules/depot-container-builds.mdc
cp skills/depot-github-runners/SKILL.md .cursor/rules/depot-github-runners.mdc
cp skills/depot-ci/SKILL.md .cursor/rules/depot-ci.mdc
cp skills/depot-general/SKILL.md .cursor/rules/depot-general.mdc
```

After manual install, open Cursor Settings -> Rules and verify these files are enabled for the workspace.

### AGENTS.md

Reference skills from your repository's `AGENTS.md` file:

```markdown
## Depot

This project uses Depot for container builds and CI. See the following skill files for reference:

- [Container Builds](./skills/depot-container-builds/SKILL.md)
- [GitHub Actions Runners](./skills/depot-github-runners/SKILL.md)
- [Depot CI](./skills/depot-ci/SKILL.md)
- [General Setup](./skills/depot-general/SKILL.md)
```

## Documentation

For full Depot documentation, visit [https://depot.dev/docs](https://depot.dev/docs).
