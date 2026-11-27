---
status: accepted
---

# Sign-in by emailed link, with Better Auth in our own process

A user signs in only by a link sent to their email address; there are no passwords and no separate registration. The mechanism is the Better Auth library with its magic-link plugin, running inside our one app, with users and sessions in our own database. Better Auth's `user` table is our user: its `name` column holds the display name, and members refer to its id directly.

We chose a library over a hosted service because every hosted option pulled another decision with it (a particular database, restricted email branding, or a low cap on free sign-in emails), and over hand-rolling because the part Better Auth still owns (creating tokens, spending each one exactly once under concurrent use, session rows and cookies) is the part where a subtle bug is a security hole.

## Considered Options

- **A hosted service** (Supabase Auth, Clerk, Stytch, Firebase, Auth0). Rejected: Supabase presumes its own Postgres, Clerk and Stytch restrict or charge for our own email branding, Firebase's free plan allows 5 sign-in emails a day, and Auth0 only works when the same browser requests and opens the link. Users would also live outside our database.
- **Hand-rolling** tokens and sessions. Rejected, narrowly: we write glue around Better Auth in four places anyway (below), and hand-rolling would add the security-critical core on top of that same glue.
- **Auth.js.** Rejected: first-class only on Next.js, and now folded into Better Auth, which it recommends for new projects.

## Consequences

Better Auth is used against its grain in four places, each deliberate. Do not "simplify" them back to its defaults.

- **The emailed link opens our own page, and only pressing its button spends the token.** Mail scanners open links before people do, and Better Auth's own verify endpoint spends the token on a `GET`. Our route action calls `auth.api.magicLinkVerify` instead, so Better Auth's emailed `url` is never sent.
- **We forward its cookies ourselves.** There is no React Router integration for server-side calls, so `Set-Cookie` headers are copied onto our responses at sign-in, sign-out and session renewal. Without the last, the session row slides but the browser's cookie does not.
- **The place to return to is stored in our own table, keyed by the token's hash,** never in the emailed URL. Better Auth cannot attach data to a token, and a destination in the URL would need open-redirect checks on every use.
- **Sending and sign-in limits are our own, with counters in the database.** Better Auth's limiter does not run for server-side calls and counts by IP address only; we also limit by email address, and counters in memory would reset whenever the host restarts the process.

Its defaults are overridden where they fall short: tokens are stored hashed (the default is plain text), a link lives 15 minutes (default 5), and a session lasts 30 days, renewed on use (default 7). Its `account` table exists but stays empty. Adding another way to sign in later is a Better Auth plugin, not a redesign.
