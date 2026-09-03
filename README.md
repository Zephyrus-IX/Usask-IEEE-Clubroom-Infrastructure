# Usask IEEE Docker Deployment

Deployment documentation and Docker Compose stack definitions for the University of Saskatchewan IEEE clubroom server PC.

This repository is intended to be the **Git-backed source of truth** for the clubroom server. Dockhand is installed manually on the server first, then this GitHub repository is connected to Dockhand so stacks can be deployed and updated through Dockhand's Git sync workflow.

> This repo stores deployment definitions and documentation. It should not store live secrets, database data, app uploads, logs, or runtime state.

---

## Expected deployment model

```text
1. Install Docker / Docker Compose on the IEEE server PC
2. Manually install Dockhand on the server PC
3. Connect this GitHub repo to Dockhand
4. Dockhand syncs the repo from GitHub
5. Each stack under stacks/<name>/ is deployed from Dockhand
6. Future changes are made in Git, pushed to GitHub, then synced/deployed through Dockhand
```

Dockhand itself is the bootstrap service. Everything else should be managed through this repository once Dockhand is online.

---

## Repository layout

Planned structure:

```text
.
├── README.md
├── docs/
│   ├── bootstrap.md              # Manual first-time server/Dockhand setup notes
│   ├── network.md                # Clubroom network/DNS notes
│   └── operations.md             # Common admin procedures
├── stacks/
│   ├── caddy/
│   │   ├── compose.yaml
│   │   ├── Caddyfile
│   │   ├── .env.example
│   │   └── README.md
│   ├── pihole/
│   │   ├── compose.yaml
│   │   ├── .env.example
│   │   └── README.md
│   ├── netalertx/
│   │   ├── compose.yaml
│   │   ├── .env.example
│   │   └── README.md
│   ├── akaunting/
│   │   ├── compose.yaml
│   │   ├── .env.example
│   │   └── README.md
│   ├── ieee-pos/
│   │   ├── compose.yaml
│   │   ├── .env.example
│   │   └── README.md
│   └── homepage/
│       ├── compose.yaml
│       ├── .env.example
│       └── README.md
└── .gitignore
```

Not every planned stack needs to exist immediately. Add each stack when it is ready to deploy.

---

## Planned stacks

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
- Optional DHCP later, only if the router/DHCP situation requires it

Preferred first deployment mode is **DNS-only**, where the existing router/DHCP server gives clients the Pi-hole address as DNS.

### `netalertx`

Network visibility and device discovery.

Initial goal:

- Discover devices on the clubroom subnet
- Track IP/MAC/hostname changes
- Help identify unknown devices
- Optionally integrate with Pi-hole once DNS is working

NetAlertX may require host networking and extra Linux capabilities for ARP/network scanning. Review its stack README before deployment.

### `akaunting`

Club finance/accounting system trial.

Initial goal:

- Track club income and expenses
- Track purchases, vendors, payments, and receipts
- Produce monthly budgeting and financial reports
- Evaluate whether it meets IEEE treasurer workflow needs

If Akaunting does not fit the club's needs, the plan is to build a custom finance module into the IEEE app.

### `ieee-pos`

Custom IEEE canteen/POS application.

Initial goal:

- Exec-facilitated snack sales
- Student/member tabs or accounts
- Prepaid balances
- Member/non-member pricing
- Inventory and restock tracking
- Cash/card payment records
- Revenue and profit/loss exports

Expected dependent service:

- PostgreSQL database

### `homepage` optional

Simple internal dashboard linking to deployed services.

This is optional and should only be added after the core services are working.

---

## Bootstrap process

### 1. Prepare the server PC

Install the base system requirements manually:

- Docker Engine
- Docker Compose plugin
- Git
- SSH access or local admin access
- Static/reserved LAN IP for the server PC

Record the server IP and network details in `docs/network.md` once finalized.

### 2. Install Dockhand manually

Dockhand is installed outside this repo first because it is needed to sync and deploy this repo.

After Dockhand is reachable:

1. Open Dockhand in the browser.
2. Connect this GitHub repository.
3. Configure the repository/branch used for deployment.
4. Sync the repository.
5. Deploy stacks from `stacks/<stack-name>/compose.yaml`.

### 3. Deploy base infrastructure stacks

Recommended order:

```text
1. caddy
2. pihole
3. netalertx
4. akaunting
5. ieee-pos
6. homepage, optional
```

Reasoning:

- Caddy gives clean URLs for later services.
- Pi-hole provides local DNS records for those URLs.
- NetAlertX is more useful once Pi-hole/local naming exists.
- Akaunting and the IEEE POS app are user-facing apps and can be deployed after the base network layer.

### 4. Configure local DNS

In Pi-hole, create local DNS records for the planned service hostnames and point them to the IEEE server PC's LAN IP.

Example:

```text
pos.ieee.local       -> <server-ip>
finance.ieee.local   -> <server-ip>
pihole.ieee.local    -> <server-ip>
network.ieee.local   -> <server-ip>
dockhand.ieee.local  -> <server-ip>
home.ieee.local      -> <server-ip>
```

Then configure client devices or the router/DHCP server to use Pi-hole as DNS.

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
/unraid/agent-workspace/projects/Usask-IEEE-Docker-Deployment
```

Do not clone this repository under `/opt/data`, `/tmp`, a home directory, or any other local-only path. The agent-workspace share is the required working location so generated files and edits remain visible in the intended server workspace.

---

## Operational guidelines

- Treat Git as the source of truth for stack definitions.
- Prefer one app per stack unless services are tightly dependent.
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
| `ieee-pos` | `ieee-pos`, `ieee-pos-db` | Yes | Custom canteen/POS app |
| `homepage` | `homepage` | Optional | Internal dashboard |

---

## Status

Initial planning repository. Compose stacks and detailed deployment documentation will be added incrementally.
