# Current IEEE Clubroom Network Audit

This document records the reverse-engineered network configuration from the existing IEEE clubroom Debian mini-PC, based on `ieee-network-audit.txt` collected on 2026-09-03.

## Host

| Field | Value |
|---|---|
| Hostname | `ece-ieee-mcn-01` |
| OS | Debian GNU/Linux 13 `trixie` |
| Kernel | `6.12.105+deb13-amd64` |
| User used for audit | `ieee` |

## Physical topology

```text
University Ethernet drop
        ↓
Debian mini-PC: ece-ieee-mcn-01
        ↓ USB Ethernet adapter
TP-Link TL-WR740N router/AP
        ↓
IEEE clubroom Wi-Fi clients + printer
```

The Debian mini-PC is the actual gateway/router. The TP-Link device is used for Wi-Fi/AP/switching.

## Interface roles

| Interface | MAC | Address | Manager | Role |
|---|---|---|---|---|
| `eno1` | `00:23:24:70:c4:c5` | `10.19.1.233/17` | NetworkManager | WAN/uplink to university network |
| `enxc8a362d428b3` | `c8:a3:62:d4:28:b3` | `192.168.0.1/24` | systemd-networkd | Private clubroom LAN |
| `wlp2s0` | `d6:8e:8e:a9:63:6e` runtime / `b8:8a:60:25:36:24` permanent | none/down | none active | Not used for AP |
| `lo` | n/a | `127.0.0.1/8`, `::1/128` | local | loopback |

WAN routing:

```text
default via 10.19.0.1 dev eno1 proto dhcp src 10.19.1.233 metric 100
10.19.0.0/17 dev eno1 proto kernel scope link src 10.19.1.233 metric 100
192.168.0.0/24 dev enxc8a362d428b3 proto kernel scope link src 192.168.0.1
```

University DNS received by NetworkManager on `eno1`:

```text
10.10.10.10
10.10.11.11
search domain: usask.ca
```

## NetworkManager

NetworkManager manages the WAN interface only.

```text
DEVICE           TYPE      STATE                   CONNECTION
eno1             ethernet  connected               Wired connection 1
lo               loopback  connected (externally)  lo
wlp2s0           wifi      disconnected            --
enxc8a362d428b3  ethernet  unmanaged               --
```

Active NetworkManager connections:

```text
Wired connection 1 -> eno1
lo                 -> lo
```

## systemd-networkd LAN config

`systemd-networkd` is enabled and configures the USB Ethernet LAN interface.

Current file:

```ini
# /etc/systemd/network/30-lan.network
[Match]
# Cannot use * or it will also match the internal ethernet device
# Instead, hard code device name of USB ethernet
# Name=enx*
Name=enxc8a362d428b3

[Network]
Address=192.168.0.1/24
DHCP=no
```

## IP forwarding

Forwarding is enabled:

```text
net.ipv4.ip_forward = 1
```

Persistent config:

```ini
# /etc/sysctl.d/99-ipforward.conf
# Setup IP forwarding for ethernet -> router
net.ipv4.ip_forward=1
```

## Firewall/NAT

The system uses `iptables-nft`; `nft list ruleset` displays iptables-managed tables.

### Filter rules

```bash
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A FORWARD -i eno1 -o enxc8a362d428b3 -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i enxc8a362d428b3 -o eno1 -j ACCEPT
```

Meaning:

- LAN to WAN traffic is allowed.
- WAN to LAN return traffic is allowed only when related/established.
- Base policies are permissive (`ACCEPT`).

### NAT rules

```bash
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-A POSTROUTING -o eno1 -j MASQUERADE
```

Meaning:

- All outbound traffic leaving `eno1` is masqueraded.
- Clubroom clients appear to the university network as the Debian mini-PC.

Persisted file:

```text
/etc/iptables/rules.v4
```

## DHCP/DNS

DHCP and DNS are provided by `dnsmasq` on the Debian host.

Service status:

```text
dnsmasq.service enabled and active/running
```

Listening ports:

```text
udp 127.0.0.1:53          dnsmasq
udp 192.168.0.1:53        dnsmasq
udp 0.0.0.0:67            dnsmasq
udp 0.0.0.0:5353          avahi-daemon
udp [::1]:53              dnsmasq
udp [fe80::...%if3]:53    dnsmasq
udp [::]:5353             avahi-daemon

tcp 127.0.0.1:53          dnsmasq
tcp 192.168.0.1:53        dnsmasq
tcp [fe80::...%if3]:53    dnsmasq
tcp [::1]:53              dnsmasq
```

Important active dnsmasq config lines:

```ini
interface=enx*
except-interface=eno1
bind-interfaces

dhcp-range=192.168.0.50,192.168.0.150,12h

dhcp-option=3,192.168.0.1
dhcp-option=6,8.8.8.8,1.1.1.1

# Static IP for Leif's Framework
dhcp-host=4c:82:a9:4c:ca:bd,192.168.0.111,Framework,infinite

# Static IP for the Lounge Chromecast
dhcp-host=d6:25:c2:0c:7a:f9,192.168.0.128,infinite
```

Client DHCP behavior:

| DHCP setting | Value |
|---|---|
| Address range | `192.168.0.50`–`192.168.0.150` |
| Gateway/router | `192.168.0.1` |
| DNS handed to clients | `8.8.8.8`, `1.1.1.1` |
| Lease time | `12h` |

Note: dnsmasq itself listens on `192.168.0.1:53`, but DHCP currently tells clients to use external DNS directly.

Current leases observed:

```text
0 d6:25:c2:0c:7a:f9 192.168.0.128 * 01:d6:25:c2:0c:7a:f9
0 4c:82:a9:4c:ca:bd 192.168.0.111 Framework 01:4c:82:a9:4c:ca:bd
1788513180 3c:2a:f4:73:58:be 192.168.0.91 BRN3C2AF47358BE 01:3c:2a:f4:73:58:be
1788516385 40:d2:8a:0c:f9:ca 192.168.0.82 * 01:40:d2:8a:0c:f9:ca
```

`BRN3C2AF47358BE` is likely the Brother printer hostname.

## QoS / priority traffic

The current system contains QoS configuration for two priority devices.

iptables mangle rules:

```bash
-A PREROUTING -s 192.168.0.111/32 -j MARK --set-xmark 0xa/0xffffffff
-A PREROUTING -s 192.168.0.128/32 -j MARK --set-xmark 0xa/0xffffffff
```

`ap-gateway.service` applies traffic control rules:

```bash
/usr/sbin/tc qdisc add dev eno1 root handle 1: prio
/usr/sbin/tc filter add dev eno1 parent 1: protocol ip handle 10 fw flowid 1:1
```

Interpretation:

- Packets from `192.168.0.111` and `192.168.0.128` are marked with firewall mark `10` (`0xa`).
- Marked packets are directed into a priority queue on `eno1`.
- This appears to have been added for the lounge Chromecast/TV and Leif's Framework laptop.

## Boot persistence

Enabled boot services:

```text
NetworkManager: enabled
systemd-networkd: enabled
dnsmasq: enabled
netfilter-persistent: enabled
ap-gateway: enabled
```

Custom boot service:

```ini
# /etc/systemd/system/ap-gateway.service
[Unit]
Description=WiFi AP Gateway
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iptables-restore /etc/iptables/rules.v4
RemainAfterExit=yes
ExecStartPost=/usr/sbin/tc qdisc add dev eno1 root handle 1: prio
ExecStartPost=/usr/sbin/tc filter add dev eno1 parent 1: protocol ip handle 10 fw flowid 1:1
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

The service is enabled and active/exited successfully.

## Not used / absent

| Component | Status |
|---|---|
| `hostapd` | not installed |
| `isc-dhcp-server` | not installed |
| `kea-dhcp4-server` | not installed |
| TLP | not installed |
| Wi-Fi AP on mini-PC | not used |

Missing diagnostic tools on the current host:

```text
ethtool
iw
rfkill
```

`tc` is not in the normal user's PATH but exists at `/usr/sbin/tc`, because the systemd service ran it successfully.

## Current architecture summary

```text
University network
    ↓ DHCP/authenticated uplink
Debian mini-PC eno1: 10.19.1.233/17
    ↓ Linux routing + NAT masquerade
Debian mini-PC enxc8a362d428b3: 192.168.0.1/24
    ↓ DHCP/DNS via dnsmasq
TP-Link TL-WR740N as AP/switch
    ↓
IEEE clubroom clients and printer on 192.168.0.0/24
```

## AP requirement summary

The AP/router behind the gateway is not the router for this design. It must be configured as a dumb AP/bridge/switch:

- AP DHCP server disabled.
- AP LAN/management IP set statically on the clubroom LAN, normally `192.168.0.2`.
- AP gateway set to `192.168.0.1`.
- Cable from the mini-PC LAN/USB Ethernet plugged into an AP LAN port, not the AP WAN port unless the device's AP mode explicitly bridges the WAN port.
- Wi-Fi clients should receive `192.168.0.x` DHCP leases from the mini-PC gateway, not from the AP.

The old/current TP-Link can be reused if it is already in this dumb-AP shape. If replaced with another consumer router, configure equivalent AP/bridge mode before connecting clients.
