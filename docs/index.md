---
title: Introduction
description: WhelkMail is a self-hosted, Cloudflare-native webmail client for the IMAP and SMTP accounts you already own.
---

# WhelkMail documentation

**Your mailboxes. Your domain. One calm inbox.**

WhelkMail is a self-hosted webmail client for the IMAP and SMTP accounts you already own. It runs entirely on Cloudflare Pages and Workers, so there is no mail server to operate and no infrastructure to babysit.

Connect Gmail, Outlook, iCloud, Fastmail, Yahoo or a custom domain and get a familiar, Gmail-style interface on top of the mailboxes you already use. Messages, sent copies, drafts and attachments stay on your provider; only a short-lived D1 cache and the strictly opt-in long-term archive copy them.

> **WhelkMail is not a mail server.** It talks IMAP and SMTP to the providers you already trust.

![WhelkMail inbox](/assets/images/screenshots/inbox.png)

## Try it, or install it

- **Live demo:** [demo.whelkmail.eu](https://demo.whelkmail.eu) is a static, backend-free preview of WhelkMail. It runs the real UI but replaces every API call with an in-memory mock, so there is no D1, no IMAP/SMTP and no network access. Every account, message and attachment is fake, and actions only change the current browser tab.
- **Installer (beta):** the [WhelkMail Installer](https://github.com/WhelkMail/installer) provisions and deploys a WhelkMail instance inside **your** Cloudflare account. It creates the Pages project and the D1 database, stores `APP_PASSWORD` and `TOKEN_ENC_KEY`, deploys the app and checks `GET /api/health`. You can run it from GitHub Actions in the browser or from the terminal with `npx github:WhelkMail/installer`. The installer is a convenience on top of the manual deployment described in [Deployment](deployment); manual deployment remains fully supported.

## Why WhelkMail

- **One inbox, many accounts.** Gmail, Outlook, iCloud, Yahoo and custom domains side by side, with colored account labels and a shared address book.
- **Cloudflare-native.** Pages Functions, D1, R2 and `cloudflare:sockets`: no VPS, no container, no mail server to patch.
- **Your data stays put.** By default the client reads from the provider. A short-lived D1 cache keeps recently listed or opened bodies for fast opens; the long-term archive on R2 or Google Drive remains strictly opt-in.
- **Privacy by default.** HTML email is rendered in a sandboxed frame, remote images and tracking pixels stay blocked unless you enable them, and credentials are encrypted at rest.
- **Actually yours.** Self-hosted, MIT licensed, no telemetry, no vendor account required beyond your Cloudflare project.

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
- Send-only "relay" accounts for domains without SMTP, with sent copies and drafts stored on Google Drive, Cloudflare R2 or an existing IMAP mailbox.
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
- Incremental, resumable IMAP metadata sync into D1.
- Optional background sync through the cron worker, with a per-account interval (1 to 30 minutes).
- Anti-loop forwarding headers, per-filter forward logs and deletion logs.

## Explore the documentation

- [Getting started](getting-started) — run WhelkMail locally and connect your first account.
- [Deployment](deployment) — the complete Cloudflare Pages guide.
- [Configuration](configuration) — bindings, secrets and environment variables.
- [Architecture](architecture) — how the client, Pages Functions, D1 and IMAP/SMTP fit together.
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
- Cloudflare Pages Functions (Workers runtime)
- Cloudflare D1 for accounts and metadata
- `cloudflare:sockets` for IMAP/SMTP
- Optional Cloudflare R2 or Google Drive for archive
- `postal-mime` for MIME parsing

## License

MIT. See [License](license).
