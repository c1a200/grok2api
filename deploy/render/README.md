# Deploy grok2api on Render (external PostgreSQL)

This guide deploys the official Docker image with **PostgreSQL for state** and a **small disk only for media files**.

## What you get

| Component | Purpose |
|-----------|---------|
| Web Service | `ghcr.io/chenyme/grok2api:latest` on port `8000` |
| PostgreSQL | accounts, keys, settings, logs (external DB) |
| Persistent Disk (`/app/data`) | local media cache only |

Do **not** use SQLite on Render without a disk for the DB file. This setup keeps the database external so redeploys do not wipe accounts/keys.

## Prerequisites

- Render account (paid Web Service + Postgres plans; Free tier is not suitable)
- Secrets generated locally:

```bash
openssl rand -hex 32
openssl rand -base64 32
```

## Option A — Blueprint (`render.yaml`)

1. Push this repository (or your fork) to GitHub.
2. Render Dashboard → **New** → **Blueprint**.
3. Select the repo; Render reads root `render.yaml`.
4. Apply. Wait until `grok2api` (web) and `grok2api-db` (Postgres) are created.
5. Open the Postgres service → copy **Internal Database URL**.
6. Copy [`config.example.yaml`](./config.example.yaml), fill in:
   - `secrets.jwtSecret`
   - `secrets.credentialEncryptionKey`
   - `bootstrapAdmin.password`
   - `database.postgres.dsn` ← paste the Internal Database URL  
     (add `?sslmode=require` if missing)
7. Web service → **Environment** → **Secret Files**:
   - Filename / path: `/run/grok2api/config.yaml`
   - Contents: your filled config
8. **Manual Deploy** → clear build cache optional → deploy.
9. Open `https://<service>.onrender.com/healthz` and `/login`.

## Option B — Manual UI (any external Postgres)

Works with Render Postgres, Neon, Supabase, RDS, etc.

1. **New Web Service** → **Deploy an existing image from a registry**  
   Image: `ghcr.io/chenyme/grok2api:latest`
2. Plan: at least **Starter**; region close to you (e.g. Singapore).
3. Health check path: `/healthz`
4. Add disk: mount `/app/data`, size ≥ 5 GB (media only).
5. Create or reuse a Postgres instance; copy a connection string with SSL.
6. Add Secret File `/run/grok2api/config.yaml` from `config.example.yaml` with the real DSN.
7. Deploy.

## Config checklist

```yaml
auth.secureCookies: true          # required on HTTPS
database.driver: postgres
database.postgres.dsn: "..."      # external DB, sslmode=require
media.local.path: "./data/media"  # on the /app/data disk
```

Keep `credentialEncryptionKey` stable. Changing it makes existing stored credentials undecryptable.

After the first successful admin login, remove or rotate `bootstrapAdmin.password` in the secret file if you want.

## Verify

```bash
curl -sS https://<service>.onrender.com/healthz
```

Then:

1. Login at `/login`
2. Change admin password
3. Add Grok tokens / API keys in the admin UI
4. Call `/v1/models` with your API key

## Updates

- Image tag `latest` does not always pull automatically. In Render, trigger **Manual Deploy** and enable “clear cache / pull latest image” when available, or pin a version tag such as `ghcr.io/chenyme/grok2api:v3.0.7`.
- Schema migrations are applied by the app on startup against Postgres; no separate migrate job is required for normal upgrades.
- Disk (media) is preserved across deploys; Postgres data is preserved on the database service.

## Limitations

- Streaming / long-running requests may hit Render proxy timeouts; test your workload.
- Outbound IP is a cloud datacenter range; Grok may require a proxy configured in the admin UI.
- Single instance uses `runtimeStore.driver: memory`. For multiple web instances, add Redis and switch the driver.
- Media is still local-disk only; multi-instance needs shared storage or sticky sessions for media URLs.

## Local dry-run of the same config shape

```bash
cp deploy/render/config.example.yaml config.yaml
# edit secrets + postgres DSN
docker run --rm -p 8000:8000 \
  -v "$PWD/config.yaml:/run/grok2api/config.yaml:ro" \
  -v grok2api-data:/app/data \
  ghcr.io/chenyme/grok2api:latest
```
