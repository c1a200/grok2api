# Deploy grok2api on Render with Neon PostgreSQL

This deployment builds this repository's Dockerfile, uses Neon for PostgreSQL state, and keeps a Render disk only for local media files.

## Deploy

1. Push this repository to GitHub.
2. In Render, select **New** → **Blueprint** and choose the repository. Render reads [`render.yaml`](../../render.yaml).
3. In Neon, copy the pooled connection string for your database. Keep its SSL parameters intact.
4. In the Render web service, open **Environment** and set this value as a **Secret**:

   ```text
   GROK2API_POSTGRES_DSN=<your Neon pooled connection string>
   ```

5. Generate application secrets locally:

   ```bash
   openssl rand -hex 32
   openssl rand -base64 32
   ```

6. Copy [`config.example.yaml`](./config.example.yaml), replace its JWT key, credential-encryption key, and bootstrap password, then add it to Render as a Secret File:

   ```text
   /run/grok2api/config.yaml
   ```

   Leave `database.postgres.dsn` empty. The service fills it from `GROK2API_POSTGRES_DSN` at startup.

7. Deploy the service. Check:

   ```text
   https://<service>.onrender.com/healthz
   https://<service>.onrender.com/login
   ```

## Runtime layout

| Component | Stores |
| --- | --- |
| Neon PostgreSQL | accounts, API keys, settings, logs |
| Render disk at `/app/data` | local image and video media |
| Render Secret File | application secrets and non-database configuration |
| Render secret environment variable | Neon connection string |

## Notes

- Set `GROK2API_POSTGRES_DSN` as a Render secret, not in Git, `render.yaml`, or the secret file.
- The example configuration already uses `auth.secureCookies: true` for Render HTTPS.
- Keep `credentialEncryptionKey` unchanged after the first successful startup, otherwise existing stored credentials cannot be decrypted.
- The Blueprint creates no Render database because Neon is external.
- For more than one web instance, configure Redis and switch `runtimeStore.driver` from `memory` to `redis`.
