---
status: accepted
---

# One server-rendered app with a pure domain library

Bill Splitter is a single full-stack app, not a web client plus a separate API. It renders pages on the server with React Router v7 in framework mode, handles mutations as form posts, and runs as one long-running Node process. The browser is the only client, nothing in the planned slices is live or collaborative in real time, and the cost ceiling is a few dollars a month, so a second deployable and an API contract would cost more than they return.

The splitting and balance rules live in their own library of pure TypeScript that imports nothing from the framework, the database or the clock. It defines no repository interfaces: the app loads plain data, passes it in, and stores what comes back. Nx module-boundary lint rules enforce the direction (`app` may import `infra` and `domain`, `infra` may import `domain`, `domain` imports nothing), so the rules can be tested, including with property tests, without the web or a database.

## Considered Options

- **Web client plus a separate API.** Rejected: a second deployable, CORS and cross-origin sessions for the sign-in link, in exchange for a contract no second client needs.
- **Client-side SPA fetching JSON.** Rejected: balances are derived on read and nothing updates live, so client state would only be something to keep in sync.
- **Serverless or edge functions.** Rejected: narrows the database to ones with HTTP or pooled drivers and ties the framework to a platform adapter. The accepted cost of a long-running process is a slow first request after idle on a free tier.
- **Next.js.** Rejected: its defaults pull toward serverless and one vendor's platform, and its caching layers are more machinery than a handful of form-driven screens need.
- **Hono or Fastify with server-side templates.** Rejected: smallest runtime, but routing conventions, form handling and error boundaries would be hand-built.
- **SvelteKit.** Rejected: same shape as React Router, but no first-party Nx plugin.
- **Repository interfaces in the domain.** Rejected: with balances derived on read the domain needs no I/O, and the only second implementation would be a test fake that pure functions do not need.

## Consequences

- The host must run a long-running Node process, and the database can be anything with an ordinary Node driver.
- A future second client (a mobile app, a public API) would need an API added alongside the pages; the pure domain library is what makes that affordable.
