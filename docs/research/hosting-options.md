# Hosting options for a Node app and its database, deployed from `main`

Research for [#12](https://github.com/clnnn/bill-splitter/issues/12), on the map [#11](https://github.com/clnnn/bill-splitter/issues/11). This is a comparison, not a choice: the choice is a later ticket.

- **Checked**: 2026-11-27, against each provider's own docs and pricing pages (linked inline). Free tiers change often; re-check before committing.
- **Constraints** (from the map): TypeScript on Node in an Nx workspace; one public production environment deployed from `main`; a handful of real users; free or a few dollars a month; managed only (no self-managed servers, no Kubernetes).
- **Method note**: pages were read through a fetch tool that summarises them, so figures are as relayed from the cited page, not hand-copied. Anything I could not confirm on a first-party page is listed under [Not verified](#not-verified).

## Summary

| Option | Cheapest workable cost | App when idle | Database | Deploy from `main` | Custom domain | Biggest catch |
|---|---|---|---|---|---|---|
| **Render** (free web service) + external Postgres | $0 | Spins down after 15 min; ~1 min cold start | Own free Postgres expires after 30 days, so pair with Neon or Supabase; own paid Postgres from $6/mo | Native: linked branch, auto-deploy on commit or after CI passes | Yes, 2 included on Hobby workspace | One-minute cold start; SMTP ports blocked on free |
| **Render** (paid web service) | $7/mo app (+ $0 Neon, or + $6/mo Render Postgres) | Always on | As above | As above | Yes | $7 to $13/mo is above "a few dollars" |
| **Fly.io** | $0 within the $5/mo Free-plan allowance (card required); list price $2.19/mo for the smallest Machine | Always on, or auto-stop with ~2 s cold start | Managed Postgres Developer $4/mo, or external | GitHub Actions workflow running `flyctl deploy` (no native Git integration) | Yes, first 10 certificates free | Going over $5 bills the whole month at standard rates |
| **Railway** | $5/mo Hobby (includes $5 usage) | Always on (optional sleep) | Postgres as a service in the same project, billed as usage | Native: linked branch, optional "Wait for CI" | Hobby: 2. Free plan: none | The $0 Free plan has no custom domains and forced sleep, so realistically $5+/mo |
| **Vercel** (Hobby) + Marketplace Postgres | $0 | No server; functions on demand | Neon, Supabase, Prisma Postgres via Marketplace (Vercel Postgres is gone) | Native: every push to the production branch goes live | Yes, 50 per project | Terms limit Hobby to personal or non-commercial use; app runs as functions, not a long-lived process |
| **Cloudflare Workers** + D1 | $0 (Paid plan $5/mo) | No server; no idle sleep | D1 (SQLite), or external Postgres through Hyperdrive | Native: Workers Builds on the production branch | Yes, but the domain must be a Cloudflare zone | Not Node: a Workers runtime with Node compatibility; 10 ms CPU per request on Free |
| **Google Cloud Run** + external Postgres | $0 within free tier | Scales to zero | No always-free Cloud SQL (30-day trial only); pair with Neon | Not verified today | Not verified today | More setup than the others; needs a billing account |
| **Koyeb** | Unclear | Free instance scales to zero after 1 hour | Free Postgres has only 5 hours of active time | Not verified today | 10 free | Docs and pricing page disagree on whether free compute still exists |

Database-only services to pair with a host:

| Database | Free tier | Idle behaviour | Catch |
|---|---|---|---|
| **Neon** (Postgres) | 1 GB per project, 100 CU-hours/month, 5 GB egress | Compute suspends after 5 min, resumes in a few hundred ms; projects are never paused or deleted for inactivity | Scale-to-zero cannot be disabled on Free; 6-hour restore window |
| **Supabase** (Postgres) | 500 MB per project, 2 projects, 5 GB egress | Project **paused** after 7 days of low activity; manual restore | No automatic backups on Free; a quiet week takes the app down |
| **Turso** (libSQL/SQLite) | 5 GB, 500M rows read and 10M rows written per month | Not stated | SQLite dialect, not Postgres |
| **Render Postgres** (free) | 1 GB | n/a | Expires 30 days after creation, deleted 14 days later; no backups |
| **Fly Managed Postgres** | None free; Developer plan $4/mo (covered by the $5 allowance) | Always on | Only one cluster and only this plan on the Free plan |
| **Cloudflare D1** (SQLite) | 5 GB, 5M rows read and 100k rows written per day | n/a | Used from Workers through a binding; SQLite dialect |

## Hosts in detail

### Render

- **Free web service**: spins down after 15 minutes without inbound traffic and takes about one minute to spin back up; 750 free instance hours per workspace per month; ephemeral filesystem; may be restarted at any time; outbound ports 25, 465 and 587 are blocked; custom domains and managed TLS are supported. Source: [Deploy for Free](https://render.com/docs/free).
- **Free Postgres**: one per workspace, 1 GB, **expires 30 days after creation**, deleted after a further 14-day grace period, no backups. Not usable as a production database. Source: [Deploy for Free](https://render.com/docs/free).
- **Paid**: cheapest web service is $7/mo (512 MB, fractional CPU); cheapest Postgres is $6/mo (256 MB RAM) plus storage at $0.30/GB. Hobby workspace is $0 and includes 5 GB bandwidth, 500 build minutes and 2 custom domains. Source: [Pricing](https://render.com/pricing).
- **Deploy from `main`**: link the service to a branch; auto-deploy is on by default ("On Commit"), or "After CI Checks Pass" to wait for GitHub checks. Zero-downtime redeploys unless a disk is attached. Source: [Deploys](https://render.com/docs/deploys).
- **Nx / monorepo**: root directory and build filters control what triggers a deploy. Source: [Monorepo Support](https://render.com/docs/monorepo-support).
- **Custom domain**: automatic TLS; 2 domains included on Hobby, $0.25/mo each beyond. Source: [Custom Domains](https://render.com/docs/custom-domains).
- **Bites**: the one-minute cold start is the first thing a returning user sees. Blocked SMTP ports mean sign-in email must go through a provider's HTTP API, not SMTP.

### Fly.io

- **Free plan**: $1 one-time credit without a card (expires after 7 days); with a card, **$5 of usage every month**. Limits: up to 3 Machines, volumes up to 5 GB, one Managed Postgres cluster on the $4 Developer plan, no dedicated IPv4. When the $5 is used up the organisation moves to Pay As You Go automatically and **the whole month is billed at standard rates from the first cent**. Source: [Free plan](https://docs.fly.io/about/free-plan).
- **Prices**: shared-cpu-1x 256 MB is $2.19/mo, 512 MB is $3.69/mo; volumes $0.15/GB-month; shared IPv4 and IPv6 free, dedicated IPv4 $2/mo; first 10 single-hostname certificates free; Managed Postgres from $4/mo (Developer, storage included). Source: [Pricing](https://docs.fly.io/about/pricing).
- **Idle behaviour**: optional. With `auto_stop_machines` the proxy stops idle Machines and restarts them on a request; "a cold start takes about 2 seconds or more for common apps", or a few hundred milliseconds with suspend. `min_machines_running = 1` keeps one up. Source: [Autostop/autostart](https://docs.fly.io/launch/autostop-autostart).
- **Deploy from `main`**: no native Git integration in the docs; the documented route is a GitHub Actions workflow with a `FLY_API_TOKEN` secret running `flyctl deploy --remote-only` on push. Source: [Continuous deployment with GitHub Actions](https://docs.fly.io/launch/continuous-deployment-with-github-actions).
- **Bites**: `fly deploy` creates 2 Machines per app by default (use `--ha=false`). Arithmetic from the prices above (my inference): one always-on 256 MB Machine ($2.19) plus Managed Postgres Developer ($4) is $6.19, which is over the $5 allowance; one Machine plus an external free database stays under it.

### Railway

- **Free plan** ($0): $1 of usage per month, no card; 1 project, up to 3 services, 0.5 GB RAM and 1 vCPU each, one 0.5 GB volume; **no custom domains**; services sleep after 5 to 10 minutes without outbound traffic; when the credit runs out services stop until the next cycle. New accounts first get a 30-day trial with a one-time $5. Sources: [Free trial](https://docs.railway.com/reference/pricing/free-trial), [Plans](https://docs.railway.com/reference/pricing/plans).
- **Hobby** ($5/mo): includes $5 of usage, overage charged on top; 2 custom domains; 5 GB volumes. Usage rates are roughly $10 per GB RAM per month, $20 per vCPU per month, $0.15 per GB volume per month, $0.05/GB egress. Source: [Pricing](https://railway.com/pricing).
- **Sleeping** (optional on paid plans): the first request to a sleeping service wakes it and may return a 502; an open database connection pool counts as activity and keeps the service awake. Source: [App sleeping](https://docs.railway.com/reference/app-sleeping).
- **Database**: Postgres runs as another service in the project with a volume, billed as usage; single-node on Free. Source: [Free trial](https://docs.railway.com/reference/pricing/free-trial).
- **Deploy from `main`**: services linked to a GitHub repo deploy on every push to the connected branch; "Wait for CI" holds the deploy until GitHub Actions pass. Source: [GitHub autodeploys](https://docs.railway.com/guides/github-autodeploys).
- **Bites**: a custom domain forces the $5 Hobby plan. Whether an always-on app plus Postgres fits inside the included $5 depends on memory actually used (my inference from the rates: 0.5 GB RAM held all month is about $5 on its own).

### Vercel (Hobby)

- **Free tier**: 1,000,000 function invocations, 4 CPU-hours of active CPU, 360 GB-hours provisioned memory, 100 GB fast data transfer per month; 100 deployments per day; 50 domains per project; 1 hour of runtime logs. If a limit is exceeded the feature is generally unavailable until 30 days have passed. Source: [Hobby plan](https://vercel.com/docs/plans/hobby).
- **Terms**: "You shall only use the Services under a Hobby plan for your personal or non-commercial use." Source: [Terms of Service, section 4](https://vercel.com/legal/terms). Pro is $20 per developer seat per month.
- **Node app shape**: Express, Hono, Fastify, NestJS, Koa and others deploy with zero configuration, but they run on Vercel Functions with Fluid compute: "there is no longer a server always running in the background." Source: [Backends on Vercel](https://vercel.com/docs/frameworks/backend).
- **Database**: Vercel Postgres no longer exists; Postgres comes from Marketplace integrations (Neon, Prisma Postgres, Supabase, AWS Aurora), which inject credentials as environment variables. Source: [Postgres on Vercel](https://vercel.com/docs/postgres).
- **Deploy from `main`**: the GitHub integration deploys every push; pushes to the production branch go live on the custom domains; every other branch and pull request gets a preview deployment by default. Source: [Vercel for GitHub](https://vercel.com/docs/git/vercel-for-github).
- **Nx**: importing an Nx monorepo configures build command and root directory automatically; needs Nx 17 or later. Source: [Deploying Nx to Vercel](https://vercel.com/docs/monorepos/nx).
- **Bites**: the application shape is constrained to request-scoped functions (no in-process background jobs or in-memory state); preview deployments are on by default while the map wants none; the non-commercial clause matters if the app ever takes money.

### Cloudflare Workers

- **Free plan**: 100,000 requests per day, **10 ms CPU time per request**, 128 MB memory, 50 subrequests per request. Over the daily limit, requests fail with error 1027 until midnight UTC. Paid plan is $5/mo with 10 million requests and up to 5 minutes CPU per request. Sources: [Pricing](https://developers.cloudflare.com/workers/platform/pricing/), [Limits](https://developers.cloudflare.com/workers/platform/limits/).
- **Not Node**: Workers "provides a subset of Node.js APIs", natively or as polyfills; `nodejs_compat` is on by default for compatibility dates from 2026-08-04. Some modules (`child_process`, `worker_threads`, `http2`) are non-functional stubs. Source: [Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/).
- **Database**: D1 (SQLite) free tier is 5 GB, 5 million rows read and 100,000 rows written per day. External Postgres is reachable through Hyperdrive, included on Free with 100,000 queries per day. Sources: [Pricing](https://developers.cloudflare.com/workers/platform/pricing/), [Hyperdrive pricing](https://developers.cloudflare.com/hyperdrive/platform/pricing/).
- **Deploy from `main`**: Workers Builds connects a GitHub repo; production-branch builds deploy, other branches create previews. Free: 3,000 build minutes per month, 1 concurrent build, 20-minute timeout. Sources: [Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/), [Builds limits](https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/).
- **Custom domain**: requires "an active Cloudflare zone" that you own. Source: [Custom Domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/).
- **Bites**: "TypeScript on Node" is only approximately met; libraries that assume a real Node process may not run. 10 ms CPU is tight for anything CPU-heavy such as password hashing. The domain's DNS zone would have to be on Cloudflare.

### Google Cloud Run

- **Free tier**: 2 million requests, 360,000 GB-seconds of memory and 180,000 vCPU-seconds per month, 1 GB outbound from North America. Source: [Free cloud features](https://docs.cloud.google.com/free/docs/free-cloud-features).
- **Database**: no always-free Cloud SQL, only a 30-day trial instance; Firestore (not SQL) has a free tier of 1 GiB, 50,000 reads and 20,000 writes per day. Source: same page. A relational database would have to come from Neon or similar.
- **Bites**: the most moving parts of any option here (project, billing account, IAM, build trigger). Deploy-from-GitHub setup and custom-domain mapping were not checked today.

### Koyeb

- The docs describe one free web service per organisation (512 MB, 0.1 vCPU, Frankfurt or Washington D.C.), which scales to zero after one hour without traffic and which Koyeb says should not be used for production, plus one free Postgres with **5 hours of active time** and 1 GB. A credit card is required. Sources: [Pricing FAQ](https://www.koyeb.com/docs/faqs/pricing), [Instances](https://www.koyeb.com/docs/reference/instances).
- The [pricing page](https://www.koyeb.com/pricing) read today lists only Pro ($29/mo), Scale ($299/mo) and Enterprise, with no $0 plan and no free compute instance; only "Free 5h" Postgres appears. The two first-party sources disagree, so the free compute tier should be treated as doubtful until confirmed by signing up.
- Cheapest paid instance in the docs: `eco-nano`, 256 MB, $1.61/mo.

## Databases in detail

- **Neon**: Free is $0 with no card: 1 GB storage per project, 100 CU-hours of compute per project per month, 5 GB egress, 6-hour restore window. Compute suspends after 5 minutes idle and this is fixed on Free; it "reactivates automatically within a few hundred milliseconds". Neon does not pause or delete Free projects for inactivity; when CU-hours or egress run out compute stops until the next period, and a full disk blocks writes. Paid (Launch) is pure usage: $0.106 per CU-hour, $0.35 per GB-month, no minimum. Sources: [Pricing](https://neon.com/pricing), [Scale to zero](https://neon.com/docs/introduction/scale-to-zero).
- **Supabase**: Free is 500 MB database per project, 2 active projects, 5 GB egress, no automatic backups. Projects with low activity over 7 days are **paused**; a paused project can be restored manually for up to 1 year. "Typically a few user requests to the database each day over the previous week is enough" to stay active. Pro is $25/mo. Sources: [Pricing](https://supabase.com/pricing), [Free project pausing](https://supabase.com/docs/guides/platform/free-project-pausing).
- **Turso**: Free is 5 GB, 500 million rows read and 10 million rows written per month, 100 databases, 1 day point-in-time restore. Cheapest paid plan $4.99/mo. Source: [Pricing](https://turso.tech/pricing).
- **Host-bundled**: Render free Postgres (expires after 30 days), Fly Managed Postgres Developer ($4/mo), Railway Postgres (usage-billed), Cloudflare D1: see each host above.

## What would bite a small always-available app

1. **Sleeping hosts.** Render free (15 min idle, ~1 min wake), Railway free (5 to 10 min idle, possible 502 on wake), Koyeb free (1 hour idle). A sign-in link opened from an email lands on a cold app. Fly's auto-stop is the mild version (~2 s). Vercel and Cloudflare have no idle sleep in this sense.
2. **Databases that disappear or pause.** Render free Postgres is deleted about six weeks after creation. Supabase pauses a project after a quiet week and needs a manual restore. Neon only suspends compute and wakes by itself.
3. **Free tiers that are really trials.** Railway's $5 trial lasts 30 days; Fly's no-card credit is $1 for 7 days; Cloud SQL is a 30-day trial.
4. **Custom domain gates.** Railway needs Hobby ($5/mo). Cloudflare needs the domain's zone on Cloudflare. The others include it free.
5. **Runtime shape.** Render, Fly, Railway, Koyeb and Cloud Run run an ordinary long-lived Node process (or container). Vercel runs the app as functions. Cloudflare is not Node.
6. **Email.** Render free blocks outbound SMTP ports, so sign-in email there must use an HTTP API.
7. **Billing surprises.** Fly bills the whole month once the $5 allowance is exceeded. Fly's recurring allowance and Koyeb need a card; Render free, Railway free and Neon free are documented as needing none.
8. **Deploy from `main`.** Native branch-linked deploys on Render, Railway, Vercel and Cloudflare; Fly needs a GitHub Actions workflow and a token secret. Vercel and Cloudflare also create preview deployments for other branches by default, which the map does not want.

## Not verified

- **Koyeb free compute**: docs say yes, pricing page shows none (see above).
- **Cloud Run**: the [pricing page](https://cloud.google.com/run/pricing) could not be read (truncated), so free-tier region restrictions, the GitHub deploy setup and custom-domain mapping are unconfirmed.
- **Render**: the base storage included with the $6/mo Postgres plan, and whether "After CI Checks Pass" is available on free services (the deploy docs do not restrict it).
- **Vercel**: how to turn preview deployments off entirely, and what counts as "commercial" beyond the one sentence in the terms (the fair-use page's examples did not render).
- **Cloudflare**: that Worker custom domains are available on the Free plan (the page lists no plan restriction, but does not say so).
- **Railway**: whether an always-on app plus Postgres really fits in Hobby's included $5; that is my arithmetic from published rates, not a provider figure.
- **Fly.io**: the $6.19 figure is likewise my arithmetic. Whether an existing organisation still gets the older under-$5 invoice waiver was not checked.
- **Neon**: how many wall-clock hours 100 CU-hours buys depends on the minimum compute size, which I did not confirm today.
- **Turso**: whether idle free databases are archived.
- Not surveyed: Heroku, Northflank, AWS (App Runner, Lambda), Azure, DigitalOcean App Platform, Deno Deploy, Netlify.
