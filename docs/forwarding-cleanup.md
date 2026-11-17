---
title: Forwarding and cleanup
description: Filter-based forwarding, loop protection and opt-in provider cleanup.
---

# Forwarding and provider cleanup

Both features are executed by the cron worker, so they share the same setup as Web Push (the `worker/` cron Worker and `CRON_SECRET`).

## Forwarding

Forwarding is a filter action: a filter can apply labels, forward to one or more recipients, or both, like Gmail.

- The cron endpoint processes messages above the previous inbox `max_uid`, evaluates enabled filters and the account incoming label, then forwards messages matched by a filter with `forward_to_json`.
- The forwarded message is built from the original EML, attached as `message/rfc822`, and stamped with `X-WhelkMail-Forwarded`; messages carrying that header are skipped to prevent loops.
- `forward_log` has a unique `(filter_id, message_id)` key so a message is forwarded once per filter.
- Forwarding only acts on new incoming messages detected after the initial sync.

## Provider cleanup

Provider cleanup is opt-in per account and requires body archiving.

- Enable **Archive bodies and attachments** and **Delete after archive** for the account.
- For a new inbox message with `archived = 1` and an `archive_key`, the cron copies the message to the provider Trash (`\Trash`, with name fallbacks), removes it from the inbox and writes `deletion_log`.
- The archived R2/Drive object is preserved when this feature is enabled.
- It never performs a permanent delete from the provider.

## Requirements

- The cron worker must be deployed and configured (see [PWA and Web Push](pwa-push)).
- Archive destinations must be connected: a `BUCKET` binding for R2, or Google Drive OAuth for Drive.
- Failed attempts are logged but not retried automatically.

## Related

- [Labels and filters](labels-filters)
- [Data model](data-model)
- [Deployment](deployment)
