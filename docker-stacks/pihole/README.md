# Pi-hole Stack

Pi-hole provides DNS filtering and local DNS records for the IEEE clubroom network.

## Current role

This stack is configured as a safe first deployment while the Linux gateway still uses host `dnsmasq` for DHCP/DNS.

Important default:

```text
Pi-hole web UI:  host port 8082
Pi-hole DNS:     host port 5353
Host dnsmasq:    still owns LAN DHCP/DNS on port 53
```

Do **not** switch Pi-hole to host port `53` until you deliberately migrate DNS away from host `dnsmasq`. Running both on port `53` will conflict.

## Deployment

1. Review `.env` in Dockhand.
2. Change `PIHOLE_WEB_PASSWORD` before deployment.
3. Deploy the stack through Dockhand.
4. Redeploy/reload Caddy so `pihole.ieee.local` points to host port `8082`.
5. Visit:

   ```text
   http://pihole.ieee.local/admin
   ```

   or directly:

   ```text
   http://<gateway-lan-ip>:8082/admin
   ```

## Local DNS records to add

After Pi-hole is running, add local DNS records pointing at the gateway/server LAN IP, usually `192.168.0.1`:

```text
finance.ieee.local   -> 192.168.0.1
pihole.ieee.local    -> 192.168.0.1
dockhand.ieee.local  -> 192.168.0.1
pos.ieee.local       -> 192.168.0.1
network.ieee.local   -> 192.168.0.1
home.ieee.local      -> 192.168.0.1
```

## DNS migration path

Recommended order:

1. Keep host `dnsmasq` serving DHCP.
2. Deploy Pi-hole on DNS host port `5353` and confirm the web UI works.
3. Test DNS manually from a client or the gateway:

   ```bash
   dig @192.168.0.1 -p 5353 pihole.ieee.local
   ```

4. If Pi-hole should become LAN DNS, change Pi-hole to host port `53` only after freeing port `53` on the host or reconfiguring `dnsmasq`.
5. Update DHCP option 6 so clients receive the Pi-hole/listening address.

## Environment

See `.env` for Dockhand-editable defaults.

| Variable | Default | Purpose |
|---|---:|---|
| `PIHOLE_WEB_PORT` | `8082` | Host web UI port for Caddy proxying |
| `PIHOLE_DNS_PORT` | `5353` | Trial DNS host port while dnsmasq owns port 53 |
| `PIHOLE_WEB_PASSWORD` | placeholder | Pi-hole admin password; change before deploy |
| `PIHOLE_DNS_UPSTREAMS` | `8.8.8.8;1.1.1.1` | Upstream DNS resolvers |
| `PIHOLE_DNS_LISTENING_MODE` | `all` | Allows DNS queries via the published host port |

## Persistent volumes

| Volume | Purpose |
|---|---|
| `pihole-etc` | Pi-hole configuration/database |
| `pihole-dnsmasq` | Pi-hole dnsmasq drop-ins |
