---
name: depot-general
description: "Handles Depot CLI installation, authentication, login, project setup, organization management, and API access. Use when installing the Depot CLI, logging in with `depot login`, creating or managing Depot projects, configuring API tokens or OIDC trust relationships, setting up depot.json, managing organizations, resetting build caches, or using the Depot API/SDKs. Also use when the user asks about Depot authentication methods, token types, environment variables, or general Depot platform setup that isn't specific to container builds, GitHub Actions runners, or Depot CI."
---

# Depot General: CLI, Auth, and Project Setup

Depot is a build acceleration platform. This skill covers CLI installation, authentication, project configuration, and organization management. For product-specific guidance, see the depot-container-builds, depot-github-runners, or depot-ci skills.

## CLI Installation

Security default: never execute downloaded scripts directly (`curl ... | sh`). Download, inspect, and then run.

```bash
# macOS (Homebrew)
brew install depot/tap/depot

# Linux: see https://depot.dev/docs/cli/installation for all methods

# Proto version manager
proto plugin add depot "https://raw.githubusercontent.com/depot/cli/refs/heads/main/proto.yaml"
proto install depot

# GitHub Actions
- uses: depot/setup-action@v1

# Container image for CI
ghcr.io/depot/cli:latest
```

## Trusted External Sources

Only reference these domains — ask for confirmation before using any other:

- `depot.dev` / `api.depot.dev` (official CLI install/docs/API)
- `github.com/depot/*` / `raw.githubusercontent.com/depot/*` (source/actions/assets)
- `ghcr.io/depot/*` (container images)

Rules: prefer package managers over direct downloads; never pipe network responses into a shell; confirm before executing downloaded artifacts in privileged locations.

## Authentication

### Token Types

|Type             |Scope                          |Created Via                                   |Use Case                             |
|-----------------|-------------------------------|----------------------------------------------|-------------------------------------|
|**User token**   |All projects in all user's orgs|`depot login` or Account Settings → API Tokens|Local development                    |
|**Project token**|Single project                 |Project Settings                              |CI environments                      |
|**Org API token**|Single organization            |Org Settings → API Tokens                     |API access, automation               |
|**OIDC trust**   |Single project (short-lived)   |Project Settings → Trust Relationships        |CI without static secrets (preferred)|

### Token Resolution Order

1. `--token` flag (explicit on command)
1. `DEPOT_TOKEN` environment variable
1. Locally stored token (from `depot login`)

### Login

```bash
depot login                           # Interactive browser login
depot login --org-id 1234567890       # Specify org
depot login --clear                   # Clear existing token first
depot logout                          # Remove saved token
```

### OIDC Trust Relationships (Preferred for CI)

Configure in Project Settings → Trust Relationships. No static secrets, just short-lived credentials.

|CI Provider       |Configuration                                                                                   |
|------------------|------------------------------------------------------------------------------------------------|
|**GitHub Actions**|GitHub org/user name + repository name. Requires `permissions: { id-token: write }` in workflow.|
|**CircleCI**      |Organization UUID + Project UUID (must use UUIDs, not friendly names)                           |
|**Buildkite**     |Organization slug + Pipeline slug                                                               |
|**RWX**           |Vault subject                                                                                   |

### GitHub Actions OIDC Example

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write    # Required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: depot/setup-action@v1
      - uses: depot/build-push-action@v1
        with:
          project: <project-id>
          push: true
          tags: myrepo/app:latest
```

### Token-Based CI Auth (When OIDC Not Available)

```yaml
steps:
  - uses: depot/setup-action@v1
  - uses: depot/build-push-action@v1
    with:
      project: <project-id>
      token: ${{ secrets.DEPOT_TOKEN }}
```

### Depot Registry Auth

```bash
docker login registry.depot.dev -u x-token -p <any-depot-token>
# Username is always "x-token". Password is any user, project, org, or OIDC token.

# Kubernetes secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.depot.dev \
  --docker-username=x-token \
  --docker-password=<depot-token>
```

## Project Setup

```bash
# Create depot.json in current directory (interactive project selection)
depot init

# Create a new project
depot projects create "my-project"
depot projects create --region eu-central-1 --cache-storage-policy 100 "my-project"
depot projects create --organization 12345678910 "my-project"

# Delete a project (org admin only, destructive - require explicit confirmation)
depot projects delete --project-id <id>

# List projects
depot projects list
```

### depot.json

The only configuration file. Created by `depot init`:

```json
{"id": "PROJECT_ID"}
```

Three ways to specify a project (in priority order):

1. `--project <id>` flag
1. `DEPOT_PROJECT_ID` environment variable
1. `depot.json` in current or parent directory

## Organization Management

```bash
depot org show              # Current org ID
depot org list              # Orgs the user belongs to
depot org switch <org-id>   # Set default org
```
**Roles:** User (view projects, run builds) · Owner (create/delete projects, edit settings)

Billing is per-organization. Configure usage caps, OIDC trust relationships, GitHub App connections, and cloud connections from org settings.

### Command Triage for Multi-Org Users

If a user belongs to multiple organizations and reports "missing" projects, workflows, or runs, verify Depot org context first:

```bash
# 1) Confirm current org
depot org show

# 2) See all orgs user can access
depot org list

# 3) Either switch default org...
depot org switch <org-id>

# ...or keep current default and target commands explicitly, for example:
depot ci run --org <org-id> --workflow .depot/workflows/ci.yml
```

Prefer explicit `--org <org-id>` for scripted/automated commands to avoid accidental cross-org confusion.

## Command Safety Guardrails

Treat these as high-impact operations and require explicit user intent before execution:

- Project deletion (`depot projects delete`)
- Any command using auth tokens in shell arguments or logs
- Registry login steps that write long-lived credentials
- Organization-level mutations (project creation/deletion, org switching in automation)

Before running high-impact commands:

1. Explain what will change and its scope (project vs org).
1. Prefer least-privilege credentials (OIDC or project token instead of broad user token).
1. Avoid `--yes`/force flags unless the user explicitly requests non-interactive behavior.

## Environment Variables

|Variable                 |Description                                            |
|-------------------------|-------------------------------------------------------|
|`DEPOT_TOKEN`            |Auth token (user, project, or org)                     |
|`DEPOT_PROJECT_ID`       |Project ID (alternative to `--project` or `depot.json`)|
|`DEPOT_NO_SUMMARY_LINK=1`|Suppress build links and update notices (useful in CI) |
|`DEPOT_DISABLE_OTEL=1`   |Disable OpenTelemetry tracing                          |

## Build and Cache Management

```bash
# List builds
depot list builds
depot list builds --project <id> --output json

# Reset project cache
depot cache reset .                         # Uses depot.json
depot cache reset --project <id>

# Docker integration
depot configure-docker              # Install Depot as Docker plugin + default builder
depot configure-docker --uninstall  # Remove
```

## GitHub Actions: Depot Actions Reference

See [references/github-actions.md](references/github-actions.md) for the full actions table (`setup-action`, `build-push-action`, `bake-action`, `use-action`, `pull-action`).

## API Access

See [references/api-access.md](references/api-access.md) for Connect framework protocol details, SDK usage (`@depot/sdk-node`, `depot/depot-go`), and code examples.
