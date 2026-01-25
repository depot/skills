---
name: bake-compose
description: Use depot bake with HCL and Docker Compose
metadata:
  tags: bake, compose, buildx, multi-target
---

## Depot Bake Basics

- `depot bake` builds multiple targets in parallel and deduplicates shared work.
- Default bake file names include `docker-bake.hcl` and `docker-compose.yml`.
- Use `-f` to point at a custom bake or compose file.

## Shared Bases with Contexts

- Use `contexts` in HCL to build a base target once and reuse it.
- Reference the base in Dockerfiles with `FROM base`.

## Docker Compose

- Preferred flow: `depot bake -f docker-compose.yml --load` to build and load locally.
- `docker compose build` works after `depot configure-docker`, but is slower to load.
- Add `x-depot.project-id` per service to shard builds across projects.

## Registry + CI

- Save bake output to Depot Registry with `depot bake --save --metadata-file=build.json`.
- Pull specific targets with `depot pull --target app,db <build-id>`.
- Push a tested target with `depot push --target app --tag registry/app:v1 <build-id>`.
