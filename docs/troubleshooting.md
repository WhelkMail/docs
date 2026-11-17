---
title: Troubleshooting
description: Common WhelkMail symptoms, likely causes and fixes for accounts, deployment and storage.
---

# Troubleshooting

## Accounts and authentication

| Symptom | Likely cause / fix |
|---|---|
| Google `AUTHENTICATIONFAILED` | Use OAuth or an app password with 2FA enabled, not the account password |
| Google OAuth stops working after ~7 days | Consent screen still in Testing; use an Internal app or complete verification, or fall back to an app password |
| Google "access blocked" | Redirect URI mismatch, missing test user, or restricted-scope verification required |
| Outlook login fails | Missing OAuth permissions, redirect URI mismatch, or admin consent |
| Connection timeout | Wrong host/port/security: 993 TLS, 465 TLS, 587 STARTTLS, 143 STARTTLS |
| Sent copy not saved | Provider lacks `\Sent`/`APPEND`; set the folder manually or adjust `auto_saves_sent` |
| Duplicate sent mail | `auto_saves_sent` is false but the provider already saves sent mail |
| Drafts not saved | No `\Drafts` folder or no `APPEND` permission |
| Attachment rejected | Provider-specific message size limit |

## Deployment

| Symptom | Cause / fix |
|---|---|
| `database_id` placeholder error | Replace the placeholder in the local `wrangler.toml`, or set the `DB` binding in the Pages dashboard |
| `D1_ERROR: no such table: accounts` | Apply the schema and record the baseline: `npm run db:init:remote` |
| `D1_ERROR: no such table: labels` / `message_labels` / `app_settings` / `filters` / `contacts` / `imap_sync` / `imap_messages` / `push_subscriptions` / `push_prefs` / `forward_log` / `deletion_log` / `sessions` / `login_attempts` / `oauth_states` / `imap_body_cache` / `imap_sync_locks` / `schema_migrations` | Run `npm run db:migrate:remote`: migrations already recorded are skipped |
| `D1_ERROR: no such column: archive_imap_bodies` / `archive_key` / `imap_archive_driver` / `auto_sync` / `delete_after_archive` / `forward_to_json` / `drafts_folder` / `spam_folder` | Run `npm run db:migrate:remote`, or apply the missing migrations in order |
| 401 loop after login | `APP_PASSWORD` mismatch, cookie blocked, site not served over HTTPS, or the `sessions` table missing (run `node scripts/migrate.mjs remote --from=012`) |
| 500 on every API call | Missing `DB` binding, missing secrets, or missing security tables |
| Google OAuth 503 | `GOOGLE_MAIL_CLIENT_ID` / `GOOGLE_MAIL_CLIENT_SECRET` missing |
| Microsoft OAuth 503 | `MICROSOFT_CLIENT_ID` / `MICROSOFT_CLIENT_SECRET` missing |
| OAuth redirect error | Redirect URI does not exactly match the app configuration |
| `CPU time limit exceeded` | Upgrade to Workers Paid |
| Custom domain error 1014/522 | DNS record missing or origin not ready |

## Storage and relay

| Symptom | Cause / fix |
|---|---|
| Drive 403 | OAuth scopes, testing-mode token expiry, or Drive not connected |
| R2 option hidden | Missing `BUCKET` binding |
| Relay 503 | Relay provider not configured for the account and no deployment fallback |
| Sending from a relay fails with `storage_failed` | No working sent-copy destination: connect Google Drive, create the `BUCKET` binding, or pick an IMAP mailbox for Sent/Drafts. The send is aborted before the relay is called |
| `too_many_recipients` when sending | The relay caps recipients per message (to + cc + bcc combined); Cloudflare Email Service allows 50 |
| Custom SMTP relay to `smtp.gmail.com` (465/587) or `smtp.mail.yahoo.com:465` fails with `Network connection lost.` | Those hosts drop the connection from Workers before AUTH. Use an app password on a host that accepts the connection, or an HTTP relay |

## Upgrades

| Symptom | Cause / fix |
|---|---|
| App returns a schema error after deploying | The code is newer than the database. Run `npm run db:migrate:remote`, then retry |
| Release check says unreachable | GitHub API and releases feed both unreachable from the Worker. The app keeps working with the cached answer; run **Check again** |
| No update notice on a fork | Only releases newer than your build are announced |

## Where to look next

- [Deployment](deployment)
- [Configuration](configuration)
- [OAuth setup](oauth)
- [Known limitations](limitations)
