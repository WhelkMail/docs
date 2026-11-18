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
   - Command line: `git fetch --tags && git checkout v1.2.3`.
2. **Deploy.** Either let Cloudflare Pages redeploy (it builds from the branch you pushed) or run it yourself:

   ```bash
   npm ci
   npm run deploy
   ```

3. **Migrate, if the release notes ask for it.**

   ```bash
   npm run db:migrate:remote
   ```

   Migrations are additive, so a release written against the previous schema keeps working while you run them; still, run them before using new features.

4. **Reload the app.** A tab left open on the old bundle shows a "reload" notice; after the reload, Settings → System reports the new version.

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
