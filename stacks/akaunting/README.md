# Akaunting Stack

Akaunting is the trial finance/accounting system for the IEEE clubroom server.

## Purpose

Use this stack to evaluate whether Akaunting can handle IEEE treasurer workflows:

- Club income and expenses
- Purchases and vendor records
- Payments
- Receipt/document attachment
- Monthly budgeting reports
- Profit/loss reporting
- Treasurer handoff records

If Akaunting does not fit the club's needs, the plan is to build a custom finance module into the IEEE POS/app stack.

## Containers

| Container | Purpose |
|---|---|
| `akaunting` | Akaunting web application |
| `akaunting-db` | MariaDB database owned by Akaunting |

## URL

Planned local URL:

```text
http://finance.ieee.local
```

Until Caddy and Pi-hole DNS are configured, use the server IP and exposed port:

```text
http://<server-ip>:8081
```

The host port can be changed with `AKAUNTING_HTTP_PORT`.

## First deployment

Akaunting has a special first-run setup mode.

1. Create stack environment values in Dockhand using `.env.example` as the template.
2. Set:

   ```env
   AKAUNTING_SETUP=true
   ```

3. Deploy the stack.
4. Open Akaunting in the browser.
5. Complete the setup wizard.
6. Change the stack environment to:

   ```env
   AKAUNTING_SETUP=false
   ```

   or remove the variable.

7. Redeploy the stack normally.

Do **not** leave `AKAUNTING_SETUP=true` after setup is complete. The upstream Docker repo explicitly warns not to reuse setup mode after first setup.

## Required environment values

| Variable | Required | Notes |
|---|---:|---|
| `AKAUNTING_HTTP_PORT` | No | Defaults to `8081` |
| `AKAUNTING_SETUP` | First run only | Use `true` only for initial setup |
| `APP_URL` | No | Defaults to `http://finance.ieee.local` |
| `LOCALE` | No | Defaults to `en-US` |
| `MYSQL_DATABASE` | No | Defaults to `akaunting` |
| `MYSQL_USER` | No | Defaults to `akaunting` |
| `MYSQL_PASSWORD` | Yes | Must be long/random |
| `DB_PREFIX` | Yes | Three random letters/numbers followed by `_`, e.g. `a7f_` |
| `COMPANY_NAME` | No | Defaults to `USask IEEE Student Branch` |
| `COMPANY_EMAIL` | Yes | Used during setup |
| `ADMIN_EMAIL` | Yes | Used during setup |
| `ADMIN_PASSWORD` | Yes | Used during setup; must be changed from the example |

## Data persistence

Docker named volumes:

| Volume | Purpose |
|---|---|
| `akaunting-data` | Akaunting application/runtime files |
| `akaunting-db` | MariaDB database files |

Do not delete these volumes unless intentionally resetting Akaunting.

## Caddy integration

Once the Caddy stack exists, proxy:

```text
finance.ieee.local -> http://<server-ip>:8081
```

A later Caddy stack may instead use a shared Docker network if we decide to couple proxy networking across stacks.

## Notes

- Redis is supported by Akaunting but intentionally omitted for the initial IEEE deployment.
- The database is grouped in this stack because it is owned by Akaunting.
- Live `.env` files and passwords must not be committed to Git.
