# Caddy Stack

Caddy is the reverse proxy for the IEEE clubroom server.

## Purpose

Caddy gives the server stable local service names instead of requiring users to remember host ports.

Planned routes:

| Hostname | Target |
|---|---|
| `finance.ieee.local` | Akaunting on host port `8081` |
| `pihole.ieee.local` | Pi-hole admin UI, once deployed |
| `network.ieee.local` | NetAlertX, once deployed |
| `dockhand.ieee.local` | Dockhand |
| `pos.ieee.local` | IEEE POS app, once deployed |
| `home.ieee.local` | Optional dashboard, once deployed |

## Containers

| Container | Purpose |
|---|---|
| `caddy` | Reverse proxy |

## Initial deployment

1. Deploy this stack through Dockhand.
2. Deploy or confirm the Akaunting stack is available on host port `8081`.
3. Configure Pi-hole/local DNS so `finance.ieee.local` points to the IEEE server PC IP.
4. Visit:

   ```text
   http://finance.ieee.local
   ```

## HTTP vs HTTPS

This initial clubroom deployment uses plain HTTP for `*.ieee.local` names.

Reason: `ieee.local` is an internal-only name and cannot receive a normal public Let's Encrypt certificate unless we later use a real domain or internal CA workflow.

Future HTTPS options:

- Use a real domain and DNS-01 certificates.
- Use an internal CA and install the CA certificate on clubroom devices.
- Keep HTTP because services stay on the trusted local clubroom network.

## Cross-stack routing model

This Caddy stack proxies to services through the Docker host using:

```text
host.docker.internal:<published-port>
```

The Compose file includes:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

This keeps each app stack independent. We can move to a shared Docker network later if we want cleaner container-to-container routing.

## Updating routes

When a new stack is deployed:

1. Confirm the service host port.
2. Uncomment or add the matching route in `Caddyfile`.
3. Commit and push the change.
4. Sync/redeploy the Caddy stack in Dockhand.
5. Add or update the matching Pi-hole local DNS record.

## Environment

See `.env` for configurable host ports. The committed `.env` is a Dockhand template; use Dockhand/local overrides for site-specific changes.

Default ports:

| Variable | Default | Purpose |
|---|---:|---|
| `CADDY_HTTP_PORT` | `80` | HTTP listener |
| `CADDY_HTTPS_PORT` | `443` | Reserved HTTPS listener |

## Persistent volumes

| Volume | Purpose |
|---|---|
| `caddy-data` | Caddy runtime data/cert storage |
| `caddy-config` | Caddy internal config storage |
