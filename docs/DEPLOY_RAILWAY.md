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

## 2. Add the persistent volume — do not skip this

Service → **Settings** → **Volumes** → add a volume with mount path:

```
/app/dbdata
```

This is where `users.db` (your WhatsApp session credentials) lives. Without the
volume, every redeploy wipes the session and you re-scan the QR code.

Railway allows one volume per service, so point `LOG_DIRECTORY` at `/tmp/logs`
(step 4) rather than the volume — per-instance log files would otherwise compete
with the session database for volume space. Leaving it unset is not harmful but
logs a `Falha ao criar diretório base de logs` error on every boot, because the
logger calls `os.MkdirAll("")`. Railway captures stdout regardless.

## 3. Add Postgres and create two databases

Add a **Postgres** service to the project. The app needs *two* databases, which
Railway does not create for you.

Open the Postgres service → **Data** tab → run:

```sql
CREATE DATABASE evogo_auth;
CREATE DATABASE evogo_users;
```

Tables are created automatically on first boot (`db.AutoMigrate`), so only the
databases themselves need to exist.

## 4. Set environment variables

On the **evolution-go** service → **Variables**. The `${{Postgres.*}}` values are
Railway variable references; paste them literally and Railway resolves them.

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

### If Postgres connection fails on boot

`RAILWAY_PRIVATE_DOMAIN` resolves over IPv6 on Railway's private network. If the
logs show a dial or DNS error, switch both URLs to the public proxy host
(`${{Postgres.RAILWAY_TCP_PROXY_DOMAIN}}` with
`${{Postgres.RAILWAY_TCP_PROXY_PORT}}` as the port). That path is IPv4 and always
works, at the cost of a little egress.

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
