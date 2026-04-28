---
name: tailscale
description: >
  Use when the user asks about Tailscale, wants to enable/disable remote access,
  check Tailscale status, expose or unexpose a service via Tailscale, or manage
  the Tailscale container. Triggers on mentions of "tailscale", "remote access",
  "expose externally", "VPN mesh", or "access from outside".
argument-hint: <action: status|start|stop|expose|unexpose|remove>
allowed-tools: [Read, Grep, Glob, Bash]
---

# Tailscale Management

Tailscale is an experimental, off-by-default service for remote access via mesh VPN.
Container name: `tailscale`, profile: `tailscale`.
State is persisted at `${CONFIG_ROOT}/tailscale/state` — auth key is only needed on first run.

## Reference: Services and Ports

| Service     | Internal URL                        |
|-------------|-------------------------------------|
| Jellyfin    | `http://jellyfin:8096`              |
| Sonarr      | `http://sonarr:8989`                |
| Radarr      | `http://radarr:7878`                |
| Lidarr      | `http://lidarr:8686`                |
| Prowlarr    | `http://prowlarr:9696`              |
| Bazarr      | `http://bazarr:6767`                |
| Seerr       | `http://seerr:5055`                 |
| Homepage    | `http://homepage:3000`              |
| qBittorrent | `http://vpn:8080` (via VPN container) |

## Operations

### Status

First check if the container is running:

```bash
docker ps --filter name=tailscale --format '{{.Names}} {{.Status}}'
```

If running, check Tailscale connection and served routes:

```bash
docker exec tailscale tailscale status
docker exec tailscale tailscale serve status
```

Report: connection state, tailnet hostname, peers, and which services are currently exposed.

### Start

```bash
docker compose --profile tailscale up -d tailscale
```

Then verify it connected:

```bash
docker exec tailscale tailscale status
```

If it shows "NeedsLogin", the user needs to set `TS_AUTHKEY` in `.env` or authenticate manually:

```bash
docker exec tailscale tailscale up
```

This will print a login URL.

### Stop

```bash
docker compose --profile tailscale stop tailscale
```

### Expose a Service

There are two modes:
- **Funnel** (public internet — anyone can access, no Tailscale client needed) — this is the default/expected mode
- **Serve** (tailnet only — only devices with Tailscale client installed)

Before exposing, verify the container is running.

#### Funnel (public tunnel)

```bash
docker exec tailscale tailscale funnel --bg <internal-url>
```

Example for Jellyfin:

```bash
docker exec tailscale tailscale funnel --bg http://jellyfin:8096
```

To expose on a specific path:

```bash
docker exec tailscale tailscale funnel --bg --set-path /jellyfin http://jellyfin:8096
```

Funnel requires enabling in Tailscale admin console under DNS > Funnel. If it fails with a policy error, tell the user to enable it there.

#### Serve (tailnet only)

```bash
docker exec tailscale tailscale serve --bg <internal-url>
```

#### Check what's exposed

After exposing, show the URL and mode:

```bash
docker exec tailscale tailscale funnel status
```

### Unexpose

Reset all routes (both funnel and serve):

```bash
docker exec tailscale tailscale funnel reset
```

Or remove a specific path:

```bash
docker exec tailscale tailscale funnel off /jellyfin
```

### Remove Entirely

Guide the user through these steps (do NOT execute destructive steps without confirmation):

1. Stop the container: `docker compose --profile tailscale stop tailscale`
2. Remove `tailscale/docker-compose.yml` from `COMPOSE_FILE` in `.env`
3. Remove `TS_AUTHKEY` and `TS_HOSTNAME` from `.env`
4. Delete the `tailscale/` directory
5. Optionally: remove the node from the Tailscale admin console at https://login.tailscale.com/admin/machines

## Troubleshooting

- **"NeedsLogin"**: Auth key expired or not set. Generate a new one at https://login.tailscale.com/admin/settings/keys
- **Can't reach services**: Make sure the tailscale container is on the `docker-compose-nas` network (it inherits this automatically)
- **Serve not working**: Tailscale Serve requires HTTPS certificates from Tailscale. Ensure MagicDNS is enabled in the admin console
- **Funnel policy error**: Funnel must be enabled in admin console at DNS > Funnel (https://login.tailscale.com/admin/dns). It's off by default
- **Funnel not reachable**: Funnel only works on ports 443, 8443, and 10000. Default (443) is fine for most cases
