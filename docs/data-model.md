---
title: Data model
description: The D1 tables WhelkMail uses, what they store and how synchronization, labels and archives work.
---

# Data model

WhelkMail stores accounts, settings, mirrored IMAP metadata, relay messages, labels, filters and notification state in Cloudflare D1.

## `accounts`

One row per connected identity.

Important fields:

- `kind`: `imap` or `relay`
- `imap_host`, `imap_port`, `imap_security`, `smtp_host`, `smtp_port`, `smtp_security`
- `username`
- `auth_type`: `password` or `oauth2`
- `secret_enc`: encrypted password or OAuth refresh token
- `access_token_enc`, `access_token_expires_at`
- `oauth_provider`: currently `microsoft`
- `sent_folder`, `drafts_folder`, `spam_folder`, `auto_saves_sent`
- `sent_copy_account_id`: IMAP account that stores sent copies and drafts when `storage_driver` is `imap` (relay accounts only)
- `reply_account_id`: account that sends the replies to messages received here. Used when a domain only forwards to this mailbox and sends through a relay account; cleared when that account is deleted
- `relay_any_address`: relay accounts only. When set, replies may leave from any address on the account's exact domain. Off by default
- `signature`: optional per-account signature, used only when `signature_mode` is `per_account`
- `incoming_label_id`: optional shared label applied automatically to every incoming message of this account
- `delete_after_archive`: opt-in provider cleanup; archived incoming messages are moved to the provider Trash
- `auto_sync`: opt-in automatic background sync for Inbox, Sent, Drafts and Spam via the cron worker
- `storage_driver`: `provider`, `drive`, `r2` or `imap` (relay accounts only)
- `relay_provider`, `relay_api_key_enc`, `relay_api_secret_enc`, `relay_domain`, `relay_region`
- `drive_*` fields for the optional Drive archive

## `messages`

Used for relay accounts: sent copies and drafts. IMAP messages are read from the provider and mirrored in `imap_messages`; the optional sync keeps metadata current, while recent bodies are held in the short-lived D1 cache. Full bodies are copied to R2/Drive only when long-term archiving is enabled.

When a relay account stores copies in an IMAP mailbox, `storage_key` holds a pointer (`imap:<accountId>:<encodedFolder>:<uid>`) instead of an object key: the body and attachments are read from that mailbox, and the copy is removed there when the message or draft is deleted. Account deletion never touches those messages.

Each row keeps the driver that actually wrote its copy: `storageForDriver(env, account, row.storage_driver)` is used for reading and for deleting objects, while `getStorage(env, account)` is only used to write new ones. Changing the destination of an account (Drive â†’ R2 â†’ mailbox) therefore leaves the historical copies readable and deletable.

Fields include folder, addresses JSON, subject, snippet, body text/HTML, size, storage key, attachment metadata, conversation key and status.

## `labels` and `message_labels`

`labels` stores the custom label definitions (name, color). Labels are shared across every account, which keeps one unified taxonomy. `message_labels` stores assignments keyed by the app message id (`i_<account>.<folder>.<uid>` for IMAP, `r_<uuid>` for relay messages).

IMAP assignments survive page reloads but refer to IMAP UIDs; if a mailbox changes UIDVALIDITY, reassign labels on affected messages. Deleting a label removes the related assignments and filter references.

## `filters`

Gmail-style automatic filters, shared across all accounts. Each row stores criteria (`from_pattern`, `to_pattern`, `subject_pattern`, `has_words`, `doesnt_have`, `has_attachment`) and actions: label ids in `label_ids_json` and forwarding recipients in `forward_to_json`. The legacy `account_id` column is kept only for schema compatibility and is set to an empty string for new rows.

Enabled filters are evaluated when the inbox is listed and when the cron worker syncs new inbox messages; matching messages receive the labels in `message_labels` and are forwarded when `forward_to_json` is not empty.

## `contacts`

Shared address book: `id`, `name`, `email`, `note`, `created_at`. A unique index on `lower(email)` prevents duplicates; creating a contact with an existing email updates the name/note instead. Contacts are not account-scoped. The UI supports search, create/edit/delete, CSV import/export, and autocomplete in To/Cc/Bcc.

## `imap_sync` and `imap_messages`

`imap_messages` mirrors IMAP message metadata (headers, flags, size, attachment flag, conversation key) in D1. `imap_sync` stores the per-account/per-folder cursor: `uid_validity`, `max_uid`, `min_uid`, `backfill_done`, status, error, counts and last sync time.

The sync is resumable and chunked (40 metadata messages per request, 8 when body archiving is enabled). It reads `UIDNEXT` from `SELECT`, fetches new UIDs above `max_uid`, and backfills older messages through bounded UID windows instead of running `UID SEARCH ALL` on every pass. Each run also refreshes flags for the most recent window, so read/unread changes made by other clients land in D1.

A folder synced in `partial` state keeps receiving backfill chunks after a short retry interval, independently of the 60-second refresh of new mail. A row in `imap_sync_locks` (with a 5-minute expiry and an owner token) prevents two Workers isolates from running the same folder at the same time; the owner renews the expiry during long phases and only the owner can release it. Expired locks are removed by the cron worker. If the lock table or owner column is missing the sync still runs unlocked.

A changed `UIDVALIDITY` resets the folder mirror. `stable_key` is the RFC `Message-ID` (or a deterministic hash fallback) and is used to transfer labels when a message moves between folders. Deleted/expunged messages are pruned after backfill when the provider message count differs from D1; archived objects are removed unless provider cleanup is enabled, in which case the archive is preserved.

The folder list reads from `imap_messages` first. Opening a folder does not block on IMAP: if the mirror is stale, the request returns the cached list immediately and schedules a background sync; the frontend polls the lightweight sync state with backoff and refreshes the list only when new UIDs or folder state change. On the first open of an account (empty mirror) the request runs one bounded sync chunk and re-reads the mirror. Sent, Drafts and Spam folder names are resolved once and stored in `accounts.sent_folder` / `accounts.drafts_folder` / `accounts.spam_folder`.

### Body cache

`imap_body_cache` is a bounded, best-effort cache: message bodies up to 512 KB are kept for 3 days, with a hard 64 MB cap. The cron worker deletes expired rows and evicts the oldest bodies until the cache is back under the cap; opportunistic cleanup also runs every 50 writes. It never replaces the provider or the long-term archive; it only avoids repeated IMAP fetches for recently listed/opened messages.

### Automatic sync

`accounts.auto_sync` enables the cron worker to advance Inbox, Sent, Drafts and Spam every 5 minutes. Each run processes one bounded chunk per folder and resumes on the next run. New IMAP accounts enable it by default; existing accounts can still rely on the sync scheduled when a folder is opened.

### Metadata retention

`app_settings.metadata_retention_days` is `0` by default (keep everything) and is edited from Settings â†’ Appearance. When set, the cron worker deletes `imap_messages` rows older than the window (never drafts, never anything on the provider) together with their `message_labels` rows, in batches of 500 up to 5.000 per run.

Pruned messages no longer appear in folder lists, search results or label views, and the incremental sync does not pull them back because it tracks `max_uid`/`min_uid` instead of re-scanning UIDs. Only a direct message id still opens one, because reading a message falls back to IMAP when the mirror has no row. The R2/Drive archive copy is untouched, and lowering then raising the window does not restore the removed rows.

Each pruning run is recorded in `app_settings.retention_stats` (`lastRunAt`, `lastDeleted`, `deletedTotal`, `cutoff`), read by `GET /api/retention` and shown in Settings â†’ Appearance. `POST /api/retention` with an optional `{ accountId }` clears the `imap_sync` cursors of the IMAP accounts and queues a sync, so the next runs walk the mailbox again and recreate the pruned rows. Label assignments are not recovered, and a full rescan costs one metadata chunk per folder per sync run.

### Body archive

`accounts.archive_imap_bodies` is optional; `accounts.imap_archive_driver` selects `r2` or `drive` for **new** archives. Each archived row records `imap_messages.archive_driver` next to `archive_key`, and reading or pruning an archived body asks that backend first and then the other configured one, so switching the archive between R2 and Drive keeps earlier bodies readable. Rows archived before the column existed keep `archive_driver = NULL` and fall back to both backends.

R2 requires a bound `BUCKET`; Drive requires the account's Google Drive OAuth connection. The sync stores the raw EML under `imap/<account>/<folder>/<uidvalidity>/<folder>-<uid>.eml` and records `archived`/`archive_key`/`archive_driver` in `imap_messages`. With Drive, `archive_key` is the Drive file id. Reading and attachment/download endpoints use the D1 body cache first, then the archive, and fall back to IMAP otherwise. When disabled, bodies and attachments stay on the provider.

## `push_subscriptions` and `push_prefs`

`push_subscriptions` stores Web Push endpoints and their `p256dh`/`auth` keys. Subscriptions are installation-wide, not account-scoped. `push_prefs` stores per-account notification preferences: `enabled` and `folders_json` (default `["inbox"]`). The cron endpoint reads these tables, syncs the selected folders and sends notifications for messages above the previous `max_uid`.

## `forward_log` and `deletion_log`

`forward_log` records the last forwarding result per `(filter_id, message_id)` and prevents duplicate forwards. `deletion_log` records provider-cleanup results per message.

Forwarding is an action of a filter. The cron endpoint evaluates filters for new incoming messages, applies their labels and forwards matching messages to the filter recipients. The original EML is attached, and the forwarded message carries `X-WhelkMail-Forwarded` to prevent loops.

Provider cleanup only runs when `archive_imap_bodies` and `delete_after_archive` are both enabled and the message is archived. The message is copied to the provider Trash and then removed from the source folder; the archived copy is preserved.

## `app_settings`

Key/value store for deployment-wide preferences: signatures (`default_signature`, `signature_mode`), remote images (`allow_remote_images`), retention (`metadata_retention_days`, `retention_stats`), 2FA material, and the release check (`update_check` cache, `update_notice_dismissed`).

`signature_mode` is `shared` (default) or `per_account`:

- `shared`: `default_signature` is used for every account and `accounts.signature` is ignored.
- `per_account`: every account uses its own `accounts.signature`; an empty value means no signature.

## Security tables

- `sessions` â€” SHA-256 hashes of server-side session IDs.
- `login_attempts` â€” rate limiting per IP.
- `oauth_states` â€” one-time OAuth state with a 10-minute TTL.
- `imap_sync_locks` â€” cross-isolate sync lock.
- `schema_migrations` â€” applied migration registry.
- `storage_cleanup_log` â€” objects that could not be removed during account deletion.

## Related

- [Mail flows](mail-flows)
- [Security](security)
- [Development](development)
