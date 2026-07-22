# n8n — local Docker setup

Self-hosted [n8n](https://n8n.io) running under Docker Compose with a PostgreSQL backend.

- **Editor:** http://localhost:5678
- **Version running:** n8n 2.30.7 (`docker.n8n.io/n8nio/n8n:latest`)
- **Database:** PostgreSQL 16 (`postgres:16-alpine`)

## Quick start

```powershell
cd c:\laragon\www\htdocs\aDocker\n8n
docker compose up -d
```

Open http://localhost:5678 and create the owner account on first load. That account lives in the
database, not in a config file — there is no default login.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Service definitions for n8n and Postgres |
| `.env` | Port, timezone, encryption key, database credentials. **Not committed.** |
| `.gitignore` | Excludes `.env` and `local-files/` |
| `local-files/` | Mounted into n8n at `/files` — use this path in Read/Write Binary File nodes |

## Configuration

Everything tunable lives in `.env`:

| Variable | Default | Notes |
|---|---|---|
| `N8N_PORT` | `5678` | Host port. Change if something else already uses 5678. |
| `N8N_HOST` | `localhost` | Hostname n8n believes it is served from. |
| `N8N_WEBHOOK_URL` | `http://localhost:5678/` | Base URL for test + production webhooks. |
| `GENERIC_TIMEZONE` | `Europe/London` | **Decides when Schedule/Cron triggers fire.** Set to your own zone. |
| `N8N_ENCRYPTION_KEY` | generated | Decrypts stored credentials. See below. |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | `n8n` / generated / `n8n` | Only reachable inside the Compose network. |

Apply changes with `docker compose up -d` — Compose recreates only what changed.

### The encryption key matters

`N8N_ENCRYPTION_KEY` encrypts every credential you save in n8n. Two rules:

1. **Back up `.env`** somewhere outside this folder. Lose the key and every stored credential
   becomes permanently unreadable — you re-enter all of them by hand.
2. **Never change it** on an existing install. Existing credentials were encrypted with the old
   key and will fail to decrypt.

## Design notes

**Postgres, not SQLite.** SQLite is n8n's default and is fine for a single trigger firing
occasionally, but it takes a write lock per execution — concurrent workflows serialize behind
each other and can time out. Postgres removes that ceiling for the cost of one extra container.

**Startup ordering.** n8n `depends_on` Postgres with `condition: service_healthy`, gated by a
`pg_isready` healthcheck. Without it, n8n races the database on boot and crash-loops through its
migrations.

**Execution pruning is on.** `EXECUTIONS_DATA_PRUNE=true` with a 336-hour (14 day) retention.
Full input/output data is saved for both successes and failures, which is what makes debugging
possible but is also what grows the database — the pruning window is the trade-off. Widen
`EXECUTIONS_DATA_MAX_AGE` if you need longer history.

**Hardening applied:** telemetry off (`N8N_DIAGNOSTICS_ENABLED=false`), Code nodes blocked from
reading host environment variables (`N8N_BLOCK_ENV_ACCESS_IN_NODE`), settings file permissions
enforced. `N8N_SECURE_COOKIE=false` is deliberate — it allows plain HTTP on localhost, and must
be flipped to `true` if this is ever exposed over HTTPS.

**Community nodes** are installable from Settings → Community nodes, including unverified
packages and use as AI tools.

## Common commands

```powershell
docker compose logs -f n8n           # tail logs
docker compose restart n8n           # restart after config changes
docker compose down                  # stop everything; volumes and data survive
docker compose ps                    # container status
```

### Upgrading

```powershell
docker compose pull
docker compose up -d
```

n8n runs its database migrations automatically on start. Take a backup first (below) — n8n
migrations are one-way, and rolling back to an older image after a migration will not work.

### Backup

Workflows, credentials, and execution history all live in Postgres:

```powershell
docker compose exec postgres pg_dump -U n8n n8n > backup.sql
```

Restore into a fresh stack:

```powershell
Get-Content backup.sql | docker compose exec -T postgres psql -U n8n -d n8n
```

A complete backup is `backup.sql` **plus** `.env` — the dump holds credentials still encrypted
with the key in that file.

## Data locations

Two named Docker volumes, both outside this folder and untouched by `docker compose down`:

- `n8n_n8n_data` → `/home/node/.n8n` (instance config, encryption key file, SSH keys)
- `n8n_postgres_data` → Postgres data directory

`docker compose down -v` deletes both. That is the one destructive command here.

## Troubleshooting

**Port 5678 already in use** — usually a leftover `docker run` container: `docker ps` then
`docker stop <name>`. Or change `N8N_PORT` in `.env`.

**"Python 3 is missing from this system" in the logs** — expected. The image ships without
Python, so only Python Code nodes are affected; the JavaScript runner works normally.

**Webhooks unreachable from outside** — `N8N_WEBHOOK_URL` points at `localhost`, which external
services cannot resolve. Put a tunnel or reverse proxy in front and set that variable to the
public URL.
