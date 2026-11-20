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

## Filters

Automatic filtering is managed in Settings -> Filters. A filter with an empty `account_id` applies to every account; a filter with an account id only sees that account's mail. The automatic account label is one of these account-scoped, match-all filters.

- Criteria are ANDed and matching is case-insensitive.
- Match fields: From, To, Subject, words, words that must not appear, and attachment presence.
- Actions can apply one or more labels, forward to recipients, or both, like Gmail.
- Matching runs whenever a folder is listed and whenever the sync ingests new messages, in every folder: Inbox, Sent, Drafts, Trash and Spam.
- Applying a filter to mail already received walks the IMAP mirror and the local `messages` rows, so relay sent copies and drafts are covered too.
- The match text covers sender, recipients, subject and snippet, **not** the full body.

Forwarding uses the original EML, attaches it as `message/rfc822` and stamps `X-WhelkMail-Forwarded` to prevent loops. Forwarding remains an incoming-mail action: matching messages are forwarded only from the inbox, even though labels are applied in every folder. See [Forwarding and cleanup](forwarding-cleanup).

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
