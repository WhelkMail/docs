---
title: Getting started
description: Requirements, local setup and connecting your first mailbox to WhelkMail.
---

# Getting started

## Fast paths

- **Workers (default):** use the [Deploy to Cloudflare button](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail). Cloudflare clones the repository into your GitHub/GitLab account, provisions D1/R2, configures Workers Builds and deploys. Enter the secrets during setup or in the dashboard.
- **Cloudflare Pages:** create the project in the [Pages dashboard](https://dash.cloudflare.com/?to=/:account/pages), connect the repository and add the `DB`/`BUCKET` bindings and secrets there. The full manual walkthrough is in [Deployment](deployment#2-cloudflare-pages).
- **Docker:** `cd deploy/docker && cp .env.example .env && docker compose up -d --build`. See [Deployment](deployment#3-docker--any-vps).

Each deployment target stores the same two required secrets (`APP_PASSWORD`, `TOKEN_ENC_KEY`) and applies the schema before first use: `npm run db:init:workers:remote` on Workers, `npm run db:init:remote` on Pages, automatic on Docker.

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

The repository tracks `wrangler.jsonc` without a `database_id`, so local dev and the Deploy to Cloudflare flow can provision D1 automatically. For a real CLI deployment, create your own database with `npx wrangler d1 create mail-client --binding DB --update-config`, or configure the `DB` binding in the Pages dashboard. Remove any legacy local `wrangler.toml`: Pages does not support custom configuration paths.

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
