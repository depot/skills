# Depot Skills

Skills that teach AI coding agents how to use [Depot](https://depot.dev). Install them into your agent of choice so it can generate correct Depot CLI commands, configurations, and workflows.

## Installation

### skills.sh

```bash
npx skills add depot/skills
```

See [skills.sh](https://skills.sh) for more info.

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

## Available skills

| Skill | Directory | Description |
|-------|-----------|-------------|
| **Container Builds** | [`skills/depot-container-builds/`](skills/depot-container-builds/SKILL.md) | `depot build`, `depot bake`, multi-platform builds, caching, Depot Registry, migration from Docker |
| **GitHub Actions Runners** | [`skills/depot-github-runners/`](skills/depot-github-runners/SKILL.md) | Managed GHA runners, runner labels/sizes, caching, Dagger, Dependabot, egress filtering |
| **Depot CI** | [`skills/depot-ci/`](skills/depot-ci/SKILL.md) | Depot CI beta, `depot ci migrate`, secrets/vars, running workflows, GHA compatibility |
| **General** | [`skills/depot-general/`](skills/depot-general/SKILL.md) | CLI installation, authentication (tokens, OIDC), project setup, org management, API, pricing |

## Documentation

For full Depot documentation, visit [https://depot.dev/docs](https://depot.dev/docs).
