---
name: core-build-flow
description: Depot CLI usage, project wiring, build/load/push flow, and auth
metadata:
  tags: depot, docker, build, push, registry, auth
---

## Quick Start

```bash
# Install
brew install depot/tap/depot   # macOS
curl -L https://depot.dev/install-cli.sh | sh   # Linux

# Authenticate
depot login

# Initialize project (creates depot.json)
depot init

# Build (drop-in replacement for docker build)
depot build -t myapp:latest .
```

## Project Selection

- Prefer `depot init` to create `depot.json` in the repo root.
- Alternatively set `DEPOT_PROJECT_ID` when you cannot write `depot.json`.

## Core Commands

| Command | Description |
|---------|-------------|
| `depot build` | Build image (drop-in for `docker build`) |
| `depot bake` | Multi-target builds from HCL/Compose |
| `depot init` | Initialize project, creates `depot.json` |
| `depot configure-docker` | Use Depot as default Docker builder |

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

## Docker CLI Integration

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

| Context | Method |
|---------|--------|
| Local dev | `depot login` |
| CI with OIDC | Trust relationship (GitHub Actions, CircleCI, Buildkite) |
| CI with token | `DEPOT_TOKEN` environment variable |
