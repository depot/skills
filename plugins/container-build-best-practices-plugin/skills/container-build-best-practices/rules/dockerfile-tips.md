---
name: dockerfile-tips
description: Dockerfile optimization and cache mount patterns
metadata:
  tags: dockerfile, cache, buildkit, layers, optimization
---

## Cache Mount Patterns

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

## Layering Tips

- Put dependency installs before copying the full source to maximize cache hits.
- Use multi-stage builds to keep runtime images small.
- Add a `.dockerignore` to exclude `node_modules`, build outputs, and secrets.

## Optimal Dockerfiles

- Use Depot's language-specific optimal Dockerfiles as starting points.
- Retrofit cache mounts into existing Dockerfiles using the guide patterns.
- Prefer language guides for Node, Python, Java, .NET, Go, PHP, Ruby, and Rust.

## When to Use Depot Features

- Use cache mounts for package managers to speed cold builds.
- Prefer `depot bake` for multi-target builds to dedupe shared work.
