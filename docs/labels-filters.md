---
title: Labels and filters
description: Shared Gmail-style labels, per-account automatic labels, contacts, signatures and automatic filters.
---

# Labels and filters

## Account chips

Messages carry `accountId`, `accountName` and `accountEmail`. The unified view shows a deterministic color label per connected address, and the account bar filters by address. This is address-level labeling, not alias-level labeling.

## Custom labels

Custom labels are separate and user-defined, shared across accounts. They can be created from the sidebar or from the filter editor.

Clicking a label opens a label view: the API reads `message_labels`, groups the assigned message ids by account and folder, fetches those IMAP UIDs directly from the provider and merges them with relay messages. The view therefore covers Inbox, Sent, Drafts and Spam across the selected accounts, like Gmail, without mirroring the whole mailbox locally. Removed or expunged messages simply disappear from the view.

When a message moves to another folder on the provider, its UID changes and the local label assignment no longer applies. The optional IMAP metadata sync mirrors headers and UIDs, but labels still follow the UID, so a move can lose the local assignment until the message is labeled again.

## Automatic account labels

Each account can define an `incoming_label_id`. The label is chosen while adding the account or in Settings; if it does not exist, it is created.

The automatic label is stored as a real filter: a match-all rule scoped to that account and marked with `auto_label`. It appears in Settings -> Filters like any other rule and can be edited or disabled there. It applies to every message of the account in every folder, not only the inbox. Changing the label in Settings updates the filter, and editing the filter keeps the account setting in step. "Apply every filter" also labels the messages that were already present.

A send-only relay account has no mailbox, so its automatic label follows the relay address into the IMAP account that receives the forwarded mail. The label is applied when the message's recipients or forwarding aliases contain that address. If the editable auto-label filter row is missing (for example on an older schema), `applyFilters` still applies `accounts.incoming_label_id` directly so old and new mail cannot diverge.

## Filters

Automatic filtering is managed in Settings -> Filters. A filter with an empty `account_id` applies to every account; a filter with an account id only sees that account's mail. The automatic account label is one of these account-scoped, match-all filters.

- Criteria are ANDed and matching is case-insensitive.
- Match fields: From, To, Subject, words, words that must not appear, and attachment presence.
- Actions can apply one or more labels, forward to recipients, or both, like Gmail.
- Matching runs whenever a folder is listed and whenever the sync ingests new messages, in every folder: Inbox, Sent, Drafts, Trash and Spam.
- Applying a filter to mail already received walks the IMAP mirror and the local `messages` rows, so relay sent copies and drafts are covered too.
- The match text covers sender, recipients, subject and snippet, **not** the full body. A message mirrored from headers only has an empty snippet: word criteria are not decided yet and are re-evaluated as soon as the body is indexed. The list endpoint re-applies filters to the visible messages as a safety net, so a failed sync chunk does not leave the page unlabelled.
- The IMAP header fetch includes the forwarding headers (`Delivered-To`, `X-Delivered-To`, `X-Original-To`, `Envelope-To`, `X-Envelope-To`, `X-Forwarded-To`, `X-Forwarded-For`, `Resent-To`, `Apparently-To` and `Received ... for`). A filter on **To** therefore matches a forwarding alias even when the forwarder rewrites the original `To`.
- A conversation row shows the union of the labels of the whole conversation, not only of its newest message. A thread selected by a label view keeps showing that label when a newer reply is not labelled yet.

The label batches are chunked so a page with many messages and labels stays inside D1's statement limit.

Forwarding uses the original EML, attaches it as `message/rfc822` and stamps `X-WhelkMail-Forwarded` to prevent loops. Forwarding remains an incoming-mail action: matching messages are forwarded only from the inbox, even though labels are applied in every folder. See [Forwarding and cleanup](forwarding-cleanup).

## Contacts

Contacts are a shared address book stored in D1, not R2/Drive. Each row has a display name, primary email, note, source, first/last name, company, job title, phone, address, website, birthday, additional emails/phones and a creation timestamp. A unique index on `lower(email)` prevents duplicates; creating a contact with an existing email updates it instead. Contacts are not account-scoped.

There are two sources:

- `manual`: entered or imported explicitly;
- `auto`: the Gmail-style **Other contacts** created from recipients. Sending a message adds its To/Cc/Bcc recipients unless Settings -> General has "I add contacts myself". Opening a reply adds unknown recipients silently. Editing an automatic contact promotes it to `manual`.

The recipient fields offer autocomplete with **prefix matching**: every typed token must start a word of the name/company or the email itself. `m` matches "Mario Rossi" but not `indirizzo@gmail.com`. A valid address that is not in the book shows an **Add <email> to contacts** entry in the same menu; pressing Enter on it creates the contact and commits the address. Enter in a recipient never sends the message.

The UI supports search, create/edit/delete, CSV import/export, vCard import (3.0 and 4.0 are detected automatically) and vCard export (3.0 by default, 4.0 optional). Bulk import goes through `POST /api/contacts/import` in batches and upserts by email.

## Signatures

`signature_mode` is `shared` (default) or `per_account`:

- `shared`: `default_signature` / `default_signature_html` are used for every account and `accounts.signature` is ignored.
- `per_account`: every account uses its own `accounts.signature` / `accounts.signature_html`; an empty value means no signature.

Signatures are edited with the rich-text composer. The HTML version is inserted in rich mode and the plain-text version is used in plain mode and as the text fallback. Existing signatures saved before rich text keep working as plain text and are converted when edited. The composer's default rich/plain format is stored per device (localStorage), so desktop and phone can differ.

## Conversations and labels

Label views and search results are grouped into conversations like the standard folders: a label view loads the whole conversation for the conversations that carry the label, and search collapses the matching messages of a thread into one row whose count refers to the matches on that page.

## Related

- [Data model](data-model)
- [Mail flows](mail-flows)
- [Auto-reply](auto-reply)
- [Forwarding and cleanup](forwarding-cleanup)
