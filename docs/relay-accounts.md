---
title: Relay accounts
description: Send-only accounts for domains without SMTP, and where their sent copies and drafts are stored.
---

# Relay accounts and storage

Relay accounts are for send-only domains. They send through an HTTP email API instead of SMTP and keep sent copies and drafts on Google Drive, Cloudflare R2 or in the Sent/Drafts folders of an IMAP account you already read.

## Supported providers

`smtp2go`, `resend`, `mailgun`, `brevo`, `sendgrid`, `mailjet`, `postmark`, and the experimental `cloudflare` Email Service.

- Credentials can be entered per account in the UI and are encrypted.
- Deployment-level `RELAY_*` variables act as a fallback.
- The sender domain must be verified with the provider (SPF/DKIM).
- Conservative attachment limits are enforced per provider: 15 MB for most, 12 MB for Mailjet, 8 MB for Brevo.

## Storage destinations

`storage_driver` picks **one** destination per relay account; the mailbox mode never runs together with the object archive:

| `storage_driver` | Sent copies | Drafts | Requirement |
| --- | --- | --- | --- |
| `drive` | `sent/<id>.eml` on Google Drive | `drafts/<id>.eml` on Google Drive | Google Drive connected |
| `r2` | `accounts/<id>/sent/<id>.eml` in R2 | `accounts/<id>/drafts/<id>.eml` in R2 | `BUCKET` binding |
| `imap` | `APPEND` to the target account's Sent folder | `APPEND` to the target account's Drafts folder | `sent_copy_account_id` pointing at a connected IMAP account |

Switching back to `drive` or `r2` clears `sent_copy_account_id`.

> A relay account always needs a **working** destination. If the sent copy cannot be saved (Drive not connected, no `BUCKET` binding, or the chosen IMAP mailbox disconnected) the send is aborted **before** the relay is called and comes back as `storage_failed`.

Copies appended to a mailbox are real messages on the provider: deleting the account or a row in D1 never deletes them. Each message keeps the destination it was written with, so changing destination does not orphan the older copies.

## Send flow

1. The server builds a complete MIME message.
2. It stores the `.eml` on the destination chosen in `storage_driver`.
   - `drive` / `r2`: the object is written before the send, so a failed send still has its copy.
   - `imap`: nothing is written to object storage; the MIME is appended to the Sent folder after the relay accepts the message, and the row keeps a pointer in `storage_key`.
3. It sends through the selected HTTP relay.
4. It records metadata in D1.

If an IMAP append fails after the relay accepted the message, the message is still sent and recorded. When the deployment has any object storage (Drive connected or the `BUCKET` binding) the MIME is written there as a rescue copy and `messages.storage_driver` records that driver. Without object storage only the D1 body remains and the response carries a warning.

A failed send is recorded as `failed` only when the provider answered with a refusal (a 4xx, or a 2xx carrying an error body). A 5xx, a timeout or a dropped connection leaves the message as `unknown`: the provider may have accepted it before failing, so the UI asks to check its activity log instead of inviting a resend.

## Replies from a relay domain

A relay with **Send from any address on this domain** (`accounts.relay_any_address`) can answer as the address the message was actually sent to, as long as it is on the account's exact domain. This makes a forwarding domain work with one account instead of one per address.

The alias is chosen from trusted forwarding headers first, then To/Cc, and only when the receiving server's topmost `Authentication-Results` vouches for the domain. The address is shown in the From field before sending and can be switched back. See [Mail flows → Reply](mail-flows#reply) for the full rules.

## Drafts

- **Drive/R2:** the MIME is stored on the object archive (`drafts/<id>.eml`, or `accounts/<id>/drafts/<id>.eml` for R2).
- **IMAP:** the draft is appended to the Drafts folder of `sent_copy_account_id`, with a pointer in `storage_key`. Reopening fetches it back with `UID FETCH`; autosave appends the new version and expunges the previous copy; deleting or sending removes the copy from that mailbox.
- The mailbox copy is mirrored as an ordinary draft of the target account and filtered out of the Drafts list and draft searches, so the draft is listed once, under the relay account that owns it.

## Google Drive

- Per-user OAuth with `drive.file`; files live in a `MailClient` folder.
- 15 GB free for personal accounts.
- Service accounts have no storage quota and cannot be used as a free bucket.

## Cloudflare R2

- Optional binding named `BUCKET`.
- When the binding is absent, R2 is hidden in the UI.

## Cloudflare Email Service

> **Experimental.** The maintainer has not exercised it against a real Cloudflare account, so delivery, suppression handling (error `10205`), rate limits and per-recipient reporting are unverified.

- API key: API token with **Email Sending: Edit**.
- The sending domain must be onboarded under **Email Service → Email Sending**.
- WhelkMail enforces at most **50 recipients** and **15 MiB of attachments** per message.
- Custom SMTP can't target Cloudflare's own SMTP endpoint from Workers.

See [OAuth setup → Cloudflare Email Service](oauth#cloudflare-email-service) for the full setup.

## IMAP body archive

Relay accounts are not the only users of storage. IMAP accounts use external storage only when body archiving is enabled per account: by default attachments, Sent copies and drafts stay on the mail provider. The same Drive/R2 destinations can be used for that optional archive ("Archive bodies and attachments" in the account's sync section).

## Related

- [Mail flows](mail-flows)
- [Configuration](configuration)
- [Data model](data-model)
