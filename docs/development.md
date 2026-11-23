---
title: Development
description: Work on WhelkMail locally: repository model, database migrations, tests, releases and publishing.
---

# Development and publishing

This page describes how to work on WhelkMail, how the two repositories relate, and what must never end up in the public one.

## Repositories

| Repository | Role |
| --- | --- |
| `mlzog/WhelkMail` (development, remote `origin`) | development. Commits can be small, messy and frequent. |
| `WhelkMail/WhelkMail` (remote `public`) | distribution. Anyone can install it. |

Both repositories share **one history**: `public/main` is an ancestor of the development branch (see the unification merge). That keeps the two in sync with plain merges and fast-forward pushes, and it is why a PR merged on GitHub is brought back with `git merge public/main` instead of a cherry-pick — a cherry-pick would create a duplicate commit for content that already exists.

## Local setup

Requirements: Node.js 24+ and npm (the Docker image uses Node 24).

```bash
npm install
npm run db:init:local   # fresh local database: schema + migration baseline
```

The tracked `wrangler.jsonc` carries the shared Workers/Pages configuration
without a `database_id`, so the Deploy to Cloudflare flow and `wrangler deploy`
can provision D1. Remove any legacy local `wrangler.toml`: Pages does not
support custom configuration paths.

Create `.dev.vars` (gitignored) with at least:

```env
APP_PASSWORD=change-me
TOKEN_ENC_KEY=<32-byte base64 key>
```

Generate the key with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

Run the app:

```bash
npm run dev            # Vite on http://localhost:5173
npm run dev:workers    # Worker + local D1 on http://localhost:8787
# or
npm run dev:pages      # Pages Functions on http://localhost:8788
```

Node/Docker locally:

```bash
npm run build:all
APP_PASSWORD=... TOKEN_ENC_KEY=... DB_PATH=./data/dev.sqlite \
  PUBLIC_ORIGIN=http://localhost:8787 node dist-server/server.mjs
```

`npm run build:node` bundles `server/entries/node.ts` with esbuild; the runtime
uses the built-in `node:sqlite` module and needs no `node_modules`.

### Working from the Acode terminal on Android

Node is not installed in the Acode Alpine terminal by default:

```bash
apk add nodejs npm
```

Alpine uses musl, and `wrangler dev` / `wrangler d1 --local` need the `workerd` binary, which may not run there. `typecheck`, `lint`, `test` and `build` work fine. When something needs a real Worker runtime, push to GitHub and use a Cloudflare Pages preview deployment instead of a local emulator.

## Database changes

- `db/schema.sql` is the schema for a **fresh install** and must always contain every column.
- `db/migrations/NNN_name.sql` upgrades an **existing** database. They are additive `ALTER`/`CREATE` statements, not idempotent, so never run them on a fresh database.
- `npm run db:migrate:local` / `db:migrate:remote` apply the missing migrations in order and record each one in `schema_migrations`, so a rerun skips what is already done and an interrupted upgrade resumes.
- `npm run db:init:local` / `db:init:remote` are for a **new** database: they apply `schema.sql` and then record the current migrations as the baseline.
- `npm run db:local` / `db:remote` apply only `schema.sql`. Never follow them with `--mark-applied` on an existing database: `CREATE TABLE IF NOT EXISTS` cannot add columns, and the registry would claim migrations that were never applied. The script refuses to record a migration whose columns or tables are missing.
- On a database created before the registry existed the table is empty while the schema is complete: the script stops and asks for either `--mark-applied` (record everything, run nothing) or `--from=NNN` (start from one migration).
- D1 runs the statements of a file one by one, so a file that fails halfway is partly applied and unrecorded. The script prints the recovery commands: `--only=NNN` re-runs that file alone, and `--only=NNN --mark-applied` records it when the remaining statements are already in place.
- `--mark-applied` verifies the columns, tables and indexes each migration creates before recording it, and refuses if any is missing. It does **not** verify data changes (`UPDATE`/`DELETE` statements): check those by hand.
- `--dry-run` prints the plan without touching the database.

When you add a table or column, update `db/schema.sql` **and** add a migration.

## Tests

```bash
npm run typecheck
npm run lint
npm run test
npm run build
```

CI runs the same four steps on every push to `main` and every pull request, plus a guard that fails when a personal config file is tracked.

## Releases

The version lives in `package.json` (mirrored in `package-lock.json`) and `APP_VERSION` in `shared/version.ts`; a test keeps the two source values in step.

1. Bump both to the new semantic version.
2. Run the four checks above, then publish to the public repository.
3. Create the tag and the release on GitHub:

   ```bash
   git tag 1.4.0
   git push public 1.4.0
   ```

   Then publish the release from the tag on [github.com/WhelkMail/WhelkMail/releases](https://github.com/WhelkMail/WhelkMail/releases). Mark the tag as a prerelease when it is a beta: `/releases/latest` skips prereleases, so installations are not told about them.
4. In the release notes, state whether the release adds a database migration.

Every deployment compares its `APP_VERSION` with the latest release (cached in D1 for a day) and shows an in-app notice with the upgrade steps. Anyone who builds from a commit between two releases runs a version that already matches the last tag, so they are not prompted until the next release is published. Keep the repository's `main` and the release tag in sync to keep that comparison honest.

## Publishing to the public repository

One-time setup:

```bash
git remote add public https://github.com/WhelkMail/WhelkMail.git
git fetch public
```

Then:

```bash
sh scripts/publish-public.sh                  # publish main
SOURCE=feature-x sh scripts/publish-public.sh  # publish another branch
```

The script refuses to run if a personal config file is tracked in `SOURCE`: `wrangler.toml`, `worker/wrangler.toml`, `.dev.vars` and its variants, `.env` and `.env.*` with any number of suffixes. The `*.example` templates are allowed and expected, including `.dev.vars.example` (used by the Deploy to Cloudflare setup page). `wrangler.jsonc` is tracked without a `database_id` so the Deploy flow and forks stay generic.

It then pushes `SOURCE` to `public/main` as a **fast-forward**. If `public/main` has commits that are missing locally the script stops and asks for a merge first:

```bash
git merge public/main        # brings a PR merged on GitHub into development
sh scripts/publish-public.sh # publishes back, fast-forward
```

Before publishing, make sure the public tree is installable:

```bash
git ls-files | grep -E 'wrangler\.toml$|\.dev\.vars' && echo "unexpected config"
! grep -q '"database_id"' wrangler.jsonc
npm ci && npm run typecheck && npm run test && npm run build && npm run build:node
```

The public repository must not contain:

- `wrangler.toml` or `worker/wrangler.toml` (they carry the Pages URL and account-scoped resource identifiers);
- `.dev.vars`, `.env*` or any secret;
- `node_modules/`, `dist/`, `.wrangler/` (local D1 state) or `coverage/`;
- seed data: `db/schema.sql` must stay `CREATE` statements only, with no `INSERT` of real accounts, contacts or messages.

`git ls-files` is the source of truth: anything tracked is published, anything untracked is not.

## Where things live

| Path | Contents |
| --- | --- |
| `src/` | React app (`App.tsx` holds the state machine, `components/` the UI) |
| `functions/api/` | API endpoint handlers (Pages native routing; Workers/Node use `server/routes.ts`) |
| `functions/_lib/` | IMAP/SMTP, D1 access, sync, cache, filters, crypto |
| `server/` | runtime-agnostic dispatcher, route table and Workers/Node/SQLite/S3 adapters |
| `server/entries/` | Workers and Node entrypoints (Pages needs none) |
| `deploy/docker/` | Dockerfile, compose file and environment example |
| `shared/` | types, schemas, presets and paging helpers shared by both sides |
| `worker/` | Pages cron worker (`index.ts` + example config) |
| `db/` | schema and migrations |
| `scripts/` | helper scripts (VAPID keys, migrations, worker config, publishing) |
| `tests/` | Vitest suites |

## Internationalization

- Default locale: English.
- Italian included.
- Choice persisted in `localStorage` (`whelkmail_locale`, falling back to the legacy `mc_locale`).
- UI strings live in `src/i18n.tsx`; add a locale object and a `LOCALES` entry to extend.
- Provider/server error strings are not translated.

## Related

- [Architecture](architecture)
- [Data model](data-model)
- [Upgrading](upgrading)
