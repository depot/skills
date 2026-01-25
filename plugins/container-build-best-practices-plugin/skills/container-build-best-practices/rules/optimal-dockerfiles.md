---
name: optimal-dockerfiles
description: Use language-specific optimal Dockerfiles and cache mounts
metadata:
  tags: dockerfile, optimization, cache, language
---

## When to Use

- Start with Depot's optimal Dockerfiles for your language.
- Use them as reference implementations or templates.

## Language Guides

- Node.js (npm, pnpm)
- Python (pip, poetry, uv)
- Java (Gradle, Maven)
- .NET (ASP.NET Core, Worker Service)
- Go, PHP (Composer), Ruby (Bundler), Rust

## Cache Mount Guidance

- Cache mounts are the primary optimization for Depot builds.
- Add `--mount=type=cache` for language package managers and build caches.
- Keep the cache mount targets aligned with language defaults.
