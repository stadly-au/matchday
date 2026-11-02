# Deployment

Part of the [AGENTS Guide](../AGENTS.md).

## Production

Two Workers deploy from `main` through `.github/workflows/deploy.yml`, after the database
migration job:

- `matchday-api` — the REST API, from `apps/api/wrangler.jsonc`.
- `matchday-scheduler` — the cron Worker that fires the crawl workflows, from
  `apps/scheduler/wrangler.jsonc`.

`matchday-api` reads `DATABASE_URL` and `SENTRY_DSN` as Cloudflare secrets. `matchday-scheduler`
reads `GITHUB_TOKEN`, sourced from the `GH_DISPATCH_TOKEN` repository secret because Actions
reserves the `GITHUB_TOKEN` name. Both deploys upload their secrets with wrangler-action's
`secrets:` input, so GitHub is the source of truth and nothing is set by hand in the Cloudflare
dashboard. Secrets never go in `wrangler.jsonc`, which is committed and holds plaintext `vars`
only.

## Pull request previews

Each pull request gets one Cloudflare Worker Preview of `matchday-api`, named
`pr-<number>-<branch>` and served at
`https://<preview>-matchday-api.<account>.workers.dev`. The URL is read from wrangler's output
file, commented on the pull request, and smoke tested. Previews exist only for the API: the
scheduler is a cron Worker, and Previews never run crons.

A Preview is a branch of the production Worker, not a Worker of its own, so it appears in the
dashboard's Preview picker and Cloudflare evicts the least recently deployed Preview once a
Worker holds 500.

`preview_urls: true` in `apps/api/wrangler.jsonc` is what gives a Preview its workers.dev URL.
It defaults to off, so a Preview has nowhere to answer until the production deploy has run once
with that setting. Previews inherit no `vars` or bindings, so the `previews` block repeats
`LOG_LEVEL` and `ENVIRONMENT`. Add a var or binding to one and you must add it to the other, or
the Preview runs without it.

`deploy-preview.yml` uploads the Preview's secrets with `wrangler preview secret bulk --name`,
piped and never written to disk. Wrangler-action's `secrets:` input cannot name a Preview and
would write to the production Worker, so it is not used here. `ENVIRONMENT` is `preview`, which
keeps Preview errors in Sentry under their own tag.

A Preview reads the Neon branch for the same pull request (`pr-<number>`), creating it if CI has
not yet, and applies migrations to it. That branch is cut from `production`, so it holds a copy of
the real data, and a branch connection string must name the `matchday` database — Neon's default
is `neondb`, which would migrate an empty database instead. CI's integration tests use the same
branch, so both workflows take one concurrency lock while they migrate; drizzle-kit does not lock
by itself and two migrators would collide on a new migration.

Closing a pull request deletes its Neon branch and its Preview. `.github/workflows/cleanup-branch.yml`
removes the Preview with the same API call `wrangler preview delete` makes, and treats an already
deleted Preview as success.
