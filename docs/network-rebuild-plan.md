# IEEE Clubroom Network Rebuild Plan

This document explains how to recreate the existing IEEE clubroom network on a replacement mini-PC and how the design can be adapted to Fedora Desktop Linux.

## Goal

Recreate the working network pattern:

```text
University authenticated Ethernet
        ↓
Mini-PC gateway/router
        ↓ USB Ethernet LAN
TP-Link TL-WR740N as access point/switch
        ↓
IEEE clubroom devices + printer
```

The mini-PC must be the only device directly connected/authenticated to the university network. Clubroom devices should sit behind a private NATed LAN.

## Target design

| Function | Target implementation |
|---|---|
| WAN/uplink | Built-in Ethernet, DHCP from university network |
| LAN/downstream | USB Ethernet adapter, static `192.168.0.1/24` |
| Routing | Linux IPv4 forwarding |
| NAT | Masquerade outbound traffic through WAN interface |
| DHCP | `dnsmasq` initially, Pi-hole later if desired |
| DNS | `dnsmasq` initially; Pi-hole later for local DNS/filtering |
| Wi-Fi | TP-Link TL-WR740N in AP/dumb switch mode |
| Container services | Dockhand, Caddy, Akaunting, Pi-hole, future POS |

## Recommended IP plan

| Item | Address/range |
|---|---|
| Clubroom LAN | `192.168.0.0/24` |
| Mini-PC LAN gateway | `192.168.0.1` |
| TP-Link AP management IP | `192.168.0.2` |
| Printer suggested reservation | `192.168.0.10` or existing lease-derived address |
| DHCP dynamic range | `192.168.0.50`–`192.168.0.150` |
| Priority/reserved devices | Keep only if still needed |

## Hardware wiring

Preferred wiring:

```text
University roof Ethernet -> mini-PC built-in Ethernet/WAN
mini-PC USB Ethernet     -> TP-Link LAN port
TP-Link LAN/Wi-Fi        -> printer and client devices
```

Use a TP-Link **LAN** port for the cable from the mini-PC. Avoid the TP-Link WAN port unless intentionally creating a second NAT layer.

## TP-Link TL-WR740N settings

Configure the TP-Link as a dumb AP/switch:

```text
Mode: Access Point, if available
DHCP server: disabled
LAN IP: 192.168.0.2
Gateway: 192.168.0.1
Wi-Fi SSID/password: club-defined
WAN port: unused unless AP mode explicitly repurposes it
```

If the old firmware has no AP mode:

1. Disable DHCP on the TP-Link.
2. Set TP-Link LAN IP to `192.168.0.2`.
3. Connect the mini-PC USB Ethernet to a TP-Link LAN port.
4. Do not use the TP-Link WAN port.

## Can this be recreated on Fedora Desktop Linux?

Yes. The design is not Debian-specific. Fedora Desktop can run the same gateway pattern:

```text
NetworkManager -> WAN DHCP and optional LAN static profile
sysctl         -> IPv4 forwarding
nftables       -> NAT/forwarding rules
iptables-nft   -> optional compatibility layer
iptables-services -> optional if preserving legacy iptables-save style
Dnsmasq        -> DHCP/DNS
systemd        -> boot persistence
Docker/Dockhand -> app stack deployment
```

The main difference is configuration style:

| Area | Debian current setup | Fedora Desktop suggested setup |
|---|---|---|
| WAN | NetworkManager DHCP | NetworkManager DHCP |
| LAN | systemd-networkd static config | NetworkManager static profile, or systemd-networkd if deliberately enabled |
| Firewall | iptables-nft restored by custom service | Native nftables/custom service is cleaner; firewalld may need coordination |
| DHCP/DNS | dnsmasq system service | dnsmasq system service |
| Containers | Docker/Dockhand | Docker/Dockhand works if installed |

For Fedora Desktop, the simplest reliable approach is to let NetworkManager own both physical NICs:

- WAN connection: DHCP, default route allowed.
- LAN connection: static `192.168.0.1/24`, never default route.

This avoids mixing NetworkManager and systemd-networkd on Fedora Desktop. The install script in this repo defaults to NetworkManager-based LAN setup for cross-distro compatibility.

## Firewall note on Fedora

Fedora often uses `firewalld` by default. The provided script uses its own nftables tables and prompts before touching firewalld. Do not blindly disable firewalld on a machine that already has other firewall policy.

If firewalld remains active, verify it does not block forwarding/NAT. For a clean dedicated gateway appliance, using a documented nftables ruleset/service is easier to reproduce than ad-hoc firewalld state.

## Rebuild phases

### Phase 1: Identify interface names

On the new PC:

```bash
ip -br link
ip -br addr
nmcli device status
```

Identify:

| Role | Example old name | New name placeholder |
|---|---|---|
| WAN/university uplink | `eno1` | `<WAN_IFACE>` |
| LAN/USB Ethernet | `enxc8a362d428b3` | `<LAN_IFACE>` |

Do not assume interface names match the old PC.

### Phase 2: Configure WAN

WAN should use DHCP from the university network.

Expected behavior:

```text
<WAN_IFACE> has a university IP
Default route is via <WAN_IFACE>
DNS/search domain may come from university DHCP
```

### Phase 3: Configure LAN

LAN should be static:

```text
<LAN_IFACE>: 192.168.0.1/24
No default route on LAN
```

### Phase 4: Enable forwarding

Persistent sysctl:

```ini
net.ipv4.ip_forward=1
```

### Phase 5: Configure DHCP/DNS

Initial dnsmasq model:

```ini
interface=<LAN_IFACE>
bind-interfaces
except-interface=<WAN_IFACE>
dhcp-range=192.168.0.50,192.168.0.150,12h
dhcp-option=3,192.168.0.1
dhcp-option=6,8.8.8.8,1.1.1.1
```

If Pi-hole replaces dnsmasq later, DHCP option 6 should usually become the Pi-hole/listening address, commonly `192.168.0.1`.

### Phase 6: Configure NAT and forwarding

Equivalent policy:

```text
allow established/related WAN -> LAN return traffic
allow LAN -> WAN traffic
masquerade outbound through WAN
```

Native nftables model:

```nft
table inet ieee_gateway {
  chain forward {
    type filter hook forward priority filter; policy accept;
    ct state established,related iifname "<WAN_IFACE>" oifname "<LAN_IFACE>" accept
    iifname "<LAN_IFACE>" oifname "<WAN_IFACE>" accept
  }
}

table ip ieee_gateway_nat {
  chain postrouting {
    type nat hook postrouting priority srcnat; policy accept;
    oifname "<WAN_IFACE>" masquerade
  }
}
```

### Phase 7: Optional QoS

The old machine prioritizes:

```text
192.168.0.111
192.168.0.128
```

with packet mark `10` and a `tc prio` queue on the WAN interface.

Only recreate this if those devices still matter. The old notes identify them as Leif's Framework laptop and the lounge Chromecast.

### Phase 8: Reboot validation

After reboot, without logging in:

```bash
ip -br addr
ip route
systemctl status dnsmasq --no-pager
systemctl status ieee-gateway --no-pager
cat /proc/sys/net/ipv4/ip_forward
sudo nft list ruleset
```

From a Wi-Fi client:

```text
IP:      192.168.0.x
Gateway: 192.168.0.1
DNS:     expected configured DNS
Internet works
Printer reachable
```

## Pi-hole migration note

The current old machine uses host dnsmasq. Pi-hole also embeds dnsmasq/FTL and wants ports 53 and optionally 67.

Do not run host dnsmasq and Pi-hole DHCP/DNS on the same host/IP/ports simultaneously.

Recommended approach:

1. Recreate working gateway with host dnsmasq first.
2. Confirm internet, DHCP, and printer work.
3. Deploy Dockhand and Docker app stacks.
4. Later migrate DHCP/DNS from host dnsmasq to Pi-hole as a deliberate change.
5. Verify clients receive the expected gateway/DNS and local service names resolve.

## Security/authorization caution

This design NATs a private clubroom network behind one university-authenticated host. Confirm the university permits this kind of shared/NATed access point. Keep DHCP isolated to the private LAN side and never bridge DHCP into the university network.
