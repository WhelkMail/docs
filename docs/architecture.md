---
title: Architecture
description: How WhelkMail's React client, Cloudflare Pages Functions, D1 and IMAP/SMTP fit together.
---

# Architecture

## Scope

WhelkMail is a **webmail client**, not a mail server.

- It connects to existing mailboxes through IMAP and SMTP.
- It does not receive mail directly, does not run an MTA, and does not listen on ports 25/143/993/465/587.
- It does not create mailboxes. Thunderbird and mobile clients can keep connecting to the same provider independently.
- For domains with no SMTP, a separate **relay account** type sends through an HTTP email API and keeps sent copies/drafts on Google Drive, Cloudflare R2 or in the Sent/Drafts folders of another IMAP account. Cloudflare Email Service is available as an experimental relay.

## High-level diagram

```
Browser (React/Vite)
        |
        | HTTPS, same-origin /api
        v
Cloudflare Pages Functions (Workers runtime)
        |
        +-- D1 (accounts, message metadata, relay archive, drafts)
        +-- IMAP over cloudflare:sockets (TLS / STARTTLS)
        +-- SMTP over cloudflare:sockets (TLS / STARTTLS)
        +-- OAuth: Google and Microsoft (XOAUTH2), Google Drive (archive)
        +-- HTTP relay APIs (send-only accounts)
        +-- R2 binding (optional, relay/IMAP archive)
```

Key constraints:

- Browsers cannot speak IMAP/SMTP, so all mailbox traffic goes through Pages Functions.
- Cloudflare Workers can open **outbound** TCP sockets with `cloudflare:sockets`. They cannot accept inbound TCP connections.
- Port 25 is blocked by Cloudflare and is not used.
- IMAP `IDLE` is not supported across requests. The UI refreshes on navigation, after sending, or on reload.

## Project structure

```
functions/                 Cloudflare Pages Functions
  _lib/
    imap/client.ts         IMAP client over cloudflare:sockets
    smtp/client.ts         SMTP client (PLAIN, LOGIN, XOAUTH2)
    mail/                  auth helpers, folder lookup, drafts, loading
    mime.ts                MIME builder
    storage/               Drive and R2 adapters
    relay/                 HTTP relay adapters
    microsoft/oauth.ts     Microsoft OAuth2 / XOAUTH2 token flow
  api/                     HTTP endpoints
shared/                    Types, Zod schemas, provider presets, paging helpers
src/                       React application
db/schema.sql              D1 schema for a fresh install
db/migrations/             Additive upgrades for existing databases
wrangler.toml.example      local Pages/D1 configuration template
worker/wrangler.toml.example  cron worker template (the real file is gitignored)
```

## Runtime pieces

| Layer | Responsibility |
|---|---|
| React app (`src/`) | UI, state machine, conversations, composer, settings |
| Pages Functions (`functions/`) | Authentication, IMAP/SMTP connections, MIME building, storage, relay calls |
| D1 (`DB`) | Accounts, settings, mirrored IMAP metadata, relay messages, short-lived body cache |
| R2 (`BUCKET`, optional) | Optional long-term archive and relay sent/draft storage |
| Google Drive (optional) | Alternative archive and relay sent/draft storage |
| Cron worker (`worker/`) | Web Push, automatic sync, forwarding and provider cleanup every 5 minutes |

## Design choices

- **Provider-first by default.** Messages, sent copies, drafts and attachments stay on the mail provider. Local copies are caches or opt-in archives.
- **No IMAP IDLE.** Workers cannot hold a long-lived connection across requests, so metadata is mirrored into D1 and refreshed in bounded chunks.
- **One shared app password.** WhelkMail is a personal / small-team tool, not a multi-tenant service.
- **One unified taxonomy.** Labels, filters, contacts and signatures are shared across accounts.

## Related

- [Mail flows](mail-flows) — the request flows end to end.
- [Data model](data-model) — every table and its role.
- [Security](security) — the threat model and protections.
