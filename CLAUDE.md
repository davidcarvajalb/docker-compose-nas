# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Docker Compose NAS — an opinionated Docker Compose setup for a self-hosted NAS with 30+ containerized services covering media management (Sonarr, Radarr, Lidarr, Jellyfin), downloads (qBittorrent over PIA VPN), reverse proxy (Traefik with Let's Encrypt SSL), and various utility services. Services are declared in `docker-compose.yml` with optional services loaded via Docker Compose profiles or separate compose files.

## Validation Command

```bash
# The CI check — validates all compose file syntax
cp .env.example .env && docker compose config
```

This is the only automated check (runs on every push via `.github/workflows/main.yml`). There are no unit tests or linters.

## Architecture

- **`docker-compose.yml`**: Main orchestration file. Always-on services are defined here directly; optional services use `profiles:` to stay off by default.
- **Service subdirectories** (e.g. `adguardhome/`, `immich/`, `tandoor/`): Contain their own `docker-compose.yml` fragments loaded via the `COMPOSE_FILE` env var (colon-separated list).
- **`.env` / `.env.example`**: All configuration flows through environment variables — credentials, paths, API keys, feature toggles. `.env` is gitignored; `.env.example` is the template.
- **`update-config.sh`**: Extracts API keys from running service configs and updates `.env`. Also sets URL bases for Sonarr/Radarr/Lidarr/Prowlarr.
- **`homepage/tpl/`**: Go templates that generate Homepage dashboard config from environment variables.
- **Networking**: All services share a single Docker network (`docker-compose-nas`). Traefik routes by path prefix (`/sonarr`, `/radarr`, etc.) using container labels.

## Adding a New Service

Follow the existing pattern:
1. Add the service definition to `docker-compose.yml` (or a new subdirectory `<service>/docker-compose.yml` for optional services loaded via `COMPOSE_FILE`).
2. Use standard labels for Traefik routing and Homepage dashboard integration.
3. Add required environment variables to `.env.example` with sensible defaults.
4. Use `${PUID}`, `${PGID}`, `${TZ}` for consistent user/group/timezone handling.
5. Add a health check following the existing patterns (curl-based, 1min interval, 10 retries).

## Debugging & Troubleshooting

### Volume/env changes require container recreation
- `docker compose restart` does NOT pick up volume or env changes — must use `docker compose up -d <service>` to recreate.
- After changing `DATA_ROOT` or `DOWNLOAD_ROOT`, ALL services using those vars need recreation.

### qBittorrent networking (VPN)
- qBittorrent uses `network_mode: "service:vpn"`, so it shares the VPN container's network stack.
- Other services must reach qBittorrent via hostname `vpn` (not `qbittorrent`), port `8080`.
- After recreating the VPN container, qBittorrent must also be recreated.
- To query qBittorrent API: `docker exec vpn curl ... http://127.0.0.1:8080/api/v2/...`

### VPN Firewall (`FIREWALL=1`)
- Enabling the firewall requires adding the Docker network subnet to `PIA_LOCAL_NETWORK` (e.g., `172.18.0.0/16`), otherwise inter-container communication breaks.
- Check the Docker network subnet with: `docker network inspect docker-compose-nas --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'`

### Service URL bases
- Most services run with a URL base matching their Traefik path prefix: `/sonarr`, `/radarr`, `/prowlarr`, `/jellyfin`, etc.
- Inter-service communication uses container hostnames + URL base: `http://sonarr:8989/sonarr`, `http://radarr:7878/radarr`, `http://prowlarr:9696/prowlarr`, `http://jellyfin:8096/jellyfin`

### Non-LinuxServer.io images
- Seerr does NOT use `PUID`/`PGID` (not a LinuxServer.io image) — may have permission issues on config directories. Fix with `chown`.

### Prowlarr + FlareSolverr
- FlareSolverr requires tags to link it to specific indexers — leaving tags empty disables it.
- FlareSolverr is enabled via the `flaresolverr` profile in `COMPOSE_PROFILES`.

### Torrent troubleshooting
- Torrents stuck in "error" state may need force-start via API: `curl -b cookies "http://127.0.0.1:8080/api/v2/torrents/setForceStart" -d "hashes=all&value=true"`
- "Error" status can be misleading — check actual download speeds and peer counts, not just the status label.

## Interaction Rules

- When the user reports an issue, trust them — they have their reasons. Never dismiss or contradict without first verifying by checking logs, configs, container state, etc.
- Always double-check before saying "no" or "that's fine" — investigate the actual state of things first.

## Commit Convention

Conventional Commits: `type(scope): description` — e.g. `feat(suggestarr): Add Suggestarr service`, `fix: Correct typo in dependabot.yml`.
