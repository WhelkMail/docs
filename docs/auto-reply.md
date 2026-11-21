---
title: Auto-reply
description: Shared or per-account vacation responder, its settings, runtime and anti-loop rules.
---

# Auto-reply (vacation responder)

WhelkMail can answer incoming mail automatically with a vacation responder. It is configured in **Settings -> Auto-reply** and supports two modes:

- **Shared**: one enabled flag, subject and body for every receiving account.
- **Per account**: each receiving IMAP account has its own enabled flag, subject and body.

Relay accounts are send-only and have no inbox, so they are not listed in the per-account editor. A shared auto-reply uses the account that received the message.

## Configuration

| Mode | Storage |
| --- | --- |
| Shared | `app_settings`: `auto_reply_enabled`, `auto_reply_subject`, `auto_reply_body`, `auto_reply_body_html` |
| Per account | `accounts`: `auto_reply_enabled`, `auto_reply_subject`, `auto_reply_body`, `auto_reply_body_html` |

`app_settings.auto_reply_mode` is `shared` (default) or `per_account`.

The body is rich text. `body_html` is sent as the HTML part and `body` is the plain-text fallback; rows saved before rich text existed keep working with plain text only. The subject is optional: when empty, WhelkMail sends `Re: <original subject>`.

## When it runs

- The reply runs for **new inbox messages**. The client-triggered sync handles them as soon as they are mirrored, and the notification cron handles messages that were not seen by a client. The `auto_reply_log` unique key on `(account_id, message_id)` makes a retried chunk harmless.
- The first sync of an account does not fire a mailbox-wide auto-reply; only mail after that is considered.
- Mail older than **seven days** is ignored.
- A sender receives at most **one reply every 24 hours**, even if they send several messages.

## Anti-loop and safety rules

WhelkMail skips:

- messages that are not in the inbox;
- its own account addresses;
- `no-reply`, `noreply`, `do-not-reply`, `mailer-daemon`, `postmaster`, `bounce` and `notifications` senders;
- subjects that already look like an automatic reply (`Auto reply`, `Out of office`, `Fuori sede`, `Risposta automatica`);
- a message that already has a row in `auto_reply_log`.

Replies carry `Auto-Submitted: auto-replied` and `X-Auto-Response-Suppress: All` so another responder does not answer them. These are heuristics, not a guarantee: a misconfigured responder that ignores the headers can still loop once until the 24-hour sender cooldown stops it.

## Requirements

The cron worker is the normal background path. Without it, the responder still runs when a client syncs new inbox mail, but nobody answers while the app is closed. See [PWA and Web Push](pwa-push) and [Deployment](deployment) for the worker setup.

The reply is sent through the receiving account's own SMTP or relay configuration. If that account cannot send, the attempt is recorded as `error` in `auto_reply_log` and no message leaves.

## Related

- [Data model](data-model)
- [Mail flows](mail-flows)
- [Configuration](configuration)
- [Troubleshooting](troubleshooting)
