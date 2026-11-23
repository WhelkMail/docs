---
title: Introduction
description: WhelkMail is a self-hosted webmail client for the IMAP and SMTP accounts you already own, deployable on Cloudflare Workers, Pages or Docker.
---

# WhelkMail documentation

**Your mailboxes. Your domain. One calm inbox.**

WhelkMail is a self-hosted webmail client for the IMAP and SMTP accounts you already own. It runs on **Cloudflare Workers** by default, on **Cloudflare Pages** for existing installations, or in a **single Docker container** on any VPS, so there is no mail server to operate and no infrastructure to babysit.

Connect Gmail, Outlook, iCloud, Fastmail, Yahoo or a custom domain and get a familiar, Gmail-style interface on top of the mailboxes you already use. Messages, sent copies, drafts and attachments stay on your provider; only a short-lived D1 cache and the strictly opt-in long-term archive copy them.

> **WhelkMail is not a mail server.** It talks IMAP and SMTP to the providers you already trust.

![WhelkMail inbox](/assets/images/screenshots/inbox.png)

## Try it, or install it

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail)

- **Live demo:** [demo.whelkmail.eu](https://demo.whelkmail.eu) is a static, backend-free preview of WhelkMail. It runs the real UI but replaces every API call with an in-memory mock, so there is no D1, no IMAP/SMTP and no network access. Every account, message and attachment is fake, and actions only change the current browser tab.
- **Deploy to Cloudflare (Workers, default):** the [Deploy to Cloudflare button](https://deploy.workers.cloudflare.com/?url=https://github.com/WhelkMail/WhelkMail) clones the repository into your GitHub/GitLab account, provisions D1/R2 and deploys the Worker.
- **Installer (Pages, beta):** the [WhelkMail Installer](https://github.com/WhelkMail/installer) provisions and deploys a **Pages** instance inside **your** Cloudflare account. It creates the Pages project and the D1 database, stores `APP_PASSWORD` and `TOKEN_ENC_KEY`, deploys the app and checks `GET /api/health`. You can run it from GitHub Actions in the browser or from the terminal with `npx github:WhelkMail/installer`. The installer is a convenience on top of the manual deployment described in [Deployment](deployment).

## Why WhelkMail

- **One inbox, many accounts.** Gmail, Outlook, iCloud, Yahoo and custom domains side by side, with colored account labels and a shared address book.
- **Cloudflare-first, not Cloudflare-only.** Workers with D1, R2 and `cloudflare:sockets` by default; the same code runs on Pages or in Docker with SQLite and any S3-compatible bucket.
- **Your data stays put.** By default the client reads from the provider. A short-lived database cache (D1 or SQLite) keeps recently listed or opened bodies for fast opens; the long-term archive on R2/S3 or Google Drive remains strictly opt-in.
- **Privacy by default.** HTML email is rendered in a sandboxed frame, remote images and tracking pixels stay blocked unless you enable them, and credentials are encrypted at rest.
- **Actually yours.** Self-hosted, MIT licensed, no telemetry; run it on your Cloudflare account or on your own server.

## What you get

### Inbox experience

- Message list with a full-width reading view or an optional right-side reading pane, plus Gmail-style multi-select for delete, move and spam actions.
- Inbox, Sent, Drafts and Spam with autosave and reopenable drafts.
- Gmail-style conversations using RFC `References` when available and a subject fallback otherwise.
- HTML bodies in a sandboxed frame with optional remote images and attachment indicators from `BODYSTRUCTURE`.
- Rich-text composer with bold, lists, links and quotes; the default rich/plain format is saved per device, and Enter in a recipient never sends the message.
- Installable PWA, mobile drawer layout and Web Push notifications with per-account/folder preferences.
- English by default, Italian available; the dictionary is extensible.

### Accounts and sending

- Multiple accounts in one view, each with its own IMAP/SMTP servers and credentials.
- Password, app-password, Google OAuth2 and Microsoft OAuth2 (XOAUTH2) authentication.
- Sending through each account's own SMTP server.
- Replies from the right address, including forwarding domains and relay accounts that send from any address on their domain.
- Send-only "relay" accounts for domains without SMTP, with sent copies and drafts stored on Google Drive, R2/S3 or an existing IMAP mailbox.
- Optional archive and provider cleanup for IMAP.

### Organization

- Gmail-style labels shared across all accounts.
- Per-account automatic labels, stored as editable match-all filters. A relay account's label follows its alias into the IMAP mailbox that receives the forwarded mail.
- Structured contacts with name, company, job title, phone, address, website, birthday, extra emails/phones and notes; vCard 3.0/4.0 import with automatic version detection, vCard 3.0 default export and 4.0 optional.
- Gmail-style **Other contacts** for autocomplete, with an explicit Add to contacts entry while typing and silent addition when replying.
- Signatures: one shared signature or a different signature per account, with rich-text formatting.

### Automation

- Gmail-style filters that target every account or a single one: match From/To/Subject/words/attachments, apply labels in every folder and/or forward incoming mail. Word criteria are re-evaluated once the body is indexed, and forwarded aliases are read from the real headers.
- Vacation responder: one shared auto-reply or a different one per account, with rich-text body and anti-loop rules.
- Incremental, resumable IMAP metadata sync into D1/SQLite.
- Optional background sync through the target scheduler (Worker cron, Pages cron worker or the Docker scheduler), with a per-account interval (1 to 30 minutes).
- Anti-loop forwarding headers, per-filter forward logs and deletion logs.
- Configuration export and import: move accounts, filters, labels, contacts and settings between deployments as a passphrase-encrypted file, with secrets re-encrypted for the target `TOKEN_ENC_KEY`.

## Explore the documentation

- [Getting started](getting-started) — run WhelkMail locally and connect your first account.
- [Deployment](deployment) — Workers, Pages and Docker side by side.
- [Configuration](configuration) — bindings, secrets and environment variables.
- [Architecture](architecture) — how the client, the API core, D1/SQLite and IMAP/SMTP fit together.
- [Mail flows](mail-flows) — listing, search, reading, sending, replies, threads and drafts.
- [Data model](data-model) — every D1 table and what it stores.
- [Security](security) — authentication, sessions, encryption and hardening.
- [Relay accounts](relay-accounts) — send-only domains and storage destinations.
- [Labels and filters](labels-filters) — shared taxonomy, automatic rules, contacts and signatures.
- [Auto-reply](auto-reply) — shared or per-account vacation responder.
- [PWA and Web Push](pwa-push) — installable app and notifications.
- [Forwarding and cleanup](forwarding-cleanup) — filter forwarding and provider deletion.
- [Development](development) — local workflow, database migrations and tests.
- [Upgrading](upgrading) — deploy a new release safely.
- [Troubleshooting](troubleshooting) — common symptoms and fixes.
- [Known limitations](limitations) — what WhelkMail deliberately does not do.

## Stack

- React 19 + Vite 8 + TypeScript + Tailwind CSS
- One runtime-agnostic API core with three thin entrypoints
- Cloudflare Workers (default) or Pages Functions, sharing D1 and R2
- Docker / Node 24 with built-in SQLite (`node:sqlite`)
- `cloudflare:sockets` on Cloudflare, `node:net`/`node:tls` on Node
- Optional R2, S3/MinIO or Google Drive for archive
- `postal-mime` for MIME parsing

## License

MIT. See [License](license).
