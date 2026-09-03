# Network Addresses

## Clubroom LAN

| Item | Address/range | Notes |
|---|---|---|
| Clubroom LAN | `192.168.0.0/24` | Private network behind gateway PC |
| Gateway PC LAN | `192.168.0.1` | Default gateway for clients |
| Primary AP | `192.168.0.2` | Static management IP, outside DHCP range |
| DHCP pool | `192.168.0.50`-`192.168.0.150` | Served by gateway dnsmasq initially |

## Reserved/static addresses

| Device | IP | MAC | Notes |
|---|---:|---|---|
| Primary AP | `192.168.0.2` | TBD | TP-Link management IP |
| Printer | TBD | TBD | Keep outside DHCP pool or reserve in DHCP |

## Service hostnames

| Hostname | Target | Purpose |
|---|---|---|
| `dockhand.ieee.local` | gateway/server LAN IP | Docker stack management |
| `finance.ieee.local` | gateway/server LAN IP | Akaunting |
| `pos.ieee.local` | gateway/server LAN IP | Canteen/POS app |
| `pihole.ieee.local` | gateway/server LAN IP | Pi-hole admin UI |
| `network.ieee.local` | gateway/server LAN IP | Network visibility tools |
| `home.ieee.local` | gateway/server LAN IP | Optional dashboard |
