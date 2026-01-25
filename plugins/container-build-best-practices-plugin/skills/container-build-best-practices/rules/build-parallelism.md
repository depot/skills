---
name: build-parallelism
description: BuildKit parallelism, deduplication, and autoscaling tradeoffs
metadata:
  tags: buildkit, parallelism, autoscaling, deduplication
---

## How Parallelism Works

- BuildKit runs independent stages in parallel when dependencies allow.
- Multi-platform builds run on separate native builders per architecture.
- Concurrent builds on the same project share layers and deduplicate work.

## When to Autoscale

- Use autoscaling for high concurrency or resource-heavy builds.
- Prefer a larger single builder first to keep cache efficiency high.
- Autoscaled builders use cache clones that do not write back to the main cache.

## Tuning Guidance

- For frequent small builds, favor a larger builder with no autoscaling.
- For resource-heavy builds, use autoscaling with 2-3 builds per instance.
- For mixed workloads, split images into separate projects for isolation.

## Bake + Autoscaling

- A single `depot bake` counts as one build for autoscaling.
- Split bake targets across projects when you need dedicated resources.
