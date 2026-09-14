# Ingress Stack

This stack owns inbound access for the IEEE clubroom server:

- `caddy` for local `*.ieee.local` service names on the clubroom LAN
- `cloudflared` for Cloudflare Tunnel access to selected public services

## Containers

| Container | Purpose |
|---|---|
| `caddy` | Local reverse proxy for `*.ieee.local` hostnames |
| `cloudflared` | Cloudflare Tunnel connector |

## Local Caddy routes

| Hostname | Target |
|---|---|
| `finance.ieee.local` | Akaunting on host port `8081` |
| `pihole.ieee.local` | Pi-hole admin UI on host port `8082` |
| `network.ieee.local` | NetAlertX on host port `20211`, once deployed |
| `dockhand.ieee.local` | Dockhand on host port `3000` |
| `home.ieee.local` | Optional dashboard on host port `8083`, once deployed |

`pos.ieee.local` / canteen is intentionally **not** defined in Caddy. The canteen app is intended to be exposed through Cloudflare Tunnel instead of local Caddy.

## Cloudflare Tunnel model

This stack uses a remotely-managed Cloudflare Tunnel token:

```yaml
command: tunnel --no-autoupdate run --token ${CLOUDFLARED_TUNNEL_TOKEN:?set_in_dockhand_local_env}
```

Set the real `CLOUDFLARED_TUNNEL_TOKEN` only in Dockhand/local overrides. Do not commit it to Git.

Cloudflare public hostname routing should be configured in the Cloudflare dashboard. For the canteen app, point the public hostname to the host-published canteen port, for example:

```text
https://<canteen-domain> -> http://host.docker.internal:8000
```

The Compose file gives `cloudflared` access to `host.docker.internal` so it can reach host-published services without coupling every stack to one shared Docker network.

## HTTP vs HTTPS locally

The local `*.ieee.local` names use plain HTTP:

```caddyfile
auto_https off
```

Reason: `ieee.local` is internal-only and cannot receive normal public Let's Encrypt certificates unless we later use a real domain or internal CA workflow. Public Cloudflare Tunnel routes terminate HTTPS at Cloudflare.

## Deployment

1. Sync the repository in Dockhand.
2. Deploy or update the `ingress` stack.
3. Set `CLOUDFLARED_TUNNEL_TOKEN` in Dockhand/local env before enabling Cloudflare Tunnel.
4. Add/update local DNS records for the `*.ieee.local` names in Pi-hole or the active DHCP/DNS system.
5. Configure public canteen hostname routing in Cloudflare, not in `Caddyfile`.

## Environment

See `.env` for Dockhand-editable defaults.

| Variable | Default | Purpose |
|---|---:|---|
| `CADDY_HTTP_PORT` | `80` | Local HTTP listener |
| `CADDY_HTTPS_PORT` | `443` | Reserved local HTTPS listener |
| `CLOUDFLARED_TUNNEL_TOKEN` | placeholder | Cloudflare Tunnel token; set through Dockhand/local override |

## Persistent volumes

| Volume | Purpose |
|---|---|
| `caddy-data` | Caddy runtime data/cert storage |
| `caddy-config` | Caddy internal config storage |
