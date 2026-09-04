# Service Inventory

This file tracks intended and deployed services for the IEEE clubroom infrastructure.

| Service | Location | Status | Purpose |
|---|---|---|---|
| Dockhand | Manual bootstrap on host | Planned | Manage Docker Compose stacks from Git |
| Caddy | `docker-stacks/caddy/` | Present | Reverse proxy/local URLs |
| Akaunting | `docker-stacks/akaunting/` | Present | Finance/accounting trial |
| Pi-hole | `docker-stacks/pihole/` | Present | DNS filtering/local DNS records; DHCP later only if deliberate |
| NetAlertX | `docker-stacks/netalertx/` | Planned | Network device visibility |
| Homepage | `docker-stacks/homepage/` | Optional | Internal dashboard |
| Canteen/POS | external app repo + `docker-stacks/canteen/` | Planned | Snack/canteen sales and tabs |

## External application repos

| App | Repo | Notes |
|---|---|---|
| Canteen/POS | `https://github.com/Zephyrus-IX/Usask-IEEE-Canteen` | Keep source code separate from this infrastructure repo |
