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
- Docker-compatible `iptables`/`iptables-nft` forwarding and masquerade rules, if enabled
- `firewalld` disabled/stopped, if enabled
- `/etc/systemd/logind.conf.d/99-ieee-gateway-appliance.conf`, if appliance power settings are enabled
- masked `sleep.target`, `suspend.target`, `hibernate.target`, and `hybrid-sleep.target`, if appliance power settings are enabled
- `/etc/udev/rules.d/99-ieee-no-usb-autosuspend.rules`, if USB autosuspend protection is enabled

It does not intentionally modify university/uplink authentication settings.

## Default reliability settings

The script defaults to enabling the settings that matched the working Fedora test setup:

```text
Disable firewalld: yes, if firewalld is installed
Install Docker-compatible iptables forwarding/NAT rules: yes
Prevent sleep/hibernate: yes
Disable USB autosuspend: yes
```

`firewalld` is useful on many Fedora desktops, but on this dedicated gateway it blocked LAN DHCP during testing and can also interfere with forwarded/NAT traffic. The script's dedicated nftables/iptables rules become the gateway firewall/routing source of truth instead.

Docker can set the iptables `FORWARD` chain policy to `DROP`. The script therefore installs equivalent iptables-nft rules at the top of the `FORWARD` chain in addition to its native nftables rules:

```bash
iptables -I FORWARD 1 -i <LAN_IFACE> -o <WAN_IFACE> -j ACCEPT
iptables -I FORWARD 1 -i <WAN_IFACE> -o <LAN_IFACE> -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -A POSTROUTING -o <WAN_IFACE> -j MASQUERADE
```

These were needed on the Fedora gateway because DHCP worked after disabling firewalld, but client internet required the Docker-compatible iptables forwarding rules.

The power settings should normally stay enabled because this PC is acting like network infrastructure, not a normal desktop. If the machine sleeps or hibernates, DHCP, NAT, DNS, and client internet access stop. Disabling USB autosuspend also helps prevent the USB Ethernet adapter from being power-managed off while it is the LAN-side interface.

## Re-running the script

It is okay to pull a newer version of this repo and run the script again to re-apply the gateway configuration. The script is intended to be idempotent for the normal gateway settings:

- it updates the existing NetworkManager LAN profile named `ieee-clubroom-lan` instead of creating duplicates;
- it overwrites the dedicated dnsmasq, systemd, logind, udev, and apply-script files it owns;
- it deletes and recreates only the dedicated nftables tables named `ieee_gateway*`;
- it removes/reinserts the exact Docker-compatible iptables FORWARD rules it owns so they stay at the top of the chain;
- it backs up touched files under a new `/root/ieee-gateway-backup-<timestamp>/` directory each run.

When re-running on the already-working gateway, select the same WAN/LAN interfaces and same IP/DHCP values unless you deliberately want to change them. The script lists detected interfaces and lets you pick by number or type the interface name directly. It suggests the WAN from the default route and suggests the LAN from an existing `192.168.0.1/24` interface or another connected non-WAN wired/USB interface. Re-running may briefly interrupt clients while dnsmasq and the gateway service restart.

Review `docs/network/rebuild-plan.md` before running it on the real clubroom PC.
