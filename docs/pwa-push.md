---
title: PWA and Web Push
description: Installable app, notifications, the cron worker and the in-app release check.
---

# PWA and Web Push

`public/manifest.webmanifest` and `public/sw.js` make the app installable. The service worker caches the app shell and static assets, never `/api/`, and handles `push` and `notificationclick`.

## How Web Push works

Web Push works in three parts:

1. The client asks for browser permission, registers the service worker and subscribes with the VAPID public key.
2. `push_subscriptions` and `push_prefs` persist the subscription and per-account/folder preferences.
3. The `worker/` cron Worker wakes up every minute and calls `POST /api/cron/notifications` with `CRON_SECRET`. That endpoint syncs the selected folders when the account's interval is due, detects new UIDs and sends encrypted Web Push messages with VAPID.

## Setup

Generate a VAPID key pair:

```bash
npm run vapid
```

Add the secrets to the Pages project:

```bash
npx wrangler pages secret put VAPID_PUBLIC_KEY --project-name whelkmail
npx wrangler pages secret put VAPID_PRIVATE_KEY --project-name whelkmail
npx wrangler pages secret put CRON_SECRET --project-name whelkmail
```

`VAPID_SUBJECT` can be a plain variable such as `mailto:you@example.com`.

Copy the cron worker template, set your Pages URL and deploy:

```bash
cp worker/wrangler.toml.example worker/wrangler.toml
# set APP_URL in worker/wrangler.toml to https://<your-project>.pages.dev
npx wrangler secret put CRON_SECRET -c worker/wrangler.toml
npm run cron:deploy
```

`npm run cron:deploy` creates the file from the template when it is missing and stops with a clear error while `APP_URL` is still empty or a placeholder.

The worker needs `APP_URL` and the same `CRON_SECRET` shared with the Pages app.

## What the cron worker does

Every minute, in the same run, it:

- sends Web Push notifications for new messages;
- applies filter labels in every folder and forwards matching inbox messages;
- runs the optional provider cleanup;
- advances the per-account automatic background sync, once each account's interval has elapsed.

The worker template uses `crons = ["* * * * *"]`. On Workers Free, use `["*/5 * * * *"]` instead: the 1 and 2 minute intervals are not usable on that plan.

Enable notifications from **Settings → Push notifications**, background sync per account in Settings, and forwarding/deletion in the account/filter sections.

## PWA assets

`public/notification-badge.png` (96x96) and `public/notification-icon.png` (192x192) are generated from the shell mark in `public/whelkmail-logo.svg`:

```bash
npm run icons
```

Android masks `badge` into a monochrome silhouette in the status bar and does not render SVG there, which is why both files are PNG and why the badge is white on a transparent background. The badge drops the thin decorative strokes of the logo: at status bar size they only add noise. After changing the files, bump `CACHE` in `public/sw.js` so installed clients pick up the new assets.

## Release check

`GET /api/update` compares `APP_VERSION` (`shared/version.ts`, mirrored in `package.json`) with the latest release on `WhelkMail/WhelkMail`.

- The GitHub call runs in the Worker with an 8-second timeout, never in the browser, so no visitor IP or mailbox data leaves the deployment.
- `UPDATE_CHECK=off` disables it and `GITHUB_TOKEN` optionally raises the rate limit.
- The check reads `/releases/latest` from the API and falls back to the releases Atom feed when the API fails, so it does not depend on the API alone.
- The answer is cached in `app_settings.update_check` for 24 hours (15 minutes after a failure or when no release exists yet), so each deployment makes at most one request per day.
- A cached copy is still served when GitHub is unreachable, and `GET /api/update?refresh=1` (Settings → System → **Check again**) bypasses the cache.
- Failures are logged from the Worker.

When the latest release is newer, the client shows a dismissible banner with the upgrade steps and a link to the upgrade guide; the dismissal is stored server-side (`update_notice_dismissed`) per release version. `POST /api/update` records it. `GET /api/health` exposes `version` too, so a tab left open on an older bundle can offer a reload when it sees a newer server.

The check is informational: it cannot update a deployment.

## Related

- [Deployment](deployment)
- [Upgrading](upgrading)
- [Configuration](configuration)
