---
title: Configuration
description: Bindings, secrets, environment variables and optional integrations for WhelkMail.
---

# Configuration

IMAP/SMTP accounts need no central mail configuration: each account stores its own servers and credentials. The values below configure the deployment itself.

## Bindings

| Binding | Required | Purpose |
|---|---|---|
| `DB` (D1) | Yes | Accounts, settings, IMAP sync metadata, relay archive/drafts metadata |
| `BUCKET` (R2) | No | Sent/draft archive for relay accounts; optional IMAP body/attachment archive |

Use exactly these names: the code reads `env.DB` and `env.BUCKET`. When `BUCKET` is absent, R2 is hidden in the UI.

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
| `CRON_SECRET` | Shared secret between Pages and the cron worker |
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
| `CRON_SECRET` | No | Shared secret for the cron worker |
| `DB` (D1 binding) | Yes | Metadata |
| `BUCKET` (R2 binding) | No | R2 archive |
| `UPDATE_CHECK` | No | `off` disables the release check |
| `GITHUB_TOKEN` | No | Raises the release-check rate limit |

## Deployment-wide settings in D1

Some preferences are stored in the `app_settings` table instead of environment variables, and are edited from the UI:

- `default_signature`, `signature_mode` â€” shared signature or one per account.
- `allow_remote_images` â€” global default for remote images.
- `metadata_retention_days`, `retention_stats` â€” metadata retention window and last cleanup.
- TOTP/2FA material.
- `update_check`, `update_notice_dismissed` â€” release-check cache and dismissal.

## Local environment file

For local development, `.dev.vars` (gitignored) holds the same values:

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

## OAuth

Google Drive, Google IMAP/SMTP and Microsoft OAuth each need an OAuth application and matching redirect URIs. See [OAuth setup](oauth).

## Signatures

`signature_mode` is `shared` (default) or `per_account`:

- `shared`: `default_signature` is used for every account and `accounts.signature` is ignored.
- `per_account`: every account uses its own `accounts.signature`; an empty value means no signature.
