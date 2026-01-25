# Depot Best Practices & Troubleshooting

## Table of Contents
- [How Depot Works](#how-depot-works)
- [Build Optimization](#build-optimization)
- [Image Size Reduction](#image-size-reduction)
- [Dockerfile Linting Issues](#dockerfile-linting-issues)
- [Caching Deep Dive](#caching-deep-dive)
- [Multi-Platform Builds](#multi-platform-builds)
- [Troubleshooting](#troubleshooting)

---

## How Depot Works

Depot achieves up to 40x faster builds through three core optimizations:

### 1. Instant Builder Startup
Depot maintains a standby pool of warm machines. Instead of waiting 40s-5min for instance provisioning, builders start in 2-3 seconds. Builders stay active for 2 minutes post-build for rapid iteration.

### 2. Persistent Distributed Caching
Layer cache persists on NVMe SSDs via a Ceph distributed storage cluster:
- Cache throughput: 900 MB/s (vs 140 MB/s with EBS)
- Shared across team members, local dev, and CI
- 50 GB storage per project by default
- No `--cache-from`/`--cache-to` configuration needed

### 3. Native Multi-Architecture
Native x86 and ARM CPUs run builds simultaneously - no QEMU emulation. Each architecture has dedicated cache and instances.

---

## Build Optimization

### Layer Ordering
Order instructions from least to most likely to change:

```dockerfile
# Good: Dependencies change less often than source
COPY package.json package-lock.json ./
RUN npm ci
COPY . .

# Bad: Source changes invalidate dependency cache
COPY . .
RUN npm ci
```

### Cache Mount Strategy
Use BuildKit cache mounts for package manager caches:

```dockerfile
# npm
RUN --mount=type=cache,target=/root/.npm npm ci

# pip
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt

# Go
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build ./cmd/server
```

### Avoid COPY --link
Despite theoretical benefits, `COPY --link` has caching bugs and often increases build times. Use traditional `COPY` instead.

### Multi-Stage Builds
Separate build and runtime stages:

```dockerfile
FROM node:22 AS build
# Build with all dev dependencies
RUN npm ci && npm run build

FROM node:22-slim AS runtime
# Only production artifacts
COPY --from=build /app/dist ./dist
```

### Parallel Stage Execution
BuildKit automatically parallelizes independent stages. Structure your Dockerfile to maximize parallelization:

```dockerfile
FROM base AS deps
RUN npm ci

FROM base AS lint
COPY . .
RUN npm run lint

FROM deps AS build
COPY . .
RUN npm run build
```

---

## Image Size Reduction

### 1. Use .dockerignore
```
node_modules
.git
*.md
Dockerfile*
.env*
dist
coverage
```

### 2. Clear Package Caches
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends package && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

### 3. Use Smaller Base Images
| Base Image | Size |
|------------|------|
| node:22 | ~1 GB |
| node:22-slim | ~200 MB |
| node:22-alpine | ~130 MB |
| distroless | ~20 MB |

### 4. Multi-Stage Builds
Only copy necessary artifacts to final stage:

```dockerfile
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
```

### 5. Use Cache Mounts (not inline caches)
Cache mounts don't add to image size:

```dockerfile
# Good: Cache mount (not in final image)
RUN --mount=type=cache,target=/var/cache/apt apt-get install -y package

# Bad: Cache in layer
RUN apt-get install -y package && rm -rf /var/cache/apt
```

---

## Dockerfile Linting Issues

Top issues found in production Dockerfiles:

### 1. Multiple Consecutive RUN (30% of Dockerfiles)
```dockerfile
# Bad
RUN apt-get update
RUN apt-get install -y curl

# Good
RUN apt-get update && apt-get install -y curl
```

### 2. Unversioned Package Installs (30%)
```dockerfile
# Bad
RUN apt-get install -y curl

# Good
RUN apt-get install -y curl=7.88.*
```

### 3. Missing --no-install-recommends (22%)
```dockerfile
# Bad
RUN apt-get install -y build-essential

# Good
RUN apt-get install -y --no-install-recommends build-essential
```

### 4. pip Cache Not Disabled (18%)
```dockerfile
# Bad
RUN pip install requests

# Good (with cache mount)
RUN --mount=type=cache,target=/root/.cache/pip pip install requests

# Good (without cache mount)
RUN pip install --no-cache-dir requests
```

### 5. Unremoved apt Lists (16%)
```dockerfile
# Bad
RUN apt-get update && apt-get install -y curl

# Good
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
```

### 6. Using cd Instead of WORKDIR (14%)
```dockerfile
# Bad
RUN cd /app && npm install

# Good
WORKDIR /app
RUN npm install
```

### 7. Shell Form for CMD/ENTRYPOINT (12%)
```dockerfile
# Bad (can't handle signals properly)
CMD npm start

# Good
CMD ["npm", "start"]
```

### Enable Linting in Depot
```bash
depot build --lint -t myapp:latest .
```

---

## Caching Deep Dive

### Cache Invalidation Rules

| Instruction | Invalidates When |
|-------------|------------------|
| ENV, CMD, WORKDIR | Instruction string changes |
| COPY, ADD | Any file content changes |
| RUN | Instruction text changes (NOT external resources) |

**Important**: `RUN apt-get update` won't re-fetch if command text unchanged, even with new security patches.

### Cache Sharing with Depot
- Cache is project-scoped by default
- All team members share the same cache
- Local dev and CI share cache
- x86 and ARM have separate caches

### Force Cache Reset
```bash
depot cache reset --project <project-id>
```

---

## Multi-Platform Builds

### Basic Multi-Platform
```bash
depot build --platform linux/amd64,linux/arm64 -t myapp:latest . --push
```

### Supported Platforms
- `linux/amd64` (Intel/AMD)
- `linux/arm64` (ARM64/Apple Silicon)
- `linux/arm/v7` (32-bit ARM)
- `linux/arm/v6` (Older ARM)
- `linux/386` (32-bit x86)

### Loading Multi-Platform Images Locally
Standard Docker can't load multi-platform manifests. Depot handles this:

```bash
# This works with Depot (fails with docker buildx)
depot build --platform linux/amd64,linux/arm64 --load .
```

### Platform-Specific Instructions
```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.23 AS build
ARG TARGETOS TARGETARCH
RUN GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /app ./cmd/server
```

---

## Troubleshooting

### Authentication Issues

**Error**: `unauthorized: authentication required`

1. Check token is set: `echo $DEPOT_TOKEN`
2. Verify project access: `depot projects list`
3. Re-authenticate: `depot login`

### Private Registry Access

Depot uses local Docker credentials. Ensure you're logged in:

```bash
docker login registry.example.com
depot build .
```

### Slow Builds

1. **Check cache hit rate**: Look for "CACHED" in build output
2. **Review layer ordering**: Dependencies should come before source code
3. **Add cache mounts**: For package manager caches
4. **Use multi-stage**: Separate build and runtime

### Build Hangs

1. **Check build logs**: `depot list builds --project <id>`
2. **Verify network**: Ensure registry/dependency sources are accessible
3. **Reset cache**: `depot cache reset --project <id>`

### Multi-Platform Issues

**Error**: `cannot export to multiple platforms without push or save`

Multi-platform builds must go somewhere:
```bash
depot build --platform linux/amd64,linux/arm64 --push .  # Push to registry
depot build --platform linux/amd64,linux/arm64 --save .  # Save to Depot Registry
depot build --platform linux/amd64,linux/arm64 --load .  # Depot handles this
```

### Cache Not Working

1. **Verify project**: Ensure same `--project` or `DEPOT_PROJECT_ID`
2. **Check layer invalidation**: Small source changes can cascade
3. **Review Dockerfile order**: Frequently changing content should be last

### OIDC Authentication Failed (CI)

For GitHub Actions:
```yaml
permissions:
  contents: read
  id-token: write  # Required for OIDC
```

Verify trust relationship is configured in Depot project settings.
