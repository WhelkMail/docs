---
title: Upgrading
description: How to upgrade an existing WhelkMail deployment, run migrations and recover from problems.
---

# Upgrading WhelkMail

WhelkMail is self-hosted: nobody can push an update into your deployment. This guide is what the in-app notice links to when a newer release exists.

## Before you start

- The installed version is in **Settings → System → Version and updates**, together with the latest release it could find.
- A newer release is announced after the app compares your build with the latest published release on [github.com/WhelkMail/WhelkMail/releases](https://github.com/WhelkMail/WhelkMail/releases). The check runs from your Worker and is cached for a day; set `UPDATE_CHECK=off` to disable it.
- Read the release notes: they say whether the release adds a database migration. A release that needs one must not be deployed before the migration (or the app answers with a schema error).

## Upgrading a deployment

1. **Get the code.** Update your copy of the repository to the release tag:
   - GitHub fork: open your fork and use **Sync fork → Update branch**, then check that your fork's `main` matches the upstream tag.
   - Command line: `git fetch --tags && git checkout 1.5.0`.
2. **Deploy.** Choose the target you use:
   - **Workers:** `npm ci && npm run deploy`
   - **Pages:** push to the connected branch, or `npm ci && npm run deploy:pages`
   - **Docker:** `git pull && docker compose up -d --build`

3. **Migrate, if the release notes ask for it.**

   ```bash
   npm run db:migrate:remote
   ```

   Migrations are additive, so a release written against the previous schema keeps working while you run them; still, run them before using new features.

4. **Reload the app.** A tab left open on the old bundle shows a "reload" notice; after the reload, Settings → System reports the new version.

## Migrations in recent releases

**1.5.0 does not add a database migration.** It adds configuration export and import: **Settings → System → Configuration export and import** writes a passphrase-encrypted file with accounts, filters, labels, contacts and settings, and restores it on another deployment by re-encrypting the account secrets with that deployment's `TOKEN_ENC_KEY`. The import replaces the configuration and clears the local mail mirror; the object archive and two-factor settings are not included. See [Configuration](configuration#configuration-export-and-import).

**1.4.0 does not add a database migration.** It adds the Workers and Docker deployment targets and shared runtime adapters. Existing Pages installations keep the same D1 database, R2 bucket and secrets; remove any legacy local `wrangler.toml` because Pages now reads the tracked `wrangler.jsonc`.

Release 1.3.0 adds `031_recheck_aliases.sql` through `035_contact_vcard_fields.sql`:

- `031` marks rows with an empty alias list for one more read, so the fixed forwarding-header fetch can find `Delivered-To`, `X-Forwarded-For` and the other aliases.
- `032` adds `contacts.source` for the Gmail-style Other contacts badge. The code also saves contacts without it, but the badge needs the column.
- `033` adds the per-account vacation-responder columns, the `auto_reply_log` table and its indexes.
- `034` adds `accounts.signature_html` and `accounts.auto_reply_body_html` for rich-text signatures and replies.
- `035` adds the structured contact fields (first/last name, company, job title, phone, address, website, birthday, additional emails/phones).

Run `npm run db:migrate:remote` before deploying 1.3.0. Existing data is preserved, and signatures/auto-reply bodies saved before 1.3.0 keep working as plain text.

Release 1.2.0 added `030_account_label_filters.sql`: it adds `filters.auto_label`, creates the unique index for each account's automatic filter and converts existing `accounts.incoming_label_id` settings into editable filters.

## If something goes wrong

- **The app returns a schema error.** The code is newer than the database. Run `npm run db:migrate:remote`, then retry.
- **Settings → System says the check is unreachable.** Neither the GitHub API nor the releases feed could be reached from the Worker. The app keeps working; the last known answer stays cached. Run **Check again** to retry immediately. An optional `GITHUB_TOKEN` secret raises the API rate limit (the feed fallback needs no token).
- **Your fork is ahead of the latest release.** No notice is shown: only releases newer than your build are announced.

## Release check details

- The check runs in the Worker with an 8-second timeout; no browser or mailbox data is involved.
- The answer is cached in D1 for 24 hours (15 minutes after a failure).
- `UPDATE_CHECK=off` disables the check entirely.
- `GITHUB_TOKEN` is optional and only raises the GitHub API rate limit.

Releases and their notes are listed at [github.com/WhelkMail/WhelkMail/releases](https://github.com/WhelkMail/WhelkMail/releases).

## Related

- [Releases and publishing](development#releases)
- [Deployment](deployment)
- [PWA and Web Push](pwa-push#release-check)
