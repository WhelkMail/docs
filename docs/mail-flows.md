---
title: Mail flows
description: How WhelkMail lists, searches, reads, sends, replies to and threads messages.
---

# Mail flows

This page walks through the main request flows between the React client, Pages Functions, D1 and the mail provider.

## List messages

1. `GET /api/messages?accountId=all&folder=inbox`
2. The list is served from the D1 mirror first. If the folder has never been synced, the request bootstraps the newest headers from IMAP; otherwise a stale folder returns the cached list immediately and schedules a background sync.
3. The sync reads `UIDNEXT` from `SELECT` and fetches new UIDs plus bounded backfill windows instead of running `UID SEARCH ALL` on every pass. `BODYSTRUCTURE` is parsed locally to detect attachments without downloading message bodies.
4. Headers are parsed with `postal-mime`; results from all accounts are merged and sorted. Relay accounts contribute their D1 `messages` rows.
5. The merge sorts with `compareByDateDesc` (`sent_at` descending, `id` descending) — the same order as the SQL `ORDER BY sent_at DESC, id DESC` and the keyset cursor (`sent_at < ? OR (sent_at = ? AND id < ?)`). The two sides must agree: with an ascending tie-break, two messages sharing a timestamp would be returned twice and, after the client deduplicates with `mergePage`, silently disappear from the list.
6. After the response, the server prefetches the newest few bodies (one message per account in the unified view, up to three for a selected account) into the D1 body cache with `UID FETCH ... (BODY.PEEK[])` on a single connection.

## Search

1. `GET /api/messages/search?q=...` returns matching messages from the D1 mirror (Inbox, Sent, Drafts, Spam, optionally one folder) plus matching contacts.
2. Matching is a case-insensitive **substring** test (`lower(column) LIKE ?`), not a tokenized full-text index: no stemming, no fuzzy matching, and every term must appear inside the searched columns (`subject`, `snippet`, `from_email`, `from_name`, `to_json`, `cc_json`, and `search_text` or `body_text`).
3. `imap_messages.search_text` keeps up to 4.000 characters of body text, filled when a body is parsed (message opened, archived, or snippet backfill). Search therefore matches body words only for messages indexed so far; the rest still match sender, recipients and subject. Relay rows search their stored `body_text`.
4. Results are paginated with the same `(date, id)` cursor and merge order as the list endpoint.

## Read a message

1. `GET /api/messages/:id`
2. The ID encodes the account, folder and UID (`i_<account>.<folder>.<uid>`) or a relay row (`r_<uuid>`).
3. The raw MIME is served from the D1 body cache when available, then from the R2/Drive archive, then from IMAP. A body fetched from IMAP or the archive is stored in the D1 cache (up to 512 KB) to make later opens and attachment requests fast.
4. Text, HTML and attachment metadata are returned. The message is marked `\Seen` for Inbox and Spam items. Cache/archive hits update D1 immediately and mark the provider asynchronously, so the HTTP response does not wait for a second IMAP connection.
5. HTML is rendered by the client in a sandboxed iframe with a restrictive CSP; scripts and forms are disabled. Remote images are blocked by default and can be enabled per message. Small `cid:` images are embedded as data URIs in the detail response; larger ones fall back to the authenticated attachment endpoint.

## Message actions

1. `POST /api/messages/actions` with `{ ids, action: "delete" | "spam" | "move", target?: "inbox" | "spam" | "trash" }`.
2. IMAP messages are grouped by account and folder; one connection copies the UIDs to the target with `UID COPY` and removes them from the source with `UID STORE +FLAGS (\Deleted)` plus `UID EXPUNGE`.
3. Delete moves the message to the provider Trash; spam moves it to the `\Junk` folder; not-spam moves it back to Inbox.
4. The D1 mirror is updated and the source/target folders are queued for sync. Moved rows stay in D1 until the target sync transfers labels through `stable_key`, then the source sync prunes them.

## Send

1. `POST /api/messages/send`
2. The server builds a complete MIME message.
3. **IMAP account:** authenticate and send through the account's own SMTP.
4. If `auto_saves_sent` is false, the same MIME is appended to the provider's Sent folder (`\Sent`, fallback `Sent`).
5. If `draftId` is present, the original draft is deleted.
6. **Relay account:** store the `.eml` on the destination chosen in `storage_driver`, send through the selected HTTP relay, then record metadata in D1.
   - `drive` / `r2`: the object is written before the send, so a failed send still has its copy.
   - `imap`: nothing is written to object storage; the same MIME is appended to the Sent folder of the account in `sent_copy_account_id` after the relay accepts the message, `storage_key` keeps a pointer (`imap:<account>:<folder>:<uid>`) and a Sent sync is queued for that account. If the append fails the message is still sent and recorded: when the deployment has any object storage (Drive connected or the `BUCKET` binding) the MIME is written there as a rescue copy and `messages.storage_driver` records that driver, so attachments and the raw view keep working; without object storage only the D1 body remains and the response carries a warning.

## Reply

1. The reply composer takes its sender from `pickReplySender`, in this order:
   - `accounts.reply_account_id` of the mailbox that received the message (explicit override);
   - the account whose address is the alias the message was actually sent to: `aliasAddresses()` collects `Delivered-To`, `X-Delivered-To`, `X-Original-To`, `Envelope-To`, `X-Envelope-To`, `X-Forwarded-To`, `X-Forwarded-For` (Cloudflare Email Routing), `Resent-To`, `Apparently-To` and `Received … for <addr>`, in header order (topmost first), and the client also looks at the message To/Cc. Only accounts that can send are considered (relay configured, or IMAP connected). When no account has exactly that address, a relay with `relay_any_address` whose domain the alias is on is used instead (see step 2). Both kinds of match go through the same checks in `pickReplyAccount()` (`shared/from-address.ts`): a forwarding header only counts when the receiving server vouches for its domain, the first such header that names an account wins, and To/Cc only count when they name exactly one such address;
   - the global "reply sender" preference (`auto` by default);
   - the account that received the message.

   The alias step is what makes a forwarding domain work like Gmail: mail addressed to `EMAIL@DOMAIN.IT` and forwarded to `EMAIL@PROVIDER.IT` keeps that address in the headers, so the reply leaves from the domain account.

2. If the chosen account is a relay with **Send from any address on this domain** on (`accounts.relay_any_address`), the reply leaves from the address the message was sent to, as long as it is on the account's exact domain (`pickDomainAlias()` in `shared/from-address.ts`):
   - the first address on the domain in `aliasAddresses()`, read top-down, but only when the receiving server's topmost `Authentication-Results` reports a passing DKIM signature for that exact domain (`verifiedDomains()`). Cloudflare Email Routing signs forwarded mail with the domain; mail delivered straight to the mailbox can't carry such a signature, so a forged `X-Original-To` on it is ignored. On forwarded mail, every server prepends its headers, so the forwarder's own record sits above anything the original sender wrote;
   - otherwise To/Cc, only when exactly one address there is on the domain;
   - otherwise the account's own address.

   With several such relays, the trusted forwarding headers are searched across all of them before any To/Cc, and To/Cc only count when they name exactly one address across all relay domains and accounts (`pickReplyAccount()`). The receiving mailbox's own address, which often heads the headers on a shared domain, is skipped while looking for the alias; mail addressed only to it stays with that account. Replying from Sent keeps the address the message was sent from, since To/Cc there are the recipients, and a relay's own Sent copy is answered by that relay. Only addresses the send endpoint accepts are offered: `shop=orders@DOMAIN.IT` is read whole, then not used, rather than cut down to `orders@DOMAIN.IT`. Two relays allowed to send as the **same** domain make the alias ambiguous: WhelkMail then offers no alias at all, and the reply leaves from the account's own address.

   The composer shows the address as its own entry in the From list, so it can be switched back before sending. `POST /api/messages/send` takes it as `fromEmail` and refuses (`from_address_not_allowed`) anything that isn't the account's own address or, with the setting on, another address on its exact domain (no subdomains). The From in the MIME, the relay call and the stored Sent copy all use it. A draft keeps only the account, so a reopened draft sends from the account's own address.

3. Everything else is the normal send flow, so the reply leaves through the chosen account's own SMTP/relay and its copy follows that account's destination. Addresses of both accounts are excluded from To/Cc to avoid replying to yourself.

## Conversations

1. `GET /api/threads?accountId=&folder=` returns one row per conversation. The group key is `COALESCE(thread_key, stable_key)`, so messages mirrored before threading existed stay visible until the backfill reaches them.
2. Pagination is a keyset cursor on `(last message date, thread key)`: the scoped query decides which conversations are on the page (`HAVING last_at < ? OR (last_at = ? AND key < ?)`), then **every message of those conversations is loaded from all folders of the account** (drafts excluded) to build the row: newest message, unread count, labels and the distinct senders, so the list reads like Gmail (`Mario, Luca +2  5`) and an answered thread counts your reply even when it lives in Sent.
3. Threads are per account: the same subject on two mailboxes stays two conversations. The unified view lists both, never merges them.
4. Ordering and the cursor use the same pair as the SQL, `(last_at, thread key)`, so conversations sharing a timestamp cannot repeat or vanish between pages.
5. `GET /api/threads/messages?accountId=&threadKey=` matches `COALESCE(thread_key, stable_key)` (`'r:' || id` for relay rows), the same keys the list used, and returns the whole conversation (Inbox, Sent and Spam of that account, drafts excluded), oldest first, so your replies appear inside the thread. The reading pane stacks one header row per message with the opened message expanded below it; clicking another row loads that message in place.
6. The endpoint does **no IMAP work at all**: it reports `syncing`, `staleAccounts` and `activeAccounts`. A `useEffect` running once the list is on screen calls `pickSyncRequests()` (`src/lib/sync-queue.ts`, keys are `accountId:folder`, so switching folder does not hit the cooldown of the previous one): at most three folders, never a locked one, never one asked in the last 3 seconds. Each selected folder is then driven by `runSyncChain()` (`src/lib/sync-chain.ts`), which keeps calling `POST /api/accounts/:id/sync` while the answer says `hasMore`, stops on error, stops when another worker holds the lock (`locked`) and gives up after a cap of 30 chunks or after 3 chunks in a row that do not advance the chunk `progress` — mirrored rows **plus archived bodies** — so a healthy archive run is not mistaken for a stall. `partial` means "more to fetch", never "already fetching". Accounts are queried in parallel. The messages of the visible conversations are loaded with a **per conversation** cap (`ROW_NUMBER() OVER (PARTITION BY key …) <= 40`), so a busy thread cannot push another one out of the page, while an exact `COUNT(*)`/`SUM(unread)` per key keeps the totals right even when only part of the history is loaded. When a chain ends without `done` (cap reached, or a lock held by another worker) the client schedules another chain after a short pause (`nextChainDelay`: 500 ms, 5 s for a lock), keeping one chain per folder, so a very large mailbox keeps going to `done`. The chain lives in the browser: closing the tab leaves the folder partial, and it resumes on the next load (or with the cron worker).
7. `thread_key` is the Message-ID of the root, computed by `threadKeyFor()` (`References[0]` → `In-Reply-To` → own Message-ID). Replies written in the app send `In-Reply-To`/`References`, so their copies thread in any client. A bounded sync phase re-reads only the headers of rows still missing a key.
8. Label views and search results are grouped as well: a label view loads the whole conversation for the conversations that carry the label (`GET /api/threads?labelId=`), and search collapses the matching messages of a thread into one row whose count refers to the matches on that page. Deleting a thread row acts on the newest message shown.
9. When a message has no `References`/`In-Reply-To` (broken chains, forwarded copies), `normalizeSubject()` strips `Re:`/`R:`/`AW:`/`SV:`/`Fwd:` and the conversation already stored for that subject is adopted — the same fallback Gmail uses. Only messages whose subject carries a reply prefix take part, so two unrelated mails sharing a subject keep their own threads; the parent lookup also covers the `messages` table, so a reply to something sent from the app joins the right conversation.
10. Settings → System → **Conversations** shows how many mirrored messages still have no key per account and rebuilds them in bulk (`GET/POST /api/threads/rebuild`, headers only, 200 messages per call).

## Drafts

- **IMAP:** `APPEND` to the `\Drafts` folder with `\Draft \Seen`; delete with `UID STORE +FLAGS (\Deleted)` + `EXPUNGE`.
- **Relay, `storage_driver = 'drive' | 'r2'`:** the MIME is stored on the object archive (`drafts/<id>.eml`, or `accounts/<id>/drafts/<id>.eml` for R2), metadata in D1 with `folder='draft'`.
- **Relay, `storage_driver = 'imap'`:** the draft is appended to the Drafts folder of the account in `sent_copy_account_id`, exactly like a draft written from that account.
  - The folder is resolved against the **target** account: its `drafts_folder` first, then the `\Drafts` special-use flag, then known names (`Drafts`, `INBOX.Drafts`, `INBOX/Drafts`, `Bozze`, `INBOX.Bozze`, `Bozza`, `INBOX.Bozza`, `Draft`, `INBOX.Draft`). A missing folder is created with `CREATE` and the append retried.
  - Flags are `(\Draft \Seen)`. The UID comes from `APPENDUID` when the server supports UIDPLUS, otherwise from the folder's `UIDNEXT - 1` or from the highest UID after a search.
  - The row in `messages` keeps `folder='draft'`, `storage_driver='imap'` and a pointer in `storage_key`. Reopening the draft fetches that message back with `UID FETCH`; autosave appends the new version and expunges the previous copy; deleting or sending removes the copy from that mailbox.
  - There is no automatic archive on Drive/R2 in this mode. If the target account is missing or not connected, saving fails with `sent_copy_target_not_connected`. When the APPEND itself fails and object storage exists, the draft is rescued there instead of being lost; with no object storage the save fails with `draft_save_failed`.
  - The mailbox copy is mirrored as an ordinary draft of the target account and filtered out of the Drafts list and draft searches, so the draft is listed once, under the relay account that owns it.
- Autosave runs 5 seconds after the last change and every 30 seconds while changes remain. Closing the composer saves pending changes first; if the draft cannot be saved the user is asked whether to discard it.
- An explicit **Save draft** closes the composer and opens the Drafts folder.

## Where a send-only account keeps its copies

`storage_driver` picks **one** destination per relay account; the two mailbox modes never run together with the object archive:

| `storage_driver` | Sent copies | Drafts | Requirement |
| --- | --- | --- | --- |
| `drive` | `sent/<id>.eml` on Google Drive | `drafts/<id>.eml` on Google Drive | Google Drive connected |
| `r2` | `accounts/<id>/sent/<id>.eml` in R2 | `accounts/<id>/drafts/<id>.eml` in R2 | `BUCKET` binding |
| `imap` | `APPEND` to the target account's Sent folder | `APPEND` to the target account's Drafts folder | `sent_copy_account_id` pointing at a connected IMAP account |

Switching back to `drive` or `r2` clears `sent_copy_account_id`. Copies appended to a mailbox are real messages on the provider: deleting the account or a row in D1 never deletes them. Each message keeps the destination it was written with, so changing destination does not orphan the older copies.

## Related

- [Data model](data-model)
- [Relay accounts](relay-accounts)
- [Known limitations](limitations)
