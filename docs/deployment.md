---
title: Deployment
description: Deploy WhelkMail on Cloudflare Workers (default), Cloudflare Pages or Docker with SQLite and S3.
---

# Deploying WhelkMail

WhelkMail is one codebase with three deployment targets:

| Target | Best for | Database | Object storage | Mail sockets |
|---|---|---|---|---|
| **Cloudflare Workers** (default) | New installations | D1 | R2 | `cloudflare:sockets` |
| **Cloudflare Pages** | Existing Pages installations | D1 | R2 | `cloudflare:sockets` |
| **Docker** | Any VPS, no Cloudflare dependency | SQLite (`node:sqlite`) | Any S3-compatible bucket | `node:net` / `node:tls` |

The frontend build and the API handlers are identical; only the entrypoint and
the adapters change. Secrets are always `APP_PASSWORD` and `TOKEN_ENC_KEY`, plus
the optional OAuth, relay and VAPID values described in
[Configuration](configuration).

## Fast paths

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail)

- **Deploy to Cloudflare (Workers, default):** use the
  [Deploy to Cloudflare button](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail).
  Cloudflare clones the repository into your GitHub/GitLab account, provisions
  D1 (and R2 if enabled), writes the resource ids into `wrangler.jsonc`,
  configures Workers Builds and deploys. The setup page asks for
  `APP_PASSWORD`, `TOKEN_ENC_KEY` and `CRON_SECRET` from `.dev.vars.example`.
  Custom Worker names are supported; the chosen name is passed through
  `WRANGLER_CI_OVERRIDE_NAME`.
  After the first deploy, clone the new repository and run
  `npm install && npm run db:init:workers:remote` once to create the schema.
- **Installer (Pages, beta):** the
  [WhelkMail Installer](https://github.com/WhelkMail/installer) creates the D1
  database and Pages project, stores the secrets, applies the schema and
  verifies `GET /api/health`. It targets Pages only; use the button above for
  Workers.
- **Docker:** `deploy/docker/compose.yml` starts the app with SQLite in a
  volume. See [Docker](#docker) below.

---

## 1. Cloudflare Workers (default)

### 1.1 Create the D1 database

The Deploy to Cloudflare button provisions D1 from the binding in
`wrangler.jsonc` and writes its id into the new repository. For a CLI deployment:

```bash
npx wrangler d1 create mail-client --binding DB --update-config
```

`wrangler.jsonc` intentionally has no `database_id`: this keeps the public
repository generic and lets both the Deploy flow and `wrangler deploy`
auto-provision the database. If you add an existing id locally, keep it out of
commits.

The D1 database keeps the legacy name `mail-client` so existing data continues
to work. Only the Worker/Pages project is named `whelkmail`. A custom Worker
name is passed through `WRANGLER_CI_OVERRIDE_NAME` by the Deploy flow; for CLI
use `WORKER_NAME=my-worker npm run deploy`.

### 1.2 Apply the schema

```bash
npm run db:init:workers:remote
```

For an existing database, apply only the missing additive migrations:

```bash
npm run db:migrate:workers:remote
```

### 1.3 Add secrets

```bash
npx wrangler secret put APP_PASSWORD
npx wrangler secret put TOKEN_ENC_KEY
npx wrangler secret put CRON_SECRET
```

Generate `TOKEN_ENC_KEY` with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Add the optional OAuth, relay and VAPID secrets the same way. `VAPID_SUBJECT`
can be a plain variable such as `mailto:you@example.com`.

### 1.4 Deploy

```bash
npm run deploy
```

This runs `npm run build` and `wrangler deploy`. Workers Assets serve `dist`;
`assets.run_worker_first` sends `/api/*` to the Worker and
`not_found_handling` keeps SPA deep links working.

The `triggers.crons` block in `wrangler.jsonc` runs the notification, sync,
filter, forwarding and vacation-responder pass directly. **No separate cron
worker is needed on Workers.** The template wakes every minute; on the Free plan
use `["*/5 * * * *"]`.

### 1.5 Local development

```bash
npm run db:init:workers:local
npm run dev            # Vite on http://localhost:5173
npm run dev:workers    # Worker + local D1 on http://localhost:8787
```

Put `APP_PASSWORD` and `TOKEN_ENC_KEY` in `.dev.vars` (gitignored).

### 1.6 Optional R2 archive

Create an R2 bucket and uncomment the `r2_buckets` block in `wrangler.jsonc`
with binding `BUCKET`. Without R2, relay accounts can still archive to Google
Drive or an IMAP mailbox.

### 1.7 Custom domain

Workers & Pages dashboard -> your Worker -> **Settings -> Domains & Routes**.
Add the hostname and update OAuth redirect URIs if you use OAuth.

---

## 2. Cloudflare Pages (existing installations)

Existing Pages projects need **no data migration**: keep the same D1 database,
R2 bucket, secrets and project. Deploy the new release and Pages keeps using
native file routing under `functions/api`; `_middleware.ts` registers the
Cloudflare socket implementation for the mail clients.

Pages does not support custom Wrangler configuration paths. The repository's
tracked `wrangler.jsonc` is the shared root config; remove any legacy local
`wrangler.toml` to avoid ambiguity. For deployed bindings, the Pages dashboard
remains the source of truth.

### 2.1 New Pages project

1. Create the database with `npx wrangler d1 create mail-client --binding DB
   --update-config` for local CLI work, or set the `DB` binding in the dashboard.
2. Pages -> **Create -> Connect to Git**; build command `npm run build`, output
   `dist`, `NODE_VERSION=22` (or newer).
3. Add the `DB` binding (and optional `BUCKET`) in **Settings -> Bindings**.
4. Add `APP_PASSWORD` and `TOKEN_ENC_KEY` in **Settings -> Variables and
   Secrets**.
5. Apply the schema with `npm run db:init:remote` (or `db:migrate:remote` for an
   existing database).
6. Deploy with `npm run deploy:pages`, or push to the connected branch.

### 2.2 Cron worker for Pages

Pages has no native cron trigger. The optional `worker/` deployment calls the
same authenticated endpoint:

```bash
cp worker/wrangler.toml.example worker/wrangler.toml
# set APP_URL to https://<your-project>.pages.dev
npx wrangler secret put CRON_SECRET -c worker/wrangler.toml
npm run cron:deploy
```

The cron worker runs notifications, background sync, filters, forwarding,
vacation replies and provider cleanup. Without it, folders still sync when they
are opened in the UI.

### 2.3 CLI deployment

```bash
npx wrangler pages project create whelkmail
npm run deploy:pages
```

With GitHub deployments, configure the D1 binding and secrets in the dashboard;
the public `wrangler.jsonc` has no `database_id`, so the Deploy flow and
`wrangler deploy` can provision the database.

---

## 3. Docker / any VPS

```bash
cd deploy/docker
cp .env.example .env
# edit APP_PASSWORD, TOKEN_ENC_KEY and PUBLIC_ORIGIN
docker compose up -d --build
```

Open `http://<host>:8787`, or put Caddy/Traefik/nginx in front for TLS and set
`PUBLIC_ORIGIN=https://mail.example.com`. OAuth callbacks and secure cookies are
derived from `PUBLIC_ORIGIN`; the proxy should forward `X-Forwarded-Proto` and
`X-Forwarded-Host` if you prefer to omit it.

The image uses Node 24, the built-in `node:sqlite` module and a bundled server
(`dist-server/server.mjs`): the runtime stage has no `node_modules` and no
native build step.

### 3.1 Data

The SQLite database lives in the `whelkmail-data` volume at
`/data/whelkmail.sqlite`. Back up that volume. Schema creation and pending
migrations run automatically on startup.

Run a single container with SQLite. Everything except IMAP/SMTP, OAuth and the
optional S3 bucket is local to the process.

### 3.2 Optional S3 / MinIO

```env
S3_ENDPOINT=http://minio:9000
S3_BUCKET=whelkmail
S3_REGION=us-east-1
S3_ACCESS_KEY_ID=...
S3_SECRET_ACCESS_KEY=...
S3_FORCE_PATH_STYLE=true
```

`docker compose --profile s3 up -d --build` also starts MinIO; create the bucket
once in its console at port 9001. Any S3-compatible service works (MinIO, AWS S3,
Backblaze B2, Cloudflare R2 through its S3 API). The UI labels the driver
**S3 / R2**; the stored driver name remains `r2`, so existing rows stay readable
when objects are copied to another bucket.

### 3.3 Cron

The container runs the notification/sync scheduler every minute when
`CRON_ENABLED=true` (default). It calls the same authenticated endpoint with
`CRON_SECRET`; if the variable is missing, a random per-process secret is
generated so the internal scheduler still works.

### 3.4 Updates

```bash
git pull
docker compose up -d --build
```

Migrations run on startup. The Docker target has no Workers CPU/subrequest
limits; IMAP sync can run for longer than on Cloudflare.

---

## 4. First login and account setup

1. Open the deployment URL.
2. Sign in with `APP_PASSWORD` (and the TOTP code if 2FA is enabled).
3. Click **Add account** in the sidebar or Settings.
4. Choose **IMAP/SMTP mailbox** or **Send-only (relay)**.
5. For IMAP: select a provider preset or enter hosts, ports and security;
   choose OAuth where supported, or a password/app password. Use **Test
   connection**.
6. For a relay: select the provider, paste its API key and choose where sent
   copies and drafts go: Google Drive, R2/S3 (if configured) or another
   account's mailbox. The mailbox destination needs a connected IMAP account.
7. Send a test message, save/reopen/delete a draft, then add a second account to
   check address labels and the unified inbox.

---

## 5. Migrations for older databases

Migrations are additive and recorded in `schema_migrations`; rerunning continues
from what is missing. Recent migrations:

| Migration | Adds |
|---|---|
| `020_relay_sent_copy` | `accounts.sent_copy_account_id` |
| `022_reply_account` | `accounts.reply_account_id` |
| `026_relay_any_address` | `accounts.relay_any_address` |
| `030_account_label_filters` | `filters.auto_label` and editable filters (1.2.0) |
| `031_recheck_aliases` | alias recheck after forwarding-header support (1.3.0) |
| `032_contact_source` | contacts source badge (1.3.0) |
| `033_auto_reply` | vacation responder tables and columns (1.3.0) |
| `034_rich_signature_autoreply` | signature and auto-reply HTML (1.3.0) |
| `035_contact_vcard_fields` | structured vCard contact fields (1.3.0) |

Run `npm run db:migrate:remote` (Cloudflare) or restart the container (Docker).
To force a starting point on D1:

```bash
node scripts/migrate.mjs remote --from=013
```

Migration `012` invalidates existing sessions; users sign in again.

---

## 6. Operations

### Logs

- Workers: `npx wrangler tail`
- Pages: project -> **Deployments -> Logs**, or
  `npx wrangler pages deployment tail --project-name whelkmail`
- Docker: `docker compose logs -f whelkmail`

### Rollback

- Workers: dashboard -> Worker -> **Deployments -> Rollback**
- Pages: project -> **Deployments** -> previous deployment -> **Rollback**
- Docker: redeploy the previous image tag and restart

### Database backup

```bash
# D1
npx wrangler d1 export mail-client --remote --output backup.sql

# Docker
docker run --rm -v whelkmail_whelkmail-data:/data -v "$PWD":/backup alpine \
  cp /data/whelkmail.sqlite /backup/whelkmail.sqlite
```

### Secret rotation

- Rotating `APP_PASSWORD` does not revoke existing sessions; they expire after
  7 days. Delete rows in `sessions` to revoke immediately.
- Rotating `TOKEN_ENC_KEY` invalidates all stored credentials, sessions and
  pending OAuth states. Users must re-enter passwords and reconnect OAuth.

---

## 7. Verification checklist

- `GET /api/health` returns `ok: true` and the expected `version`.
- Login succeeds with `APP_PASSWORD` (and TOTP if enabled).
- Security headers are present on `/` and `/api/health`.
- **Test connection** succeeds for an IMAP account.
- Sending adds a copy to the provider Sent folder.
- A draft can be saved, reopened and deleted.
- Two accounts produce colored labels and a unified inbox.
- Optional: Drive/R2/S3 archive works for relay accounts.
- Optional: OAuth reconnect works after a token refresh.

## 8. Costs and limits

- Workers/Pages free tiers exist, but IMAP/SMTP is CPU-heavy: Workers Paid (from
  $5/month) is recommended. Docker has no such CPU limit.
- D1 free tier: 5 GB storage plus daily read/write quotas.
- R2 free tier: 10 GB storage, free egress. S3/MinIO limits depend on the host.
- Google Drive: 15 GB shared with the Google account.
- No IMAP IDLE/push: the UI refreshes on navigation, after sending or on reload.

See [Troubleshooting](troubleshooting) for deployment errors and fixes.
