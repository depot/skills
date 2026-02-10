---
name: "depot"
displayName: "Build containers with Depot"
description: "Write optimized Dockerfiles, configure CI/CD pipelines, debug failed builds, and set up multi-platform container builds with Depot"
keywords:
  [
    "depot",
    "docker",
    "container",
    "dockerfile",
    "container image",
    "multi-platform",
    "arm64",
    "build cache",
    "layer caching",
    "CI/CD",
    "build logs",
    "build debugging",
  ]
author: "Depot"
---

## Overview

Depot provides remote container builders with persistent caching. It is a drop-in replacement for `docker build` and `docker buildx build` that runs builds on fast cloud infrastructure with automatic layer caching across builds.

## Relevant Files

The following files are relevant to Depot, you should familiarize yourself with them:

- `depot.json` - Depot project configuration
- `**/Dockerfile*`, `**/*.dockerfile` - Dockerfiles
- `**/docker-bake.hcl`, `**/docker-compose*.yml` - Bake and Compose configs
- `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `.buildkite/*.yml` - CI/CD workflows

## Installation and setup

If the `depot` CLI is not installed, it can be installed via the following:

```bash
# macOS
brew install depot/tap/depot
# Linux
curl -L https://depot.dev/install-cli.sh | sh
# Windows (PowerShell)
iwr https://depot.dev/install-cli.ps1 -useb | iex
```

The CLI needs two things to perform a remote build, the user needs to be authenticated and the CLI needs a Depot project ID:

```bash
# Authenticate the user interactively via their browser
depot login

# Interactively initialize project (creates depot.json)
depot init
```

Project IDs can also be set via the `DEPOT_PROJECT_ID` environment variable or as a `--project PROJECT_ID` flag. The project ID can be found in the Depot dashboard or by running `depot projects list`. In CI, authentication is provided via an OIDC integration (GitHub Actions, CircleCI, Buildkite, etc.) or an API token set as the `DEPOT_TOKEN` environment variable.

## Core Commands

The Depot CLI embes the Docker Buildx library, so the `depot build` command is a drop-in replacement for `docker build` and `docker buildx build`, and the `depot bake` command is a drop-in replacement for `docker buildx build`.

| Command               | Description                                                             |
| --------------------- | ----------------------------------------------------------------------- |
| `depot build`         | Build image (drop-in for `docker buildx build`)                         |
| `depot bake`          | Multi-target builds from HCL/Compose (drop-in for `docker buildx bake`) |
| `depot init`          | Initialize project, creates `depot.json`                                |
| `depot projects list` | List all Depot projects                                                 |

## Build Workflow

```bash
# Build and load to local Docker
depot build -t myapp:latest . --load

# Build and push to registry
depot build -t ghcr.io/org/myapp:v1.0 . --push

# Multi-platform (native ARM + x86, no emulation)
depot build --platform linux/amd64,linux/arm64 -t myapp:latest . --push

# Save to Depot Registry (no external registry needed)
depot build -t myapp:latest . --save
```

### Docker CLI Integration

Run `depot configure-docker` once to route `docker build` and `docker buildx build` through Depot. Look for `[depot]` prefixed logs to confirm the build is using Depot.

## Depot Registry

Build once, deploy anywhere:

```bash
# Build and save
depot build -t myapp:v1.0 . --save --metadata-file=build.json

# Pull by build ID
BUILD_ID=$(jq -r '."depot.build"' build.json)
depot pull --project <id> -t myapp:v1.0 $BUILD_ID

# Push to production registry later
depot push --project <id> -t prod.registry.io/myapp:v1.0 $BUILD_ID
```

## Authentication

| Context       | Method                                                   |
| ------------- | -------------------------------------------------------- |
| Local dev     | `depot login`                                            |
| CI with OIDC  | Trust relationship (GitHub Actions, CircleCI, Buildkite) |
| CI with token | `DEPOT_TOKEN` environment variable                       |

Verify authentication with `depot projects list`.

### OIDC Support by Provider

| Provider            | OIDC Support | Setup                                                             |
| ------------------- | ------------ | ----------------------------------------------------------------- |
| GitHub Actions      | Yes          | Add trust relationship in Depot, use `id-token: write` permission |
| CircleCI            | Yes          | Add trust relationship in Depot project settings                  |
| Buildkite           | Yes          | Add trust relationship in Depot project settings                  |
| GitLab CI           | No           | Use project token                                                 |
| Jenkins             | No           | Use project token                                                 |
| AWS CodeBuild       | No           | Use project token                                                 |
| Bitbucket Pipelines | No           | Use project token                                                 |
| Travis CI           | No           | Use project token                                                 |

### GitHub Actions with OIDC

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write # Required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: depot/build-push-action@v1
        with:
          context: .
          tags: myapp:latest
```

### CircleCI with OIDC

```yaml
jobs:
  build:
    docker:
      - image: cimg/base:current
    steps:
      - checkout
      - run:
          name: Install Depot
          command: curl -L https://depot.dev/install-cli.sh | sh
      - run:
          name: Build
          command: depot build -t myapp:latest .
```

### Project Tokens (For Providers Without OIDC)

1. Go to Depot project settings → Tokens
2. Create a new project token
3. Add as `DEPOT_TOKEN` secret in your CI provider

```yaml
# GitLab CI example
build:
  image: alpine
  before_script:
    - curl -L https://depot.dev/install-cli.sh | sh
  script:
    - depot build -t myapp:latest .
  variables:
    DEPOT_TOKEN: $DEPOT_TOKEN
```

```groovy
// Jenkins example
pipeline {
    environment {
        DEPOT_TOKEN = credentials('depot-token')
    }
    stages {
        stage('Build') {
            steps {
                sh 'curl -L https://depot.dev/install-cli.sh | sh'
                sh 'depot build -t myapp:latest .'
            }
        }
    }
}
```

## Dockerfile Optimization

### Cache Mount Patterns

```dockerfile
# Node.js
RUN --mount=type=cache,target=/root/.npm npm ci

# Python
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

# Go
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build ./cmd/server
```

### Layering Tips

- Put dependency installs before copying the full source to maximize cache hits.
- Use multi-stage builds to keep runtime images small.
- Add a `.dockerignore` to exclude `node_modules`, build outputs, and secrets.
- Use Depot's language-specific optimal Dockerfiles as starting points.
- Use cache mounts for package managers to speed cold builds.
- Prefer `depot bake` for multi-target builds to dedupe shared work.

## Troubleshooting

### Slow Builds

- Confirm the build runs with `depot build` (not `docker build`).
- Verify cache mounts are present for package managers.
- Check for overly broad `COPY . .` before dependency installs.

### Cache Misses

- Ensure the Dockerfile avoids changing files earlier than needed.
- Make sure CI uses the same Depot project for shared caches.

### Common Errors

- `depot: command not found`: Ensure install location is in PATH, or run with full path.
- `401 Unauthorized`: Run `depot login` again, or check `DEPOT_TOKEN` is set correctly.
- `No project found`: Run `depot init` or set `DEPOT_PROJECT_ID` environment variable.
- OIDC fails in CI: Verify trust relationship matches repo/org, check required permissions.
- `Keep alive ping failed to receive ACK within timeout`: Scale up builder size or enable autoscaling.
- `Unable to acquire machine, please retry`: Check `status.depot.dev`, then scale up or enable autoscaling.
- `Our services aren't available right now` with `cache-to/from type=gha`: Remove `--cache-from/--cache-to type=gha` and rely on Depot cache.
- `failed to mount /tmp/buildkit-mount`: Reset build cache in project settings.
- `401 Unauthorized` pulling from Docker Hub: Check Docker Hub status, use AWS mirror, or authenticate.
- `failed to load ref`: Increase cache storage/retention policy if it persists.
- `.git directory not found in build context`: Set `BUILDKIT_CONTEXT_KEEP_GIT_DIR=1` in build args.
- `cannot merge resource due to conflicting Schema URL`: Set `DEPOT_DISABLE_OTEL=1`.
- `remote error: tls: bad record MAC`: Network instability; retry or run in CI.
- `timed out connecting to machine: failed to create temp file`: Check temp dir permissions/disk space.
- Multi-arch shows `unknown/unknown`: Disable provenance with `--provenance false`.
- Private registry pulls fail: `docker login` before `depot build`.

### Common Build Failure Patterns

| Log Pattern                             | Likely Cause                    | Solution                                |
| --------------------------------------- | ------------------------------- | --------------------------------------- |
| `COPY failed: file not found`           | File missing from build context | Check `.dockerignore`, verify file path |
| `RUN npm install` fails                 | Network or dependency issue     | Add cache mount, check package.json     |
| `OOMKilled`                             | Out of memory                   | Increase builder size or optimize build |
| `unauthorized: authentication required` | Registry auth failed            | Run `docker login` before build         |
| `context canceled`                      | Build timeout or interruption   | Check for resource constraints          |

## Bake and Docker Compose

- `depot bake` builds multiple targets in parallel and deduplicates shared work.
- Default bake file names include `docker-bake.hcl` and `docker-compose.yml`. Use `-f` for custom files.
- Use `contexts` in HCL to build a base target once and reuse it across targets.
- Preferred Compose flow: `depot bake -f docker-compose.yml --load` to build and load locally.
- `docker compose build` works after `depot configure-docker`, but is slower to load.
- Save bake output to Depot Registry with `depot bake --save --metadata-file=build.json`.

## Build Parallelism

- BuildKit runs independent stages in parallel when dependencies allow.
- Multi-platform builds run on separate native builders per architecture.
- Concurrent builds on the same project share layers and deduplicate work.
- Prefer a larger single builder first to keep cache efficiency high.
- Autoscaled builders use cache clones that do not write back to the main cache.
- For resource-heavy builds, use autoscaling with 2-3 builds per instance.

## Observability

- Build logs show step status with `cached` or `deduplicated` badges.
- Filter steps by status or search step names to find slow stages.
- Build details show resource utilization and OOM warnings.
- Use the build list charts to spot sudden cache or duration changes.
- Inspect build context size when builds spike unexpectedly.

## Debugging Failed Builds

### Find Projects and Builds

```bash
# List all projects
depot projects list

# List builds for a project
depot list builds --project <project-id>

# JSON output for scripting
depot projects list --output json
depot list builds --project <project-id> --output json
```

**Get build ID from build output:**

Look for: `Build: https://depot.dev/orgs/<org>/projects/<project>/builds/<build-id>`

**Get build ID from metadata file:**

```bash
depot build --metadata-file=build.json -t myapp:latest .
jq -r '."depot.build"' build.json
```

### Debug via Web UI

1. Open the build URL in the Depot dashboard
2. Failed steps are highlighted in red
3. Click a step to expand its logs
4. Use **Share build** to share with teammates

## Optimal Dockerfiles

Depot provides language-specific optimal Dockerfiles for Node.js (npm, pnpm), Python (pip, poetry, uv), Java (Gradle, Maven), .NET, Go, PHP, Ruby, and Rust. See the [optimal Dockerfiles docs](https://depot.dev/docs/container-builds/optimal-dockerfiles/overview) for templates and cache mount patterns.

## Registries

- Save builds with `--save`, retrieve with `depot pull`, promote with `depot push`.
- Default retention is 7 days; adjust in Project Settings.
- Authenticate Docker CLI with username `x-token` and a Depot token for Depot Registry access.
- Depot uses local Docker credentials for private registries; run `docker login` before `depot build`.

## Reference Documentation

For detailed reference, fetch from the live docs at https://depot.dev/docs:

- [Optimal Dockerfiles](https://depot.dev/docs/container-builds/optimal-dockerfiles/overview)
- [CI/CD integrations](https://depot.dev/docs/container-builds/integrations/github-actions)
- [CLI reference](https://depot.dev/docs/cli/reference)
- [Troubleshooting](https://depot.dev/docs/container-builds/troubleshooting)
- [Full docs index](https://depot.dev/llms.txt)
