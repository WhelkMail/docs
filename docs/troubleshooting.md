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
| `database_id` placeholder error | Replace the placeholder in `wrangler.jsonc` for local CLI use, or set the `DB` binding in the dashboard. Docker uses SQLite and does not read this value |
| `D1_ERROR: no such table: accounts` | Apply the schema and record the baseline: `npm run db:init:remote` |
| `no such table: accounts` on Docker | The container could not write `/data`; check the volume permissions and `DB_PATH`, then restart |
| `D1_ERROR: no such table: labels` / `message_labels` / `app_settings` / `filters` / `contacts` / `auto_reply_log` / `imap_sync` / `imap_messages` / `push_subscriptions` / `push_prefs` / `forward_log` / `deletion_log` / `sessions` / `login_attempts` / `oauth_states` / `imap_body_cache` / `imap_sync_locks` / `schema_migrations` | Run `npm run db:migrate:remote`: migrations already recorded are skipped |
| `D1_ERROR: no such column: archive_imap_bodies` / `archive_key` / `imap_archive_driver` / `auto_sync` / `delete_after_archive` / `forward_to_json` / `drafts_folder` / `spam_folder` / `source` / `signature_html` / `auto_reply_*` / `first_name` / `last_name` / `emails_json` / `phones_json` | Run `npm run db:migrate:remote`, or apply the missing migrations in order (031-035 for 1.3.0 and newer) |
| 401 loop after login | `APP_PASSWORD` mismatch, cookie blocked, site not served over HTTPS, or the `sessions` table missing (run `node scripts/migrate.mjs remote --from=012`). On Docker, set `PUBLIC_ORIGIN` to the public HTTPS URL |
| 500 on every API call | Missing `DB` binding, missing secrets, or missing security tables. On Docker, check `APP_PASSWORD`/`TOKEN_ENC_KEY` and the `/data` volume |
| OAuth redirect points to localhost on Docker | `PUBLIC_ORIGIN` is unset while the proxy terminates TLS; set it to the public URL |
| Google OAuth 503 | `GOOGLE_MAIL_CLIENT_ID` / `GOOGLE_MAIL_CLIENT_SECRET` missing |
| Microsoft OAuth 503 | `MICROSOFT_CLIENT_ID` / `MICROSOFT_CLIENT_SECRET` missing |
| OAuth redirect error | Redirect URI does not exactly match the app configuration |
| `CPU time limit exceeded` | Upgrade to Workers Paid, or use the Docker target which has no Workers CPU limit |
| Custom domain error 1014/522 | DNS record missing or origin not ready |

## Storage and relay

| Symptom | Cause / fix |
|---|---|
| Drive 403 | OAuth scopes, testing-mode token expiry, or Drive not connected |
| R2/S3 option hidden | Missing `BUCKET` binding (Cloudflare) or `S3_*` variables (Docker) |
| Relay 503 | Relay provider not configured for the account and no deployment fallback |
| Sending from a relay fails with `storage_failed` | No working sent-copy destination: connect Google Drive, create the `BUCKET` binding, or pick an IMAP mailbox for Sent/Drafts. The send is aborted before the relay is called |
| `too_many_recipients` when sending | The relay caps recipients per message (to + cc + bcc combined); Cloudflare Email Service allows 50 |
| Custom SMTP relay to `smtp.gmail.com` (465/587) or `smtp.mail.yahoo.com:465` fails with `Network connection lost.` | Those hosts drop the connection from Workers before AUTH. Use an app password on a host that accepts the connection, or an HTTP relay |

## Mail, contacts and automation

| Symptom | Cause / fix |
|---|---|
| New mail has no label after a filter | Check that the filter is enabled and, for word rules, that the body has been indexed. The list endpoint re-applies filters on load; reload the folder. If the filter targets a forwarding alias, apply migration 031 and let a sync re-read the headers |
| Filter matches old mail but not new mail | The filter likely uses `To` with a forwarding domain. Confirm the forwarding headers (`Delivered-To`, `X-Forwarded-For`, `Received ... for`) are present and run a sync; see [Labels and filters](labels-filters) |
| Other contacts badge never appears | Migration 032 (`contacts.source`) is missing. Contacts still save; run `npm run db:migrate:remote` to enable the badge |
| Sending does not add the recipient to contacts | Settings -> General has "I add contacts myself", or the address belongs to one of the configured accounts |
| Typing an address does not show suggestions | Suggestions start after the first character and use prefix matching; `m` does not match `indirizzo@gmail.com` |
| Enter in a recipient sends the message | Update to 1.3.0: Enter commits the address or picks the highlighted suggestion; only a click on **Send** sends |
| vCard import fails | The card needs a valid email and migration 035 must be applied. vCard 3.0/4.0 are detected automatically; uncommon encodings are not decoded |
| vCard export is 3.0 by default | Expected. Pick vCard 4.0 in the export format selector when needed |
| Rich-text toolbar does nothing or bolds everything | Update to 1.3.0; the toolbar now runs on pointerdown and restores the selection. If an existing signature is already bold, select it and use **Clear formatting** |
| Auto-reply does not send | Enable it in Settings -> Auto-reply, check the mode (shared/per account), and make sure the notification cron runs or a client syncs new inbox mail. Check `auto_reply_log.status` for `error`; old mail (>7 days) and the first sync are ignored |
| Auto-reply replies twice | The unique `(account_id, message_id)` index should prevent it; if the table is missing, apply migration 033. A second response can also come from another mailbox/account |

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
