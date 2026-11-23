---
title: Getting started
description: Requirements, local setup and connecting your first mailbox to WhelkMail.
---

# Getting started

## Fast paths

- **Workers (default):** use the [Deploy to Cloudflare button](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail). Cloudflare clones the repository into your GitHub/GitLab account, provisions D1/R2, configures Workers Builds and deploys. Enter the secrets during setup or in the dashboard.
- **Pages installer (beta):** the [WhelkMail Installer](https://github.com/WhelkMail/installer) creates a Pages deployment in **your** Cloudflare account. Browser path: copy the [installer template](https://github.com/WhelkMail/installer), add `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `APP_PASSWORD` and `TOKEN_ENC_KEY` as repository secrets, then run **Install WhelkMail**. Terminal path: with Node.js 24+ run `npx github:WhelkMail/installer`.
- **Docker:** `cd deploy/docker && cp .env.example .env && docker compose up -d --build`. See [Deployment](deployment#3-docker--any-vps).

The installer stores secrets, creates the Pages project and the D1 database, deploys the app and verifies the health endpoint. It is still **beta**, so check the run log; the manual path in [Deployment](deployment) is the fully documented fallback. Re-running it is safe: resources and the database are reused and migrations are idempotent.

To look around without deploying anything, open the [live demo](https://demo.whelkmail.eu). It is a static preview of the real UI where every API call is mocked in the browser: no real mailbox, no network access and no data leaves the tab.

## Requirements

- Node.js 24 or newer and npm (the Docker image uses Node 24).
- A Cloudflare account for the Workers/Pages targets, or Docker on a VPS.
- An IMAP/SMTP mailbox you already own, or a relay provider account for a send-only domain.

## Local setup

```bash
npm install
npm run db:init:local
```

The repository tracks `wrangler.jsonc` with a placeholder D1 ID; for local development that is fine. For a real deployment, create your own database and replace `database_id`, or configure the `DB` binding in the Pages dashboard. Remove any legacy local `wrangler.toml`: Pages does not support custom configuration paths and the public repository must keep the placeholder.

If you already have a local database created before labels/signatures, run `npm run db:migrate:local`. If you only need the security tables (sessions, rate limiting, OAuth state), start from that migration with `node scripts/migrate.mjs local --from=012`; existing sessions are invalidated.

## Environment file

Create `.dev.vars` in the project root (it is gitignored):

```env
APP_PASSWORD=change-me
TOKEN_ENC_KEY=<32-byte base64 key>
```

Generate the key with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

## Run the app

Workers (default) and Pages can share the same local D1 database:

```bash
npm run db:init:workers:local   # or db:init:local
npm run dev                     # Vite on http://localhost:5173
npm run dev:workers             # Worker + local D1 on http://localhost:8787
```

Pages alternative:

```bash
npm run dev:pages               # Pages Functions on http://localhost:8788
```

Open `http://localhost:8787` (Workers) or `http://localhost:8788` (Pages) and
sign in with `APP_PASSWORD`.

Node/Docker locally:

```bash
npm run build:all
APP_PASSWORD=... TOKEN_ENC_KEY=... DB_PATH=./data/dev.sqlite \
  PUBLIC_ORIGIN=http://localhost:8787 node dist-server/server.mjs
```

## Connect your first account

1. Sign in with `APP_PASSWORD` (plus the TOTP code if you enabled two-factor authentication).
2. Click **Add account** in the sidebar or in Settings.
3. Choose **IMAP/SMTP mailbox** or **Send-only (relay)**.
4. For an IMAP mailbox:
   - Pick a provider preset or enter IMAP/SMTP hosts, ports and security manually.
   - Choose OAuth where supported, or enter a password / app password.
   - Use **Test connection** in Settings.
   - Optionally choose an automatic label: the label is created (or reused) and becomes an editable match-all filter that applies to every folder of the account.
5. For a send-only domain:
   - Select the relay provider and paste its API key.
   - Choose where sent copies and drafts go: **Google Drive**, **R2/S3** (only if storage is configured) or **another account's mailbox**.
6. Send a test message.

| Account settings | Connect an address |
| --- | --- |
| ![Account settings](/assets/images/screenshots/settings.png) | ![Connect an address](/assets/images/screenshots/add-account.png) |

## Authentication per provider

| Provider | Method |
|---|---|
| Google accounts | OAuth2 (recommended) or app password (2-Step Verification required) |
| Outlook / Microsoft 365 | OAuth2 + XOAUTH2 (basic auth is no longer supported by Microsoft) |
| Yahoo | App password |
| iCloud | App-specific password |
| Generic IMAP | Account password or app password |
| Relay accounts | Provider HTTP API key (+ secret/domain where required) |

OAuth application setup is covered in [OAuth setup](oauth).

## First checks

- Save a draft, reopen it from **Drafts**, and delete it.
- Add a second account to see address labels and the unified inbox.
- Confirm that a sent message appears in the provider's Sent folder.

## Next steps

- [Deployment](deployment) — Workers, Pages or Docker.
- [Configuration](configuration) — all bindings, secrets and variables.
- [PWA and Web Push](pwa-push) — install the app and enable notifications.
