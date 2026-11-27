---
status: accepted
---

# Fly.io and Neon, deployed from `main` by one GitHub Actions workflow

Production is one always-on Fly.io Machine (512 MB, region `fra`) running the app's Docker image, with its data in Neon's free PostgreSQL 18 in `aws-eu-central-1`, so the app and its database sit in the same city. It is served at `https://splitr.io`, and sign-in email goes out through Resend's HTTP API from `no-reply@mail.splitr.io`. One GitHub Actions workflow runs on every push to `main`. A `check` job lints, typechecks and tests the whole workspace against a PostgreSQL 18 service container, and a `deploy` job runs `flyctl deploy --remote-only` only after it passes. Pull requests run `check` only. This is the only free-or-near-free combination that keeps the app a long-running Node process that is always on: the Machine fits inside Fly's $5 monthly allowance, and Neon never pauses the project.

## Considered Options

- **Render free.** Rejected: the app sleeps after 15 minutes and takes about a minute to wake, which is the first thing a friend opening a sign-in link would see.
- **Render paid, or a second Fly Machine.** Rejected on cost: $7 or more a month, above the ceiling. A second Machine would remove the few seconds of downtime per deploy.
- **Railway Hobby.** Rejected: $5 a month, and whether the app and Postgres fit inside the included usage is unverified.
- **Vercel or Cloudflare Workers.** Rejected by ADR 0001: they run functions or a runtime that isn't Node, not one long-running process.
- **Fly Managed Postgres.** Rejected: at $4 a month it pushes the total past the $5 allowance. **Supabase free** pauses after a quiet week.
- **Postmark, SES, MailerSend, Brevo.** Rejected: each adds a human account review or a hard 100-a-month cap. Resend needs only DNS verification.
- **Fly's generated Dockerfile or a buildpack.** Rejected: neither understands the Nx workspace. The multi-stage `Dockerfile` and `fly.toml` are hand-written and checked in.

## Consequences

- **Migrations run as Fly's `release_command`**, once per deploy on a temporary Machine built from the new image, over Neon's direct connection string. A failure aborts the deploy and the old version keeps serving. This replaces the `migrate && serve` start command from ADR 0005. Because a revert or `fly deploy --image <previous>` can put the previous app version back on top of a newer schema, every migration must stay compatible with the previous app version: add in one deploy, rename or drop in a later one.
- **The app connects through Neon's pooler** (PgBouncer in transaction mode) with a `pg` pool of at most 5 and an idle timeout under Neon's 5-minute suspend. Session state doesn't survive the pooler, so the code never uses `SET`, advisory locks or `LISTEN`. Row locks inside a transaction are unaffected.
- **The client IP for sign-in limits is `Fly-Client-IP`**, falling back to the socket address when the header is absent (locally and in tests). `X-Forwarded-For` is never read, because Fly keeps a client-supplied value. The lookup sits behind one function, so another host changes only that function.
- **Deploys drop requests for a few seconds**: with one Machine, a rolling deploy can't overlap old and new. A `GET /healthz` check that doesn't touch the database gates each deploy, so a suspended Neon compute can't fail it.
- **Secrets live only in Fly** (`fly secrets set`): the database URLs, the Resend key and the Better Auth secret. GitHub holds only a deploy token scoped to this app. Non-secret config is in `fly.toml`.
- **Going over Fly's $5 allowance bills the whole month** at standard rates, so the Machine size and count are fixed in `fly.toml`.
