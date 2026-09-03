# USask IEEE Clubroom Infrastructure

Git-backed source of truth for the University of Saskatchewan IEEE clubroom infrastructure: the Linux gateway/server PC, private clubroom network, access point setup, host rebuild scripts, Docker/Dockhand stacks, backup notes, and operating docs.

> This repo stores deployment definitions and documentation. It should not store live secrets, database data, app uploads, logs, or runtime state.

---

## Scope

This repository covers:

- Linux mini-PC gateway/server setup
- Clubroom private LAN, DHCP/DNS, AP configuration, and routing docs
- Host setup scripts and reproducible rebuild notes
- Docker Compose/Dockhand stack definitions
- Inventory of hardware, addresses, and services
- Backup/restore guidance
- Deployment notes for external app repos such as the canteen/POS app

The custom canteen/POS application source code lives separately at:

```text
https://github.com/Zephyrus-IX/Usask-IEEE-Canteen
```

This repo should contain the POS deployment stack and operational notes, not the full app source code, unless we later deliberately choose submodules or vendoring.

---

## Expected deployment model

```text
1. Install base Linux OS on the IEEE clubroom PC
2. Configure the gateway/network layer if this PC is routing the clubroom LAN
3. Install Docker / Docker Compose
4. Manually install Dockhand
5. Connect Dockhand to this GitHub repo
6. Dockhand syncs the repo from GitHub
7. Each stack under docker-stacks/<name>/ is deployed from Dockhand
8. Future infrastructure changes are made in Git, pushed to GitHub, then synced/deployed through Dockhand
```

Dockhand itself is the bootstrap service. Everything else should be managed through this repository once Dockhand is online.

---

## Repository layout

```text
.
├── README.md
├── docs/
│   ├── backup-and-restore.md
│   └── network/
│       ├── current-audit.md
│       └── rebuild-plan.md
├── host-config/
│   └── gateway/
│       ├── README.md
│       └── configure-clubroom-gateway.sh
├── docker-stacks/
│   ├── akaunting/
│   │   ├── compose.yaml
│   │   ├── .env.example
│   │   └── README.md
│   ├── caddy/
│   │   ├── compose.yaml
│   │   ├── Caddyfile
│   │   ├── .env.example
│   │   └── README.md
│   └── canteen/
│       └── README.md
├── inventory/
│   ├── hardware.md
│   ├── network-addresses.md
│   └── services.md
└── .gitignore
```

Not every planned stack needs to exist immediately. Add each stack when it is ready to deploy.

---

## Key docs

| File | Purpose |
|---|---|
| `docs/network/current-audit.md` | Reverse-engineered current Debian gateway setup |
| `docs/network/rebuild-plan.md` | Debian/Fedora rebuild plan for the clubroom network |
| `host-config/gateway/README.md` | How to run the interactive gateway setup helper |
| `host-config/gateway/configure-clubroom-gateway.sh` | Interactive host-network setup script |
| `docs/backup-and-restore.md` | What belongs in Git vs real backups and a restore outline |
| `inventory/hardware.md` | Physical device inventory |
| `inventory/network-addresses.md` | IP ranges, static addresses, and local service hostnames |
| `inventory/services.md` | Intended/deployed service list |

---

## Planned Docker stacks

### `caddy`

Reverse proxy for clean local service URLs.

Planned routes:

| Hostname | Target |
|---|---|
| `pos.ieee.local` | IEEE POS/canteen app |
| `finance.ieee.local` | Akaunting |
| `pihole.ieee.local` | Pi-hole admin UI |
| `network.ieee.local` | NetAlertX |
| `dockhand.ieee.local` | Dockhand |
| `home.ieee.local` | Optional dashboard |

Caddy is preferred over Nginx Proxy Manager for this deployment because the proxy configuration can be stored directly in Git and managed through Dockhand.

### `pihole`

Local DNS and network-level filtering for the clubroom network.

Initial goal:

- DNS filtering
- Local DNS records for `*.ieee.local` services
- Optional DHCP later, only after the base gateway is proven working

Do not run host `dnsmasq` DHCP/DNS and Pi-hole DHCP/DNS on the same ports/interfaces at the same time.

### `netalertx`

Network visibility and device discovery.

Initial goal:

- Discover devices on the clubroom subnet
- Track IP/MAC/hostname changes
- Help identify unknown devices
- Optionally integrate with Pi-hole once DNS is working

NetAlertX may require host networking and extra Linux capabilities for ARP/network scanning.

### `akaunting`

Club finance/accounting system trial.

Initial goal:

- Track club income and expenses
- Track purchases, vendors, payments, and receipts
- Produce monthly budgeting and financial reports
- Evaluate whether it meets IEEE treasurer workflow needs

If Akaunting does not fit the club's needs, the plan is to build a custom finance module into the IEEE app.

### `canteen`

Deployment wrapper for the separate custom IEEE canteen/POS application.

Source repo:

```text
https://github.com/Zephyrus-IX/Usask-IEEE-Canteen
```

Initial goal:

- Exec-facilitated snack sales
- Student/member tabs or accounts
- Prepaid balances
- Member/non-member pricing
- Inventory and restock tracking
- Cash/card payment records
- Revenue and profit/loss exports

Preferred long-term deployment is a built container image, for example:

```text
ghcr.io/zephyrus-ix/usask-ieee-canteen:<tag>
```

### `homepage` optional

Simple internal dashboard linking to deployed services. Add only after core services are working.

---

## Bootstrap process

### 1. Prepare the server/gateway PC

Install the base system requirements manually:

- Docker Engine
- Docker Compose plugin
- Git
- SSH access or local admin access
- Static/reserved LAN IP for the server/gateway PC

If the PC is also acting as the clubroom router, follow `docs/network/rebuild-plan.md` and `host-config/gateway/README.md`.

### 2. Install Dockhand manually

Dockhand is installed outside this repo first because it is needed to sync and deploy this repo.

After Dockhand is reachable:

1. Open Dockhand in the browser.
2. Connect this GitHub repository.
3. Configure the repository/branch used for deployment.
4. Sync the repository.
5. Deploy stacks from `docker-stacks/<stack-name>/compose.yaml`.

### 3. Deploy base infrastructure stacks

Recommended order:

```text
1. caddy
2. pihole
3. netalertx
4. akaunting
5. canteen
6. homepage, optional
```

Reasoning:

- Caddy gives clean URLs for later services.
- Pi-hole provides local DNS records for those URLs.
- NetAlertX is more useful once Pi-hole/local naming exists.
- Akaunting and the IEEE POS app are user-facing apps and can be deployed after the base network layer.

### 4. Configure local DNS

In Pi-hole or the active DHCP/DNS system, create local DNS records for the planned service hostnames and point them to the IEEE server/gateway PC's LAN IP.

Example:

```text
pos.ieee.local       -> <server-ip>
finance.ieee.local   -> <server-ip>
pihole.ieee.local    -> <server-ip>
network.ieee.local   -> <server-ip>
dockhand.ieee.local  -> <server-ip>
home.ieee.local      -> <server-ip>
```

Then configure the active DHCP service to hand out the intended DNS server to clients.

---

## Secrets and environment files

Commit:

- `compose.yaml`
- `.env.example`
- stack `README.md` files
- static config files that do not contain secrets
- documentation

Do **not** commit:

- live `.env` files
- passwords
- API keys
- database files
- uploaded receipts/documents
- app runtime data
- logs
- generated caches

Each stack should include a `.env.example` file documenting required environment variables. Live `.env` files should be created on the server/Dockhand side.

---

## Agent/Hermes working convention

When Hermes or another AI agent works on this repository, the repo must be cloned and edited only under the Unraid agent workspace share:

```text
/unraid/agent-workspace/projects/Usask-IEEE-Clubroom-Infrastructure
```

Do not clone this repository under `/opt/data`, `/tmp`, a home directory, or any other local-only path. The agent-workspace share is the required working location so generated files and edits remain visible in the intended server workspace.

---

## Operational guidelines

- Treat Git as the source of truth for infrastructure definitions.
- Prefer one app per Docker stack unless services are tightly dependent.
- Group databases with the app that owns them.
- Keep Dockhand as the manually bootstrapped control service.
- Use Caddy for Git-tracked reverse proxy configuration.
- Keep optional tools optional until the core system is stable.
- Do not expose admin tools publicly without a deliberate security review.
- Do not enable automatic image updates until the recovery/update process is documented.

---

## Current target stack summary

| Stack | Containers | Required | Purpose |
|---|---|---:|---|
| `caddy` | `caddy` | Yes | Reverse proxy and local service URLs |
| `pihole` | `pihole` | Yes | DNS filtering and local DNS records |
| `netalertx` | `netalertx` | Maybe | Network device discovery/visibility |
| `akaunting` | `akaunting`, `akaunting-db` | Yes | Club finance/accounting trial |
| `canteen` | canteen app, PostgreSQL | Yes | Custom canteen/POS app deployment |
| `homepage` | `homepage` | Optional | Internal dashboard |
