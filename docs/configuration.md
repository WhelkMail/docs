---
title: Configuration
description: Bindings, secrets, environment variables and optional integrations for WhelkMail.
---

# Configuration

IMAP/SMTP accounts need no central mail configuration: each account stores its own servers and credentials. The values below configure the deployment itself. Workers and Pages read Cloudflare bindings; Docker reads environment variables and creates the SQLite database and `BUCKET` adapter at startup.

## Bindings

| Binding | Targets | Required | Purpose |
|---|---|---|---|
| `DB` (D1) | Workers, Pages | Yes | Accounts, settings, IMAP sync metadata, relay archive/drafts metadata |
| SQLite file | Docker | Automatic | Same schema, stored in the `/data` volume |
| `BUCKET` (R2) | Workers, Pages | No | Sent/draft archive; optional IMAP body/attachment archive |
| `S3_*` | Docker | No | Same archive on any S3-compatible bucket; exposed to the code as `BUCKET` |

Use exactly these names on Cloudflare: the code reads `env.DB` and `env.BUCKET`. When storage is absent, the archive option is hidden in the UI.

## Required secrets

| Name | Type | Purpose |
|---|---|---|
| `APP_PASSWORD` | Secret | Login password for the whole app |
| `TOKEN_ENC_KEY` | Secret | 32-byte base64 key for AES-256-GCM encryption of stored credentials |

Generate the encryption key with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

> Changing `TOKEN_ENC_KEY` makes all stored credentials unreadable and invalidates sessions and pending OAuth states. Users must re-enter passwords and reconnect OAuth accounts.

## Optional variables

| Name | Purpose |
|---|---|
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google Drive archive for relay and IMAP accounts |
| `GOOGLE_MAIL_CLIENT_ID` / `GOOGLE_MAIL_CLIENT_SECRET` | Google OAuth for IMAP/SMTP |
| `MICROSOFT_CLIENT_ID` / `MICROSOFT_CLIENT_SECRET` / `MICROSOFT_TENANT` | Outlook / Microsoft 365 OAuth |
| `RELAY_PROVIDER` / `RELAY_API_KEY` / `RELAY_API_SECRET` | Deployment-wide relay fallback |
| `MAILGUN_DOMAIN` / `MAILGUN_REGION` | Mailgun relay fallback |
| `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_SUBJECT` | Web Push notifications |
| `CRON_SECRET` | Shared secret between the app and its scheduler (Worker trigger, Pages cron worker or Docker scheduler) |
| `UPDATE_CHECK` | Set to `off` to stop the outbound release check |
| `GITHUB_TOKEN` | Optional token, only used to raise the release-check rate limit |

Relay credentials can also be entered per account in the UI and are encrypted at rest. Per-account values take precedence over the deployment fallbacks.

`MICROSOFT_TENANT` defaults to `common` for personal and work accounts.

## Environment reference

| Name | Required | Purpose |
|---|---|---|
| `APP_PASSWORD` | Yes | App login |
| `TOKEN_ENC_KEY` | Yes | AES-GCM key for stored credentials |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | No | Drive archive |
| `GOOGLE_MAIL_CLIENT_ID` / `GOOGLE_MAIL_CLIENT_SECRET` | No | Google OAuth for IMAP/SMTP |
| `MICROSOFT_CLIENT_ID` / `MICROSOFT_CLIENT_SECRET` | No | Outlook OAuth |
| `MICROSOFT_TENANT` | No | Defaults to `common` |
| `RELAY_PROVIDER` | No | Default relay fallback |
| `RELAY_API_KEY` | No | Default relay key fallback |
| `RELAY_API_SECRET` | No | Mailjet fallback secret |
| `MAILGUN_DOMAIN` / `MAILGUN_REGION` | No | Mailgun fallback |
| `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` | No | Web Push VAPID keys |
| `VAPID_SUBJECT` | No | VAPID contact (`mailto:`) |
| `CRON_SECRET` | No | Shared secret for the scheduler |
| `DB` (D1 binding) | Yes on Cloudflare | Metadata |
| `BUCKET` (R2 binding) | No | R2 archive on Cloudflare |
| `S3_ENDPOINT` / `S3_BUCKET` / `S3_REGION` | No | S3-compatible archive on Docker |
| `S3_ACCESS_KEY_ID` / `S3_SECRET_ACCESS_KEY` | No | S3 credentials on Docker |
| `S3_FORCE_PATH_STYLE` | No | Path-style URLs (default `true`, suited to MinIO) |
| `DB_PATH` | No | SQLite file, default `/data/whelkmail.sqlite` in Docker |
| `PUBLIC_ORIGIN` | No | Public URL for OAuth callbacks and secure cookies on Docker |
| `PORT` / `HOST` / `STATIC_DIR` | No | Docker listener and build directory |
| `CRON_ENABLED` | No | `false` disables the Docker scheduler |
| `UPDATE_CHECK` | No | `off` disables the release check |
| `GITHUB_TOKEN` | No | Raises the release-check rate limit |

## Deployment-wide settings in D1

Some preferences are stored in the `app_settings` table instead of environment variables, and are edited from the UI:

- `default_signature`, `default_signature_html`, `signature_mode` — shared signature or one per account, plain text plus rich HTML.
- `contact_auto_add` — Gmail-style Other contacts from sent recipients. On by default; `0` leaves the address book manual.
- `auto_reply_mode`, `auto_reply_enabled`, `auto_reply_subject`, `auto_reply_body`, `auto_reply_body_html` — shared vacation responder. See [Auto-reply](auto-reply).
- `allow_remote_images` — global default for remote images.
- `metadata_retention_days`, `retention_stats` — metadata retention window and last cleanup.
- TOTP/2FA material.
- `update_check`, `update_notice_dismissed` — release-check cache and dismissal.

## Configuration export and import

**Settings → System → Configuration export and import** moves the configuration to another WhelkMail deployment. One JSON file carries accounts, labels, filters, contacts, notification preferences and the settings listed above as deployment-wide settings. Mailboxes, cached bodies and the R2/S3/Drive archive are not part of the file.

- **Export** (`POST /api/config/export`) decrypts the stored account secrets with the current `TOKEN_ENC_KEY`, then encrypts the whole bundle with a passphrase you choose (PBKDF2-SHA256 with 100,000 iterations, then AES-256-GCM). The file is safe at rest and the passphrase cannot be recovered.
- **Import** (`POST /api/config/import`) decrypts the file with the passphrase, re-encrypts every account secret with the destination `TOKEN_ENC_KEY` and replaces the configuration tables. Two installations may therefore use different keys.

An import replaces accounts, filters, labels, contacts and the settings it manages, and clears the local mail mirror (mail stays on the providers and is rebuilt by the next sync). Sessions, two-factor settings and the object archive are left untouched. Push subscriptions are device-specific and are not portable: subscribe again on the new device. If an account secret cannot be decrypted with the source key, the export stops and names the account, so re-enter its password first.

The 100,000-iteration ceiling is the Cloudflare Workers limit; the same value is used on every target so a file exported by one deployment can be imported by another. The derivation runs inside the Worker, so on a Workers Free plan a large export may exceed the free CPU budget; use Docker or Workers Paid when needed.

## Local environment file

For local Cloudflare development, `.dev.vars` (gitignored) holds the same values:

```env
APP_PASSWORD=change-me
TOKEN_ENC_KEY=<32-byte base64 key>
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_MAIL_CLIENT_ID=
GOOGLE_MAIL_CLIENT_SECRET=
MICROSOFT_CLIENT_ID=
MICROSOFT_CLIENT_SECRET=
MICROSOFT_TENANT=common
RELAY_PROVIDER=smtp2go
RELAY_API_KEY=
```

For Docker, copy `deploy/docker/.env.example` to `deploy/docker/.env`. The
minimum is `APP_PASSWORD`, `TOKEN_ENC_KEY` and `PUBLIC_ORIGIN`; add `S3_*` for
the archive, OAuth/relay/VAPID values when needed, and set `CRON_ENABLED=false`
if the scheduler should not run. `PUBLIC_ORIGIN` must match the public URL when
a reverse proxy terminates TLS.

## OAuth

Google Drive, Google IMAP/SMTP and Microsoft OAuth each need an OAuth application and matching redirect URIs. See [OAuth setup](oauth).

## Signatures

`signature_mode` is `shared` (default) or `per_account`:

- `shared`: `default_signature` / `default_signature_html` are used for every account and `accounts.signature` is ignored.
- `per_account`: every account uses its own `accounts.signature` / `accounts.signature_html`; an empty value means no signature.

Signatures are edited with the rich-text composer. `signature_html` is the HTML part; `signature` remains the plain-text fallback. The composer's default rich/plain format is a per-device preference stored in the browser (`localStorage: mc_compose_rich`), not in D1.

## Auto-reply

The shared or per-account vacation responder is documented in [Auto-reply](auto-reply). It stores `auto_reply_*` keys in `app_settings` (shared) or the matching columns on `accounts` (per account).
