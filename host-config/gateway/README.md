# Gateway Host Configuration

## `configure-clubroom-gateway.sh`

Interactive helper for configuring a replacement IEEE clubroom gateway PC.

Supported target pattern:

```text
University Ethernet -> Linux mini-PC gateway -> USB Ethernet LAN -> TP-Link AP/switch
```

The script is intended for a fresh/dedicated gateway PC running Debian-like or Fedora-like Linux.

Run from the repo root:

```bash
sudo bash host-config/gateway/configure-clubroom-gateway.sh
```

It prompts before applying changes and backs up touched files under:

```text
/root/ieee-gateway-backup-<timestamp>/
```

It may modify:

- NetworkManager LAN connection profile
- `/etc/dnsmasq.d/ieee-clubroom.conf`
- `/etc/dnsmasq.conf` only to add a conf-dir include if missing
- `/etc/sysctl.d/99-ieee-ipforward.conf`
- `/usr/local/sbin/ieee-gateway-apply.sh`
- `/etc/systemd/system/ieee-gateway.service`
- active nftables tables named `ieee_gateway*`

It does not intentionally modify university/uplink authentication settings.

Review `docs/network/rebuild-plan.md` before running it on the real clubroom PC.
