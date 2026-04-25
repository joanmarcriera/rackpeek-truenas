# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

TrueNAS SCALE deployment configuration for [RackPeek](https://github.com/Timmoth/RackPeek) — a home lab infrastructure documentation tool. This repo contains no application code: only deployment artifacts (docker-compose, env template) and documentation.

Upstream image: `aptacode/rackpeek:latest` ([Docker Hub](https://hub.docker.com/r/aptacode/rackpeek/))

## Deployment Targets

Three supported deployment paths (all documented in README.md):

1. **TrueNAS SCALE Custom App** — via the web UI (no CLI needed)
2. **docker compose** — `docker compose --env-file docker/.env up -d`
3. **docker run** — one-liner for quick testing

## Key Files

| File | Purpose |
|------|---------|
| `docker/docker-compose.yml` | Compose service definition |
| `docker/.env.example` | Template for user-supplied values; copy to `docker/.env` |
| `README.md` | End-user deployment guide (all three options) |

## Architecture

RackPeek stores its entire state in a single `config.yaml` on the mounted volume (`/app/config` inside the container). There is no database. The ZFS dataset on TrueNAS is mapped to `/app/config` so config survives container updates.

```
TrueNAS ZFS dataset  (/mnt/<pool>/rackpeek/config)
    └── config.yaml
         ↕ volume mount
    /app/config  (inside container)
         ↕
    aptacode/rackpeek  →  port 8080  (web UI + optional REST API)
```

## Environment Variables (RackPeek)

| Variable | Notes |
|----------|-------|
| `RPK_YAML_DIR` | Internal config dir; always `/app/config` when using the compose file |
| `RPK_API_KEY` | Optional; enables REST API with `X-Api-Key` header auth; empty = API disabled (503) |

## Conventions for This Repo

- Do not vendor or copy RackPeek source code here — this repo is deployment-only.
- Keep `docker/docker-compose.yml` minimal; avoid adding services not needed for a basic RackPeek deployment.
- When bumping the upstream image version, update `RACKPEEK_VERSION` in `.env.example` and note the change in a commit message.
- `docker/.env` is gitignored (contains secrets); `.env.example` is the committed template.
- ZFS dataset paths in examples use `/mnt/MainPool/rackpeek/` as a placeholder — the actual pool name varies per installation.

## Updating

```bash
# Pull latest image and recreate container
docker compose --env-file docker/.env pull
docker compose --env-file docker/.env up -d
```

Config on the ZFS volume is unaffected by image updates.
