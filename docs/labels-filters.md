---
title: Labels and filters
description: Shared Gmail-style labels, per-account incoming labels, contacts, signatures and automatic filters.
---

# Labels and filters

## Account chips

Messages carry `accountId`, `accountName` and `accountEmail`. The unified view shows a deterministic color label per connected address, and the account bar filters by address. This is address-level labeling, not alias-level labeling.

## Custom labels

Custom labels are separate and user-defined, shared across accounts. They can be created from the sidebar or from the filter editor.

Clicking a label opens a label view: the API reads `message_labels`, groups the assigned message ids by account and folder, fetches those IMAP UIDs directly from the provider and merges them with relay messages. The view therefore covers Inbox, Sent, Drafts and Spam across the selected accounts, like Gmail, without mirroring the whole mailbox locally. Removed or expunged messages simply disappear from the view.

When a message moves to another folder on the provider, its UID changes and the local label assignment no longer applies. The optional IMAP metadata sync mirrors headers and UIDs, but labels still follow the UID, so a move can lose the local assignment until the message is labeled again.

## Incoming labels

Each account can define an `incoming_label_id`. When the inbox is listed, every incoming message of that account receives the shared label automatically. The label is chosen while adding the account or in Settings; if it does not exist, it is created.

## Filters

Automatic filtering is managed in Settings â†’ Filters and is shared across accounts.

- Criteria are ANDed and matching is case-insensitive.
- Match fields: From, To, Subject, words, words that must not appear, and attachment presence.
- Actions can apply one or more labels, forward to recipients, or both, like Gmail.
- Matching runs when the inbox is listed and when the cron worker syncs new inbox messages.
- The match text covers sender, recipients, subject and snippet, **not** the full body.

Forwarding uses the original EML, attaches it as `message/rfc822` and stamps `X-WhelkMail-Forwarded` to prevent loops. See [Forwarding and cleanup](forwarding-cleanup).

## Contacts

Contacts are a shared address book: `id`, `name`, `email`, `note`, `created_at`. A unique index on `lower(email)` prevents duplicates; creating a contact with an existing email updates the name/note instead. Contacts are not account-scoped.

The UI supports search, create/edit/delete, CSV import/export, and autocomplete in To/Cc/Bcc.

## Signatures

`signature_mode` is `shared` (default) or `per_account`:

- `shared`: `default_signature` is used for every account and `accounts.signature` is ignored.
- `per_account`: every account uses its own `accounts.signature`; an empty value means no signature.

## Conversations and labels

Label views and search results are grouped into conversations like the standard folders: a label view loads the whole conversation for the conversations that carry the label, and search collapses the matching messages of a thread into one row whose count refers to the matches on that page.

## Related

- [Data model](data-model)
- [Mail flows](mail-flows)
- [Forwarding and cleanup](forwarding-cleanup)
