# Deploying Evolution Go on Railway

## Why not Vercel / Netlify / serverless

Evolution Go cannot run on a serverless platform. This is architectural, not a
configuration problem:

- **It is a long-running server.** `main()` calls `srv.ListenAndServe()` and
  blocks until SIGINT/SIGTERM. Serverless platforms invoke a handler per
  request; nothing binds a port.
- **It needs a writable, persistent disk.** The auth database is SQLite at
  `/app/dbdata/users.db`. Serverless filesystems are read-only apart from
  `/tmp`, which is wiped between invocations — the WhatsApp session would be
  destroyed continuously, forcing a QR re-scan on every request.
- **WhatsApp needs a persistent websocket.** `whatsmeow` holds a live socket to
  WhatsApp's servers. Functions are frozen or killed right after responding, so
  the connection cannot survive. The same applies to the heartbeat goroutine and
  the RabbitMQ/NATS producers.
- **It builds with `CGO_ENABLED=1`** and needs `ffmpeg`, `poppler-utils` and
  `libwebp` at runtime (see `Dockerfile`).

Railway runs the container as a real process with a persistent volume, so all
four hold.

## 1. Create the service

1. Railway → **New Project** → **Deploy from GitHub repo** → select this repo.
2. Railway reads `railway.json` and builds from the `Dockerfile`. Do **not** set
   a custom start command — the Dockerfile's `ENTRYPOINT ["/app/server"]`
   handles startup, and a start command would be appended to it as an argument.

`railway.json` pins `numReplicas: 1` deliberately. WhatsApp sessions are
stateful and tied to one SQLite file; a second replica would fight the first over
the same session and disconnect it.

## 2. State: Postgres, not a volume

Where the WhatsApp session lives depends entirely on whether
`POSTGRES_AUTH_DB` is set:

- **`POSTGRES_AUTH_DB` set (recommended on Railway).** `initAuthDB` returns
  early (`cmd/evolution-go/main.go:273`) and `whatsmeow` opens a Postgres
  session store. Nothing is written to `dbdata/`, so **no volume is needed** —
  the container is disk-stateless and survives redeploys cleanly.
- **`POSTGRES_AUTH_DB` empty.** Session credentials fall back to SQLite at
  `/app/dbdata/`. You then *must* mount a volume at `/app/dbdata`
  (Service → **Settings** → **Volumes**) or every redeploy wipes the session and
  forces a QR re-scan.

Step 4 sets `POSTGRES_AUTH_DB`, so the volume is optional. Mounting one anyway
is harmless.

Set `LOG_DIRECTORY` to `/tmp/logs` either way. Leaving it unset is not fatal but
logs a `Falha ao criar diretório base de logs` error on every boot, because the
logger calls `os.MkdirAll("")` (`pkg/logger/logger.go:39`). Railway captures
stdout regardless.

## 3. Add Postgres

Add a **Postgres** service to the project. That is all — you do **not** need to
create the databases by hand. `ensureDBExists` (`pkg/config/config.go:79`)
connects to the `postgres` maintenance database, checks `pg_database`, and runs
`CREATE DATABASE` itself; tables then come from `db.AutoMigrate`. Railway's
default user is a superuser, so it has the required privilege.

Two variables matter, and they are not interchangeable:

- **`POSTGRES_USERS_DB` — required.** Holds the instance/message/label tables,
  and is also what `CreateAuthDB()` connects to. If it is empty, the app exits
  with `[CONFIG] required database configuration variables are missing`.
- **`POSTGRES_AUTH_DB` — optional, but set it on Railway.** It is the
  `whatsmeow` session store (`pkg/whatsmeow/service/whatsmeow.go:322`). When
  set, `initAuthDB` returns early and **no SQLite file is used at all** — the
  WhatsApp session lives in Postgres instead of `dbdata/main.db`.

## 4. Set environment variables

On the **evolution-go** service → **Variables** — not on the Postgres service.

> **The `${{Postgres.*}}` references below only resolve if your Postgres service
> is named exactly `Postgres`.** Railway often names it `Postgres-xxxx` or
> similar, and an unresolved reference silently becomes an empty string — which
> surfaces as `[CONFIG] required database configuration variables are missing`.
> Check the service's name in the sidebar and substitute it, or use Railway's
> **Add Reference** button, which inserts the correct name for you. See
> *Troubleshooting* below.

| Variable | Value |
|---|---|
| `SERVER_PORT` | `8080` |
| `GLOBAL_API_KEY` | **generate a new random key — see below** |
| `CLIENT_NAME` | `evolution` |
| `POSTGRES_AUTH_DB` | `postgresql://${{Postgres.PGUSER}}:${{Postgres.PGPASSWORD}}@${{Postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/evogo_auth?sslmode=disable` |
| `POSTGRES_USERS_DB` | `postgresql://${{Postgres.PGUSER}}:${{Postgres.PGPASSWORD}}@${{Postgres.RAILWAY_PRIVATE_DOMAIN}}:5432/evogo_users?sslmode=disable` |
| `DATABASE_SAVE_MESSAGES` | `false` |
| `CONNECT_ON_STARTUP` | `true` |
| `OS_NAME` | `Evolution GO` |
| `LOG_TYPE` | `console` |
| `LOG_DIRECTORY` | `/tmp/logs` |
| `DEBUG_ENABLED` | `0` |
| `WEBHOOK_FILES` | `true` |
| `EVENT_IGNORE_STATUS` | `true` |
| `QRCODE_MAX_COUNT` | `5` |
| `TZ` | `Asia/Kolkata` |

Generate the API key — never reuse the one in `.env.example`, which is public:

```bash
openssl rand -hex 16
```

Note the exact env names: the code reads `DEBUG_ENABLED` and `LOG_TYPE`. The
`WADEBUG` and `LOGTYPE` keys in the upstream compose files are stale and are
silently ignored.

## 5. Expose it

Service → **Settings** → **Networking** → **Generate Domain**.

The healthcheck in `railway.json` hits `/server/ok`, which is unauthenticated.
Railway will not mark the deploy live until it returns 200.

## 6. Verify

```bash
# Health — expects 200
curl https://<your-domain>/server/ok

# Authenticated check
curl -H "apikey: <GLOBAL_API_KEY>" https://<your-domain>/instance/all
```

Manager UI: `https://<your-domain>/manager`

## Operational notes

- **Never run more than one replica.** See step 1.
- **Back up the volume.** `/app/dbdata/users.db` holds your session
  credentials; losing it means re-pairing every instance.
- **Do not enable scale-to-zero / idle shutdown.** Suspending the container
  drops the WhatsApp websocket.
- **Media features** (`/send/media` document thumbnails, audio conversion) rely
  on `ffmpeg` and `pdftoppm` from the runtime image — they work on Railway
  because the Dockerfile installs them.

## Troubleshooting

### `[CONFIG] required database configuration variables are missing`

`POSTGRES_USERS_DB` resolved to an empty string. The check
(`pkg/config/config.go:222`) fails only when it is empty *and* the discrete
`POSTGRES_HOST/PORT/USER/PASSWORD/DB` set is incomplete. In order of likelihood:

1. **Postgres service name mismatch.** `${{Postgres.PGUSER}}` requires a service
   named exactly `Postgres`. Unresolved references become empty.
2. **Variables set on the wrong service** — they must be on `evolution-go`.
3. **Postgres service not linked** to the app service.

The reference-free fix, which works regardless of naming: open the Postgres
service → **Variables**, copy the literal values, and set the URLs by hand on
the `evolution-go` service:

```
POSTGRES_USERS_DB=postgresql://postgres:<PGPASSWORD>@<RAILWAY_PRIVATE_DOMAIN>:5432/evogo_users?sslmode=disable
POSTGRES_AUTH_DB=postgresql://postgres:<PGPASSWORD>@<RAILWAY_PRIVATE_DOMAIN>:5432/evogo_auth?sslmode=disable
```

Both databases are created automatically on boot, so the names need not exist
yet. Confirm the values landed by checking the deploy log for
`Connecting to database on: ...` (`pkg/config/config.go:145`) — an empty host
there means the variable is still unresolved.

### If Postgres connects but the dial fails

`RAILWAY_PRIVATE_DOMAIN` resolves over IPv6 on Railway's private network. If the
logs show a dial or DNS error, switch both URLs to the public proxy host
(`${{Postgres.RAILWAY_TCP_PROXY_DOMAIN}}` with
`${{Postgres.RAILWAY_TCP_PROXY_PORT}}` as the port). That path is IPv4 and always
works, at the cost of a little egress.

