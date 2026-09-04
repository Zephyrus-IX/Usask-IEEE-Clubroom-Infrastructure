# Backup and Restore

This repository stores infrastructure definitions and documentation, not runtime data.

## Commit to Git

- Docker Compose files
- committed stack `.env` templates containing only defaults/placeholders
- static non-secret config files
- host setup scripts
- network/AP/DNS documentation
- inventory and operational runbooks

## Do not commit to Git

- local Dockhand `.env` overrides or real secret values
- passwords, API tokens, private keys
- database files
- uploaded receipts/documents
- app runtime data
- logs and caches

## Runtime data that needs real backups

| Data | Likely location | Backup method |
|---|---|---|
| Docker appdata | host/Dockhand appdata path TBD | file-level backup or volume backup |
| PostgreSQL databases | service-specific volumes | scheduled `pg_dump` or volume snapshot |
| Akaunting uploads | Akaunting appdata/volume | file-level backup with database backup |
| Canteen/POS database | PostgreSQL volume | scheduled `pg_dump` |
| Caddy config/certs | Caddy volume | backup if not fully reproducible from Git |

## Restore outline

1. Install base OS and Docker.
2. Configure network gateway using `host-config/gateway/configure-clubroom-gateway.sh` or the documented manual process.
3. Install Dockhand manually.
4. Connect Dockhand to this Git repo.
5. Restore appdata/database backups before starting dependent stacks.
6. Deploy Docker stacks from `docker-stacks/`.
7. Verify DNS, reverse proxy routes, and application logins.
