---
title: Known limitations
description: What WhelkMail deliberately does not do, and the trade-offs behind it.
---

# Known limitations

WhelkMail is a client, not a mail server. These are the deliberate boundaries and trade-offs.

## Synchronization and search

- No IMAP IDLE. Folder lists are served from the D1 mirror and a background sync refreshes new mail and flags. While a sync is in progress, the client polls the lightweight sync status with backoff and refreshes the list when new UIDs, backfill progress or the folder state change. Sync can also be started manually from Settings and runs every 5 minutes for accounts with automatic sync enabled (new IMAP accounts enable it by default).
- The first open of an account bootstraps the newest messages from the provider; afterwards the folder list comes from D1. Sync uses UIDNEXT and bounded backfill windows instead of scanning every UID on each run.
- Search runs against the D1 mirror across Inbox, Sent, Drafts and Spam, with an optional per-folder scope, and also matches contacts. Matching is a case-insensitive substring test (`LIKE`), not a full-text index: no stemming and no fuzzy matching.
- IMAP bodies are indexed progressively (a few per sync chunk, plus every message you open), so a search matches body text only for messages that have been indexed so far; the rest still match sender, recipients and subject.
- Lists and search results are paginated with a keyset cursor (`Load more messages` at the bottom of the list) instead of one fixed 100-message cap.

## Mailbox features

- Inbox, Sent, Drafts and Spam are available as folders. Messages can be selected in bulk, deleted, moved to Inbox/Spam/Trash, or marked as spam from the reading pane.
- Recent message bodies (up to 512 KB) are cached in D1 for up to 3 days; larger messages and attachments are re-read from the provider unless a long-term archive is configured. Inline `cid:` images are embedded in the detail response.
- A label view uses the D1 mirror when available and falls back to fetching the labeled UIDs from the provider.
- Mirrored IMAP metadata is kept indefinitely by default. Settings â†’ Appearance can limit it to a retention window: older messages are removed from the D1 mirror (never from the provider, and never drafts), together with their label assignments. Pruned messages disappear from lists, search and label views; only a direct message link still opens one. **Download metadata again** clears the sync cursors so the mirror is rebuilt from the provider, but their labels are not restored. The archive copy in R2/Drive is left untouched.
- Filters run when the inbox is loaded and when a sync ingests new inbox messages. Matching uses From/To/Subject/snippet, not the full message body.
- Forwarding and provider deletion run from the cron worker, so they require the same setup as Web Push. They only act on new incoming messages detected after the initial sync.

## Sending and addresses

- The sender is the account address: there is no free-form "send as". Replies pick the account whose address received the message, and **Reply as** in Settings â†’ Accounts overrides that choice per mailbox.
- A relay account with **Send from any address on this domain** replies from the address on its domain that the message was sent to, without an account per address.
- The alias is read from headers that aren't signed, so a forwarding header only counts when the receiving server's topmost `Authentication-Results` reports a passing DKIM signature for that domain. A server that adds none, or a signed message replayed straight to the inbox with a forwarding header added, can still mislead it. The address is always limited to the relay's exact domain and is shown in the From field before sending, and two relays allowed to send as the same domain make the alias ambiguous, so none is offered.
- Switching the IMAP body archive between R2 and Drive keeps the previously archived messages readable: each row records the backend that wrote it (legacy rows fall back to trying both).
- A relay account needs a **working sent-copy destination** (Google Drive, R2 or an IMAP mailbox). If the copy cannot be saved the send is aborted before the relay is called and returns `storage_failed`, so the message is never handed to the provider.
- Send-only accounts that keep sent copies and drafts in an IMAP mailbox (`storage_driver = 'imap'`) depend on that account: while it is disconnected, new drafts fail until you reconnect it or switch the relay account back to Drive/R2. When the deployment has Drive or R2 available, a failed APPEND falls back to it so the message (attachments and `.eml` included) is not lost; without object storage only the body stays readable from D1. The draft is also a real message in that mailbox, but its mirrored copy is filtered out of the Drafts list and of draft searches, so it is listed once, under the relay account.
- A relay send is recorded as `failed` only when the provider answered with a refusal (a 4xx, or a 2xx carrying an error body). A 5xx, a timeout or a dropped connection leaves the message as `unknown`: the provider may have accepted it before failing.

## HTML rendering

- HTML bodies are rendered in a sandboxed frame. Remote images are blocked by default and can be enabled per message. Links open in a new tab; scripts and forms are disabled.
- Small inline `cid:` images are embedded in the detail response; larger ones fall back to the authenticated attachment endpoint, which uses the D1 body cache when the message fits.

## Deployment and operations

- Single shared `APP_PASSWORD` (with optional TOTP); this is a personal / small-team tool, not a multi-tenant service.
- Login rate limiting is per IP and best-effort, not a substitute for Cloudflare Access or a WAF.
- Account deletion removes D1 rows and makes a best-effort cleanup of R2/Drive objects referenced in D1; it never deletes mail from the provider. Objects that cannot be removed (for example a revoked Drive token) are listed in Settings â†’ System â†’ "Objects left behind".
- Deleting a mailbox that stores the sent copies of a send-only account needs a second confirmation, because those copies stay on the provider but their attachments and `.eml` stop being reachable from the app.
- Provider error details are returned to the authenticated admin to help with IMAP/SMTP setup.
- New releases are announced inside the app by comparing the installed version with the latest GitHub release. The check is informational only: it cannot update a deployment. Set `UPDATE_CHECK=off` to disable it.
- Cloudflare Email Service is experimental: delivery, suppression handling, rate limits and per-recipient failures are unverified. Use SMTP2GO or another listed relay when you need a path tested in practice.
- Workers Paid is recommended because IMAP/SMTP TLS handshakes could exceed the free plan CPU budget.
- Google OAuth uses the restricted `https://mail.google.com/` scope. A public production app needs Google verification (CASA); personal/testing apps can use OAuth in Testing mode, where refresh tokens expire after 7 days. App passwords remain available as an alternative.
- Custom SMTP relays to `smtp.gmail.com` (465 and 587) and `smtp.mail.yahoo.com:465` fail with `Network connection lost.` from Workers, before authentication; other hosts (for example `smtp.fastmail.com:465`) connect normally. Use an HTTP relay, or an SMTP host that accepts the connection.
- There is no per-user isolation or audit log.

## Related

- [Mail flows](mail-flows)
- [Security](security)
- [Troubleshooting](troubleshooting)
