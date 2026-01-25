---
name: troubleshooting
description: Diagnose slow builds, cache misses, and common errors
metadata:
  tags: troubleshooting, cache, ci, build failures
---

## Slow Builds

- Confirm the build runs with `depot build` (not `docker build`).
- Verify cache mounts are present for package managers.
- Check for overly broad `COPY . .` before dependency installs.

## Cache Misses

- Ensure the Dockerfile avoids changing files earlier than needed.
- Make sure CI uses the same Depot project for shared caches.

## Common Errors

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
