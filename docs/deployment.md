---
title: Deployment
description: The complete step-by-step guide to run WhelkMail on Cloudflare Pages, D1 and Workers.
---

# Deploying WhelkMail on Cloudflare

This is the complete, step-by-step guide to run WhelkMail on Cloudflare. You do not need a VPS, a mail server, or an external backend.

> **Important:** this is not a static site. Deploy it as a **Pages project with Functions**. The `functions/` directory is the API (IMAP/SMTP over `cloudflare:sockets`). Uploading only `dist` without Functions will break every API call.

## Fast path: the installer (beta)

The [WhelkMail Installer](https://github.com/WhelkMail/installer) automates the resource creation and deployment described in this guide. It runs against **your** Cloudflare account and performs the same steps: it creates (or reuses) the D1 database `mail-client` and the Pages project, writes the `DB` binding and the optional `BUCKET` binding, uploads `APP_PASSWORD`, `TOKEN_ENC_KEY` and the optional cron/VAPID secrets, applies the schema and migrations, deploys `dist` and verifies `GET /api/health`.

- **Browser:** copy the installer repository, add the required secrets in GitHub, then run the **Install WhelkMail** workflow.
- **Terminal:** `npx github:WhelkMail/installer` (flags such as `--dry-run`, `--r2`, `--cron`, `--push`, `--ref <tag>`, `--project-name <name>`).

The installer is **beta**. It is a convenience on top of the manual process below, which remains the reference path and the fallback if a run fails. Re-running the installer is safe: existing resources are reused, secrets are overwritten and database migrations are idempotent. To try the app first without deploying, use the [live demo](https://demo.whelkmail.eu) — a static, backend-free preview with mocked API calls and fake data.

## What you will create

| Cloudflare resource | Purpose |
|---|---|
| Pages project | Static UI + Pages Functions (`functions/`) |
| D1 database | Accounts, settings, IMAP sync metadata, relay archive/drafts metadata |
| Secrets | App password, encryption key, optional OAuth and relay keys |
| R2 bucket (optional) | Sent/draft archive for relay accounts; optional IMAP body/attachment archive |
| Custom domain (optional) | HTTPS hostname for the app |

## Before you start

- A Cloudflare account.
- A GitHub repository containing this project.
- Node.js 22 or newer and npm.
- Wrangler authenticated:

  ```bash
  npx wrangler login
  ```

- A Workers Paid plan is strongly recommended. IMAP/SMTP TLS handshakes and authentication routinely exceed the 10 ms CPU budget of the free plan.

## 1. Deploy from GitHub (recommended)

### 1.1 Create the D1 database

```bash
npx wrangler d1 create mail-client
```

The command prints a `database_id`. Use it in one of two ways:

- **GitHub deployment:** set the `DB` binding in the Pages dashboard; `wrangler.toml` can stay local and untracked.
- **CLI/local:** copy `wrangler.toml.example` to `wrangler.toml` and paste the ID:

```toml
[[d1_databases]]
binding = "DB"
database_name = "mail-client"
database_id = "<paste-the-database-id-here>"
```

The zero UUID in the example is fine for local development. For remote CLI commands and deployments that read `wrangler.toml`, replace it with the real D1 ID or the Functions will fail.

> The D1 database intentionally keeps the legacy name `mail-client` so existing deployments and data continue to work after the whelkmail rebrand. Only the Pages project is named `whelkmail`.

Dashboard equivalent: **Storage & Databases → D1 SQL Database → Create**.

### 1.2 Create the Pages project from GitHub

1. Open **Workers & Pages** in the Cloudflare dashboard.
2. **Create** → **Pages** → **Connect to Git**.
3. Select the repository and branch.
4. Project name: `whelkmail`.
5. Build settings:

| Setting | Value |
|---|---|
| Framework preset | None / Vite |
| Build command | `npm run build` |
| Build output directory | `dist` |
| Root directory | `/` |

Add an environment variable `NODE_VERSION=22` in the Pages project settings if the default Node version is older.

`functions/` is detected automatically; no extra configuration is needed.

### 1.3 Add bindings

In the Pages project: **Settings → Bindings**.

Required:

| Binding name | Type | Target |
|---|---|---|
| `DB` | D1 database | `mail-client` |

Optional:

| Binding name | Type | Target |
|---|---|---|
| `BUCKET` | R2 bucket | any bucket you create |

Use exactly these names. The code reads `env.DB` and `env.BUCKET`.

### 1.4 Add secrets and variables

In the Pages project: **Settings → Variables and Secrets**. Add them to **Production** and, if you use preview deployments, to **Preview** as well.

Required:

| Name | Type | Value |
|---|---|---|
| `APP_PASSWORD` | Secret | the login password for the whole app |
| `TOKEN_ENC_KEY` | Secret | 32 random bytes encoded in base64 |

Generate the key:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

CLI equivalent:

```bash
npx wrangler pages secret put APP_PASSWORD --project-name whelkmail
npx wrangler pages secret put TOKEN_ENC_KEY --project-name whelkmail
```

Optional values are listed in [Configuration](configuration).

> Changing `TOKEN_ENC_KEY` later invalidates all stored IMAP/SMTP passwords, OAuth refresh tokens and relay API keys. Plan key rotation accordingly.

### 1.5 Apply the database schema

```bash
npm run db:init:remote
```

This runs:

```bash
wrangler d1 execute mail-client --remote --file=./db/schema.sql
```

Verify:

```bash
npx wrangler d1 execute mail-client --remote \
  --command "SELECT name FROM sqlite_master WHERE type='table';"
```

You should see `accounts` and `messages` (plus the other tables).

If the database already existed before labels, signatures or later features were added, apply the additive migrations:

```bash
npm run db:migrate:remote
```

The migrations are additive and recorded in `schema_migrations`, so a plain rerun only continues with what is left. To force a starting point:

```bash
node scripts/migrate.mjs remote --from=013
```

Migration `012` also invalidates existing sessions: users must sign in again. Two-factor authentication is available in Settings → Security and is disabled by default.

### 1.6 Web Push notifications (optional)

Generate a VAPID key pair:

```bash
npm run vapid
```

Add to the Pages project:

```bash
npx wrangler pages secret put VAPID_PUBLIC_KEY --project-name whelkmail
npx wrangler pages secret put VAPID_PRIVATE_KEY --project-name whelkmail
npx wrangler pages secret put CRON_SECRET --project-name whelkmail
```

`VAPID_SUBJECT` can be a plain variable such as `mailto:you@example.com`.

The cron worker config is personal, so it is gitignored. Copy the template, set your Pages URL and deploy:

```bash
cp worker/wrangler.toml.example worker/wrangler.toml
# set APP_URL in worker/wrangler.toml to https://<your-project>.pages.dev
npx wrangler secret put CRON_SECRET -c worker/wrangler.toml
npm run cron:deploy
```

`npm run cron:deploy` creates the file from the template when it is missing and stops with a clear error while `APP_URL` is still empty or a placeholder.

The worker wakes up every minute and calls `POST /api/cron/notifications` on the Pages app. In the same run it applies filter labels in every folder, forwards matching inbox messages, runs the optional provider cleanup, and advances each account's automatic background sync when its own interval has elapsed. The template uses `crons = ["* * * * *"]`; on Workers Free, `["*/5 * * * *"]` is recommended. Enable notifications from **Settings → Push notifications**, background sync per account in Settings, and forwarding/deletion in the account/filter sections.

See [PWA and Web Push](pwa-push) for the full setup.

### 1.7 Deploy

- **Git integration:** push to the connected branch. Cloudflare builds and deploys automatically.
- **Manual retry:** Pages project → **Deployments** → **Retry deployment**.

The first deployment URL is `https://whelkmail.pages.dev` (or the project name you chose).

### 1.8 First login and account setup

1. Open the deployment URL.
2. Sign in with `APP_PASSWORD`.
3. Click **Add account** (Sidebar or Settings).
4. Choose **IMAP/SMTP mailbox** or **Send-only (relay)**.
5. For an IMAP mailbox:
   - Select the provider preset or enter IMAP/SMTP hosts, ports and security manually.
   - Choose OAuth where supported, or enter a password / app password.
   - Use **Test connection** in Settings.
6. For a send-only domain:
   - Select the relay provider and paste its API key.
   - Choose where sent copies and drafts go: **Google Drive**, **R2** (only if the `BUCKET` binding exists) or **another account's mailbox**. The third option appends every message to that IMAP account's Sent folder and every draft to its Drafts folder, and needs a connected IMAP account: without one the option stays disabled and saving drafts would fail (`sent_copy_target_not_connected`).
   - Only one destination is active at a time; switching back to Drive/R2 clears the mailbox choice.
7. Send a test message.
8. Save a draft, reopen it from **Drafts**, and delete it. With a mailbox destination, check that the draft also shows up in that account's Drafts folder on the provider.
9. Add a second account to see address labels and the unified inbox.

### 1.9 Custom domain

1. Pages project → **Custom domains** → **Set up a custom domain**.
2. Enter the hostname and follow the DNS instructions.
3. Wait for the certificate to become active.
4. If you use OAuth, update the redirect URIs to the new hostname.

## 2. Deploy from the CLI instead of GitHub

```bash
npx wrangler pages project create whelkmail
npm run build
npx wrangler pages deploy dist --project-name whelkmail
```

Or simply:

```bash
npm run deploy
```

`npm run deploy` runs `npm run build` and then `wrangler pages deploy dist`.

With CLI deployments, bindings and secrets come from the project configuration and your local `wrangler.toml`. With GitHub deployments, configure the D1 binding and secrets in the Pages dashboard instead; keep `wrangler.toml` local and untracked.

## 3. Migrations for older databases

If the database already existed before relay support was added, apply the missing migrations with the script rather than by hand:

```bash
npm run db:migrate:remote
```

It records what it applied, so a rerun only continues with what is left. Relevant relay migrations:

| Migration | Adds |
|---|---|
| `020_relay_sent_copy` | `accounts.sent_copy_account_id` (Drive/R2/IMAP destination of the sent copies) |
| `022_reply_account` | `accounts.reply_account_id` (which relay answers for a forwarding mailbox) |
| `026_relay_any_address` | `accounts.relay_any_address` (`Send from any address on this domain`) |

Other recent migrations:

| Migration | Adds |
|---|---|
| `030_account_label_filters` | `filters.auto_label`, the unique index for each account's automatic filter, and the conversion of existing `incoming_label_id` settings into editable filters (1.2.0) |
| `031_recheck_aliases` | Marks rows with an empty alias list for one more read after the forwarding headers were added to the IMAP fetch (1.3.0) |
| `032_contact_source` | `contacts.source` for the Other contacts badge (1.3.0) |
| `033_auto_reply` | Per-account vacation-responder columns, `auto_reply_log` and its indexes (1.3.0) |
| `034_rich_signature_autoreply` | `accounts.signature_html` and `accounts.auto_reply_body_html` (1.3.0) |
| `035_contact_vcard_fields` | Structured contact fields: first/last name, company, job title, phone, address, website, birthday, additional emails/phones (1.3.0) |

For a database so old that it predates relay support entirely:

```sql
ALTER TABLE accounts ADD COLUMN relay_api_key_enc TEXT;
ALTER TABLE accounts ADD COLUMN relay_api_secret_enc TEXT;
ALTER TABLE accounts ADD COLUMN relay_domain TEXT;
ALTER TABLE accounts ADD COLUMN relay_region TEXT;
```

## 4. Optional R2 archive

R2 is used by send-only relay accounts and by IMAP accounts that enable the optional body/attachment archive. By default IMAP accounts keep everything on the mail provider.

1. **R2 Object Storage → Create bucket**.
2. Pages project → **Settings → Bindings** → add an R2 bucket binding named `BUCKET`.
3. Redeploy.

If the binding is absent, the R2 option is hidden in the UI.

## 5. Verification checklist

- `GET /api/health` returns `ok: true`.
- Login succeeds with `APP_PASSWORD` (plus the TOTP code if 2FA is enabled).
- Optional: 2FA can be enabled in Settings → Security, and a recovery code can complete the login.
- Security headers are present on `/` and `/api/health`.
- **Test connection** succeeds for an IMAP account.
- Sending adds a copy to the provider Sent folder (or the app appends it when `auto_saves_sent` is off).
- A draft can be saved, reopened and deleted.
- Two accounts produce colored labels and a unified inbox.
- Optional: Drive/R2 archive works for relay accounts and for IMAP accounts with body archiving enabled.
- Optional: OAuth reconnect works after a token refresh.

## 6. Operations

### Logs

Pages project → **Deployments** → select deployment → **Logs**, or:

```bash
npx wrangler pages deployment tail --project-name whelkmail
```

### Rollback

Pages project → **Deployments** → select a previous deployment → **Rollback**.

### D1 backup

```bash
npx wrangler d1 export mail-client --remote --output backup.sql
```

### Schema updates

Re-run `npm run db:remote` after changing `db/schema.sql`. `CREATE TABLE IF NOT EXISTS` does not alter existing tables; use `ALTER TABLE` for additive changes.

### Secret rotation

- Rotating `APP_PASSWORD` does not revoke existing sessions automatically; they remain valid until their 7-day expiry. To revoke all sessions immediately, delete the rows in `sessions` (or change `TOKEN_ENC_KEY`, which also invalidates credentials).
- Rotating `TOKEN_ENC_KEY` invalidates all stored credentials, sessions and pending OAuth states. Users must re-enter passwords and reconnect OAuth accounts.

## 7. Costs and limits

- Pages and Workers have free quotas, but IMAP/SMTP is CPU-heavy: use Workers Paid (from $5/month).
- D1 free tier: 5 GB storage and daily read/write quotas.
- R2 free tier: 10 GB storage, free egress.
- Google Drive: 15 GB shared with the Google account.
- No IMAP IDLE/push: the UI refreshes on navigation, after sending, or on reload.

See [Troubleshooting](troubleshooting) for deployment errors and their fixes.
