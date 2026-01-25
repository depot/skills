---
name: registries
description: Depot Registry usage and private registry access
metadata:
  tags: registry, push, pull, auth
---

## Depot Registry

- Save builds with `depot build --save` or `depot bake --save`.
- Use `depot pull` to retrieve builds and `depot push` to promote to another registry.
- Default retention is 7 days; adjust retention in Project Settings when needed.

## Auth for Depot Registry

- Authenticate Docker CLI with username `x-token` and a Depot token.
- Pull tokens are read-only and expire after 1 hour.

## Private Registries

- Depot uses local Docker credentials; run `docker login` before `depot build`.
- If a private base image fails to pull, verify the login and retry.
