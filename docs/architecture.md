---
title: Architecture
description: How WhelkMail's React client, runtime-agnostic API core, D1/SQLite and IMAP/SMTP fit together.
---

# Architecture

## Scope

WhelkMail is a **webmail client**, not a mail server.

- It connects to existing mailboxes through IMAP and SMTP.
- It does not receive mail directly, does not run an MTA, and does not listen on ports 25/143/993/465/587.
- It does not create mailboxes. Thunderbird and mobile clients can keep connecting to the same provider independently.
- For domains with no SMTP, a separate **relay account** type sends through an HTTP email API and keeps sent copies/drafts on Google Drive, R2/S3 or in the Sent/Drafts folders of another IMAP account. Cloudflare Email Service is available as an experimental relay.

## High-level diagram

```
Browser (React/Vite)
        |
        | HTTPS, same-origin /api
        v
API core (server/app.ts + functions/api handlers)
        |
        +-- Cloudflare Workers (default): D1, R2, cloudflare:sockets, cron triggers
        +-- Cloudflare Pages (legacy):   D1, R2, cloudflare:sockets, worker/ cron
        +-- Docker / Node 24:            SQLite (node:sqlite), S3, node:net/tls, built-in cron
        |
        +-- OAuth: Google and Microsoft (XOAUTH2), Google Drive (archive)
        +-- HTTP relay APIs (send-only accounts)
```

Key constraints:

- Browsers cannot speak IMAP/SMTP, so all mailbox traffic goes through the API backend.
- Cloudflare Workers can open **outbound** TCP sockets with `cloudflare:sockets`; Node/Docker uses `node:net`/`node:tls`. Neither accepts inbound TCP connections.
- Port 25 is blocked by Cloudflare and is not used.
- IMAP `IDLE` is not supported across requests. The UI refreshes on navigation, after sending, or on reload.

## Project structure

```
functions/                 API endpoint handlers (Pages native routing)
  _lib/
    mail/socket.ts         pluggable mail transport
    imap/client.ts         IMAP client (transport injected at startup)
    smtp/client.ts         SMTP client (PLAIN, LOGIN, XOAUTH2)
    mail/                  auth helpers, folder lookup, drafts, loading, search text
    mime.ts                MIME builder (text + HTML alternative)
    filters.ts             filter matching, labels, relay alias scope
    contacts.ts            contact rows/values, auto contacts, vCard field parsing
    auto-reply.ts          vacation responder, cooldown and auto_reply_log
    storage/               Drive and R2 adapters
    relay/                 HTTP relay adapters
    microsoft/oauth.ts     Microsoft OAuth2 / XOAUTH2 token flow
  api/                     HTTP endpoints, including POST /api/contacts/import
server/                    Runtime-agnostic dispatcher, route table, Workers/Node entries
  adapters/                 D1/SQLite, R2/S3, cloudflare:sockets/node:net+tls
deploy/docker/             Dockerfile, compose file and environment example
shared/                    Types, Zod schemas, provider presets, paging helpers
src/                       React application
  components/RichTextEditor.tsx  contentEditable composer/toolbar
  lib/vcard.ts             vCard 3.0/4.0 parser and serializer
db/schema.sql              D1/SQLite schema for a fresh install
db/migrations/             Additive upgrades for existing databases
wrangler.jsonc             shared Workers/Pages configuration (placeholder D1 id)
worker/wrangler.toml.example  Pages cron worker template (the real file is gitignored)
```

## Runtime pieces

| Layer | Responsibility |
|---|---|
| React app (`src/`) | UI, state machine, conversations, composer, settings |
| API core (`functions/` + `server/`) | Authentication, IMAP/SMTP connections, MIME building, storage, relay calls |
| D1 (`DB`) or SQLite | Accounts, settings, mirrored IMAP metadata, relay messages, short-lived body cache |
| R2/S3 (`BUCKET`, optional) | Optional long-term archive and relay sent/draft storage |
| Google Drive (optional) | Alternative archive and relay sent/draft storage |
| Scheduler | Workers cron trigger, Pages `worker/` cron, or the Docker scheduler: Web Push, automatic sync (per-account interval), forwarding, vacation responder and provider cleanup |

## Design choices

- **Provider-first by default.** Messages, sent copies, drafts and attachments stay on the mail provider. Local copies are caches or opt-in archives.
- **No IMAP IDLE.** Serverless targets cannot hold a long-lived connection across requests, so metadata is mirrored into D1/SQLite and refreshed in bounded chunks. Docker uses the same bounded-refresh model for consistency.
- **One shared app password.** WhelkMail is a personal / small-team tool, not a multi-tenant service.
- **One unified taxonomy.** Labels, contacts and signatures are shared across accounts. Filters are global by default but can be scoped to one account; the automatic account label is one of these scoped filters.

## Related

- [Mail flows](mail-flows) — the request flows end to end.
- [Data model](data-model) — every table and its role.
- [Security](security) — the threat model and protections.
