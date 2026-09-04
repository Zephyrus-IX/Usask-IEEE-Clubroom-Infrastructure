# Canteen/POS Deployment Stack

The canteen/POS application source code lives in a separate repository:

```text
https://github.com/Zephyrus-IX/Usask-IEEE-Canteen
```

This infrastructure repo should contain only the deployment definition, environment example, backup notes, and operational runbook for running the app on the clubroom server.

## Recommended deployment model

Prefer deploying a built image from GitHub Container Registry once the app is ready:

```text
ghcr.io/zephyrus-ix/usask-ieee-canteen:<tag>
```

During early development, a local build from a checked-out source repo can be used, but that couples the server to a specific filesystem path and is less reproducible.

## Why not a Git submodule by default?

A submodule pins the app source at an exact commit, but it makes Dockhand/Git workflows and handoff to future execs more fragile. Keep the app repo separate unless exact source pinning becomes necessary.

## Files to add when ready

- `compose.yaml`
- `.env` Dockhand template with placeholders/defaults only
- backup/restore notes for database and uploaded files
- first-deploy instructions for migrations/admin account creation
