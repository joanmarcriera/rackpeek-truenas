# rackpeek-truenas

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**[RackPeek](https://github.com/Timmoth/RackPeek) deployment for [TrueNAS SCALE](https://www.truenas.com/truenas-scale/).**

RackPeek is a web UI and CLI tool for documenting home lab infrastructure as YAML. This repo provides a TrueNAS-ready deployment: a `docker-compose.yml` tuned for TrueNAS SCALE, step-by-step Custom App instructions, and a config directory backed by a ZFS dataset so your inventory survives container updates.

---

## Architecture

```
TrueNAS SCALE
└── ZFS pool
    └── /mnt/MainPool/rackpeek/config/   ← persisted on ZFS
        └── config.yaml                  ← your infrastructure YAML

    Docker container: aptacode/rackpeek
    └── port 8080  →  web UI + REST API
        /app/config → ZFS dataset (volume mount)
```

Config lives entirely in a single `config.yaml` file on the ZFS dataset. The container is stateless — update or recreate it at any time without losing inventory.

---

## Quick start on TrueNAS SCALE

### Option A — Custom App (recommended)

1. In the TrueNAS web UI go to **Apps → Discover Apps → Custom App**.
2. Set the image to `aptacode/rackpeek:latest`.
3. Under **Environment Variables**, optionally add:

   | Variable | Value |
   |----------|-------|
   | `RPK_API_KEY` | a secret key to protect the REST API (leave empty to disable) |

4. Under **Storage**, add a host path volume:
   - **Host path:** `/mnt/MainPool/rackpeek/config` *(adjust to your pool name)*
   - **Mount path:** `/app/config`

5. Under **Networking**, add a port forward:
   - Container port `8080` → Host port `8080`

6. Click **Install**. After a few seconds, open `http://truenas-ip:8080` in your browser.

> **First run:** RackPeek creates an empty `config.yaml` in `/app/config` if none exists. Open the web UI and start adding devices.

### Option B — docker compose

```bash
# On TrueNAS, open a shell and clone this repo to a ZFS dataset:
git clone https://github.com/joanmarcriera/rackpeek-truenas.git \
  /mnt/MainPool/rackpeek/deploy
cd /mnt/MainPool/rackpeek/deploy/docker

cp .env.example .env
# Edit .env: set CONFIG_ROOT to your ZFS dataset path

docker compose --env-file .env pull
docker compose --env-file .env up -d
```

Then open `http://truenas-ip:8080`.

### Option C — docker run (one-liner)

```bash
docker run -d \
  --name rackpeek \
  --restart unless-stopped \
  -p 8080:8080 \
  -v /mnt/MainPool/rackpeek/config:/app/config \
  aptacode/rackpeek:latest
```

---

## Environment variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RPK_YAML_DIR` | no | `/app/config` | Internal config directory — do not change unless you remap the volume |
| `RPK_API_KEY` | no | *(empty — API disabled)* | Secret key for REST API endpoints; header `X-Api-Key` |

When `RPK_API_KEY` is empty the REST inventory API returns `503`. The web UI is always available regardless.

---

## Data layout

```
/app/config/               (mapped from your ZFS dataset)
└── config.yaml            single file — full infrastructure inventory
```

RackPeek reads and writes only `config.yaml`. Back up this file to protect your inventory. Because it is plain YAML it is easy to version-control directly.

---

## REST API

The inventory API is available at `http://truenas-ip:8080/api/` when `RPK_API_KEY` is set.

```bash
# List all resources
curl -H "X-Api-Key: $RPK_API_KEY" http://truenas-ip:8080/api/inventory

# Export Ansible inventory
curl -H "X-Api-Key: $RPK_API_KEY" http://truenas-ip:8080/api/ansible-inventory
```

---

## Updating RackPeek

```bash
cd /mnt/MainPool/rackpeek/deploy/docker
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

Config is safe on the ZFS volume — the update only replaces the container image.

---

## Security notes

- Restrict port `8080` to your LAN or use the TrueNAS built-in Nginx reverse proxy for TLS termination if remote access is needed.
- Set `RPK_API_KEY` if you use the REST API. The web UI has no authentication — treat it as an internal tool.
- The container runs as a non-root user and does not require any special Linux capabilities.

---

## License

MIT — see [LICENSE](LICENSE).
