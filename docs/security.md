---
title: Security
description: WhelkMail's authentication, session handling, encryption, security headers and threat model.
---

# Security model

WhelkMail is intended for personal or small trusted deployments. This page describes what it protects and what it deliberately does not.

## Authentication

- **Single shared password + optional 2FA:** `APP_PASSWORD` protects the whole deployment. Two-factor authentication (TOTP, RFC 6238) can be enabled in Settings → Security; it is **off by default** and works with any authenticator app. Recovery codes are shown once and stored only as SHA-256 hashes.
- **Sessions:** random 256-bit session IDs; only their SHA-256 hashes are stored in D1 (`sessions`). TTL is 7 days. Cookies are `HttpOnly`, `SameSite=Lax`, and `Secure` on HTTPS. Logout revokes the session. Deleting the `sessions` table rows invalidates all sessions.
- **Brute-force protection:** login and 2FA attempts are rate limited per IP in D1 (10 attempts per 15 minutes, then a 15-minute block).
- **Origin check:** mutating requests with a foreign `Origin` header are rejected.

## OAuth

- One-time state stored in D1 with a 10-minute TTL.
- PKCE (`S256`) for Google, Microsoft and Google Drive.

## Credential encryption

- IMAP/SMTP passwords, OAuth refresh tokens and relay API keys are encrypted with AES-256-GCM using `TOKEN_ENC_KEY` (32 raw bytes, base64 encoded).
- Session/challenge signatures use an HKDF-derived subkey.
- **Key rotation:** changing `TOKEN_ENC_KEY` makes all stored credentials unreadable and invalidates sessions and pending OAuth states. Users must re-enter passwords and reconnect OAuth accounts.

## Response headers

CSP, HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy` and `Cross-Origin-Opener-Policy` are set on static assets (`public/_headers`) and API responses (middleware).

## Email rendering

- HTML bodies are rendered in a sandboxed iframe with a restrictive CSP.
- Scripts and forms are blocked.
- Remote images and tracking pixels are blocked by default and can be enabled per message (or globally in Settings).
- Small `cid:` inline images are embedded as data URIs; larger ones use the authenticated attachment endpoint.
- Links open in a new tab.

## Data protection

- By default, messages, sent copies, drafts and attachments stay on the mail provider.
- The D1/SQLite body cache is short-lived and bounded (512 KB per body, 3 days, 64 MB total).
- Long-term archiving on R2/S3 or Google Drive is strictly opt-in per account.
- Deleting an account never deletes mail from the provider.

## Secrets hygiene

- `.dev.vars`, `.env` and `.wrangler` are gitignored.
- Production secrets belong in the target's secret store: Workers/Pages project settings, or `deploy/docker/.env` on Docker.
- The public repository must never contain `wrangler.toml`, `worker/wrangler.toml`, `.dev.vars`, `.env*` or any secret. The tracked `wrangler.jsonc` keeps a placeholder D1 id; do not commit your real id.
- On Docker, protect the data volume (`/data`) and the `.env` file like any other credential store.

## No multi-tenancy

There is no per-user isolation. This is intended for personal or small trusted deployments. Put Cloudflare Access (or the authentication offered by your reverse proxy on Docker) in front for stronger authentication. Login and 2FA rate limiting are present but per-IP only, and are not a substitute for Cloudflare Access or a WAF.

## Account authentication methods

| Provider | Method |
|---|---|
| Google accounts | OAuth2 (recommended) or app password (2-Step Verification required) |
| Outlook / Microsoft 365 | OAuth2 + XOAUTH2 (basic auth is no longer supported by Microsoft) |
| Yahoo | App password |
| iCloud | App-specific password |
| Generic IMAP | Account password or app password |
| Relay accounts | Provider HTTP API key (+ secret/domain where required) |

## Recovering from lost 2FA

If you lose both the authenticator and the recovery codes, disable 2FA from a D1 console (on Docker, run the same SQL against `/data/whelkmail.sqlite` with the `sqlite3` CLI or a Node script):

```sql
DELETE FROM app_settings WHERE key IN ('totp_enabled', 'totp_secret_enc', 'totp_recovery_json');
DELETE FROM sessions;
```

## Known security limitations

- The reply alias comes from headers that aren't signed. Forwarding headers only count when the receiving server's topmost `Authentication-Results` reports a passing DKIM signature for the domain, which stops a forged header on mail delivered straight to the inbox. A receiving server that adds no `Authentication-Results` of its own, or a genuinely signed message replayed with a forwarding header, can still mislead it. The address is always limited to the relay's exact domain and is shown in the From field before sending.
- Provider error details are returned to the authenticated admin to help with IMAP/SMTP setup. Generic API errors carry a stable `code`; only authenticated administrators see provider details.
- There is no audit log.

## Hardening checklist

- Use a long random `APP_PASSWORD`.
- Enable TOTP two-factor authentication.
- Keep `TOKEN_ENC_KEY` secret and backed up.
- Serve the app over HTTPS only.
- Put Cloudflare Access in front for anything beyond a personal deployment.
- Rotate secrets according to the procedure in [Deployment](deployment#secret-rotation).

## Related

- [Configuration](configuration)
- [OAuth setup](oauth)
- [Known limitations](limitations)
