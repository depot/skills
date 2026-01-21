---
name: depot
description: Build Docker and container images up to 40x faster using Depot's remote builders with automatic layer caching. Use when users want to build container images, create Dockerfiles, set up CI/CD for container builds, optimize Docker build performance, or troubleshoot slow builds. Triggers include "docker build", "container image", "Dockerfile", "multi-platform build", "arm64 build", "build cache", slow builds, or CI/CD container pipelines.
---

# Depot Container Builds

Depot provides remote container builders with persistent layer caching on NVMe SSDs, native multi-platform support (no QEMU), and seamless CI/CD integration.

## Quick Start

```bash
# Install
brew install depot/tap/depot   # macOS
curl -L https://depot.dev/install-cli.sh | sh   # Linux

# Authenticate
depot login

# Build (drop-in replacement for docker build)
depot build -t myapp:latest .
```

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

## Key Advantages

### Automatic Caching
- Layer cache persists on fast NVMe storage
- Shared across your entire team by project
- No `--cache-from`/`--cache-to` configuration needed
- 900 MB/s cache throughput (vs 140 MB/s with EBS)

### Native Multi-Platform
- Native x86 and ARM builders (no QEMU emulation)
- Builds run in parallel on dedicated hardware
- Support for: `linux/amd64`, `linux/arm64`, `linux/arm/v7`, `linux/arm/v6`

### Instant Startup
- Warm builder pool starts in 2-3 seconds
- Builders stay active for 2 minutes for rapid iteration

## Depot Registry

Build once, deploy anywhere:

```bash
# Build and save
depot build -t myapp:v1.0 . --save --metadata-file=build.json

# Pull by build ID
BUILD_ID=$(jq -r '.["depot.build"]' build.json)
depot pull --project <id> -t myapp:v1.0 $BUILD_ID

# Push to production registry later
depot push --project <id> -t prod.registry.io/myapp:v1.0 $BUILD_ID
```

## Docker Bake (Multi-Target)

```bash
# From HCL file
depot bake -f docker-bake.hcl

# From docker-compose.yml
depot bake -f docker-compose.yml --load
```

Bake builds all targets in parallel with automatic deduplication of shared work.

## CI/CD Quick Start

### GitHub Actions (OIDC)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: depot/build-push-action@v1
        with:
          project: <project-id>
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

For all CI platforms (CircleCI, GitLab, Jenkins, etc.), see [ci-integration.md](references/ci-integration.md).

## Authentication

| Context | Method |
|---------|--------|
| Local dev | `depot login` |
| CI with OIDC | Trust relationship (GitHub Actions, CircleCI, Buildkite) |
| CI with token | `DEPOT_TOKEN` environment variable |

## Reference Documentation

| Document | Use When |
|----------|----------|
| [dockerfiles.md](references/dockerfiles.md) | Writing/optimizing Dockerfiles with cache mounts for Node, Python, Go, Rust, etc. |
| [ci-integration.md](references/ci-integration.md) | Setting up CI/CD pipelines (GitHub Actions, CircleCI, GitLab, etc.) |
| [best-practices.md](references/best-practices.md) | Optimizing builds, reducing image size, troubleshooting issues |
| [cli-reference.md](references/cli-reference.md) | Full CLI command reference and flags |

## Common Patterns

### Dockerfile with Cache Mounts

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

See [dockerfiles.md](references/dockerfiles.md) for complete patterns.

### Build Arguments and Secrets

```bash
# Build args
depot build --build-arg VERSION=1.0.0 -t myapp:1.0.0 . --push

# Secrets from file
depot build --secret id=npmrc,src=.npmrc -t myapp:latest .

# Secrets from environment
depot build --secret id=token,env=GITHUB_TOKEN -t myapp:latest .
```

### Private Registries

Depot uses your local Docker credentials:

```bash
docker login registry.example.com
depot build -t registry.example.com/myapp:latest . --push
```
