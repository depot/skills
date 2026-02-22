# Depot AI Agent Skills

Skills that teach AI coding agents how to use [Depot](https://depot.dev). Install them into your agent of choice so it can generate correct Depot CLI commands, configurations, and workflows.

## What's in this repo

| Skill | Directory | Description |
|-------|-----------|-------------|
| **Container Builds** | [`skills/depot-container-builds/`](skills/depot-container-builds/SKILL.md) | `depot build`, `depot bake`, multi-platform builds, caching, Depot Registry, migration from Docker |
| **GitHub Actions Runners** | [`skills/depot-github-runners/`](skills/depot-github-runners/SKILL.md) | Managed GHA runners, runner labels/sizes, caching, Dagger, Dependabot, egress filtering |
| **Depot CI** | [`skills/depot-ci/`](skills/depot-ci/SKILL.md) | Depot CI beta, `depot ci migrate`, secrets/vars, running workflows, GHA compatibility |
| **General** | [`skills/depot-general/`](skills/depot-general/SKILL.md) | CLI installation, authentication (tokens, OIDC), project setup, org management, API, pricing |

## Installation

### skills.sh

```bash
npx skills add depot/skills
```

See [skills.sh](https://skills.sh) for more info.

### Claude Code

Add the marketplace and install the Depot plugin:

```bash
claude plugin marketplace add depot/skills
claude plugin install depot@depot-skills
```

### Cursor

Copy SKILL.md files into `.cursor/rules/` as `.mdc` files:

```bash
mkdir -p .cursor/rules
cp skills/depot-container-builds/SKILL.md .cursor/rules/depot-container-builds.mdc
cp skills/depot-github-runners/SKILL.md .cursor/rules/depot-github-runners.mdc
cp skills/depot-ci/SKILL.md .cursor/rules/depot-ci.mdc
cp skills/depot-general/SKILL.md .cursor/rules/depot-general.mdc
```

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

### OpenAI / Codex

Include the contents of relevant SKILL.md files in your custom instructions or system prompt. For Codex CLI, place skill files in your project and reference them in your configuration.

## Documentation

For full Depot documentation, visit [https://depot.dev/docs](https://depot.dev/docs).
