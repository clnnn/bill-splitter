# Emailed-link sign-in in TypeScript: options

Research for [#14](https://github.com/clnnn/bill-splitter/issues/14), on the map [#11](https://github.com/clnnn/bill-splitter/issues/11). Checked on 2026-11-27. This is a comparison, not a choice; the choice is a later ticket.

Constraints carried from the map: TypeScript on Node in an Nx workspace; web framework and database not yet chosen; free or near-free; one production environment sending real sign-in email from a domain the developer controls; a friend who opens a group invite link must land back on that group after signing in.

Every claim below cites the source that owns it. Anything I could not confirm from a primary source is listed under [Not verified](#not-verified) rather than stated as fact.

## The three shapes

1. **A library in our own process**: the app owns the user and session tables in its own database and sends the email through whichever provider is chosen. Candidates: Better Auth, Auth.js, `passport-magic-link`.
2. **A hosted auth service**: users and sessions live with the vendor; the app verifies a token or cookie. Candidates: Supabase Auth, Clerk, Stytch, Firebase Authentication, Auth0. WorkOS AuthKit is listed only to rule it out for links.
3. **Hand-rolled**: two tables and two routes, written against the security practices in the last section.

All three still need an email provider for a custom sending domain, except where a hosted service sends the mail itself (noted per option).

## Libraries

### Better Auth (magic link plugin)

- **Frameworks**: describes itself as a "framework-agnostic authentication (and authorization) framework for TypeScript" ([repo](https://github.com/better-auth/better-auth)). On plain Node/Express it mounts as a catch-all route through `toNodeHandler(auth)`, registered before body parsers ([Express integration](https://www.better-auth.com/docs/integrations/express)).
- **Databases**: built-in Kysely adapter for SQLite (including Cloudflare D1), PostgreSQL, MySQL and MSSQL; adapters for Drizzle, Prisma and MongoDB ([database docs](https://www.better-auth.com/docs/concepts/database)).
- **Where data lives**: in our database, in four core tables: `user`, `session`, `account`, `verification` ([database docs](https://www.better-auth.com/docs/concepts/database)). The docs also describe a database-less stateless mode, but whether the magic link plugin works in it is not documented (see Not verified).
- **Sessions after the link**: a `session_token` cookie backed by a row in the `session` table; default lifetime 7 days (`expiresIn` 604800 s), extended when used after `updateAge` (86400 s) ([session management](https://www.better-auth.com/docs/concepts/session-management), [options reference](https://www.better-auth.com/docs/reference/options)).
- **Link lifetime**: `expiresIn`, default 300 seconds ([magic link plugin](https://www.better-auth.com/docs/plugins/magic-link)).
- **Single use**: yes; verification "consumes the token atomically on the first attempt", and a retry fails with `INVALID_TOKEN` ([magic link plugin](https://www.better-auth.com/docs/plugins/magic-link)).
- **Token**: default is `generateRandomString(32, "a-z", "A-Z")`, i.e. 32 characters over 52 letters, about 182 bits ([plugin source, v1.7.2](https://app.unpkg.com/better-auth@1.7.2/files/dist/plugins/magic-link/index.mjs)). **Stored in plain text by default**: `storeToken` defaults to `"plain"`; `"hashed"` or a custom hasher is opt-in ([magic link plugin](https://www.better-auth.com/docs/plugins/magic-link)).
- **Return to the starting page**: `callbackURL` on the sign-in request is carried into the link and redirected to after verification; `newUserCallbackURL` and `errorCallbackURL` cover first-time users and failures ([magic link plugin](https://www.better-auth.com/docs/plugins/magic-link)). The verify endpoint runs an origin check on all three ([plugin source](https://app.unpkg.com/better-auth@1.7.2/files/dist/plugins/magic-link/index.mjs)); by default only the app's base URL is trusted ([options reference](https://www.better-auth.com/docs/reference/options)).
- **Email-scanner prefetch**: the verify endpoint is a `GET` that consumes the token and creates the session in the same request ([plugin source](https://app.unpkg.com/better-auth@1.7.2/files/dist/plugins/magic-link/index.mjs)), so a scanner that fetches the link spends it. No built-in mitigation is documented. Because `sendMagicLink` receives both `url` and `token`, the email can instead point at a page of ours that calls verify only on a button press.
- **Rate limiting**: `{ window: 60, max: 5 }` on the request and verify endpoints by default ([magic link plugin](https://www.better-auth.com/docs/plugins/magic-link)).
- **Email**: we implement `sendMagicLink`; any provider.
- **Cost**: MIT, free.
- **Maintenance**: `better-auth` 1.7.28 published 2026-11-24, repository pushed 2026-11-26, about 30.9k stars (npm registry and GitHub API, 2026-11-27).

### Auth.js (`@auth/core`, `next-auth`, `@auth/express`)

- **Frameworks**: built around Next.js. `next-auth` v5 is still in beta and v4 in maintenance mode; `@auth/express` and `@auth/sveltekit` are "experimental"; Fastify, Remix, Nuxt and Astro are open PRs ([integrations](https://authjs.dev/getting-started/integrations)).
- **Databases**: a database adapter is mandatory for email sign-in: "It is not possible to enable an email provider without using a database" ([email docs](https://authjs.dev/getting-started/authentication/email)). Adapters include Drizzle, Kysely, Prisma, PostgreSQL, MongoDB, Cloudflare D1, DynamoDB, Supabase, Upstash Redis and others ([integrations](https://authjs.dev/getting-started/integrations)).
- **Where data lives**: our database (users, accounts, sessions, verification tokens via the adapter) ([adapter reference](https://authjs.dev/reference/core/adapters)).
- **Sessions after the link**: database sessions by default once an adapter is configured, otherwise an encrypted JWT in the cookie; `maxAge` defaults to 30 days ([core reference](https://authjs.dev/reference/core)).
- **Link lifetime**: 24 hours by default, overridable with the provider's `maxAge` ([Nodemailer provider](https://authjs.dev/getting-started/providers/nodemailer), [`send-token.ts`](https://github.com/nextauthjs/next-auth/blob/main/packages/core/src/lib/actions/signin/send-token.ts)).
- **Single use**: yes; `useVerificationToken` returns the token "and deletes it so it can only be used once" ([adapter reference](https://authjs.dev/reference/core/adapters)).
- **Token**: `randomString(32)` is 32 random bytes from `crypto.getRandomValues`, hex-encoded (256 bits); the database stores `SHA-256(token + secret)`, not the token ([`send-token.ts`](https://github.com/nextauthjs/next-auth/blob/main/packages/core/src/lib/actions/signin/send-token.ts), [`web.ts`](https://github.com/nextauthjs/next-auth/blob/main/packages/core/src/lib/utils/web.ts)).
- **Return to the starting page**: `callbackUrl` is embedded in the emailed link ([`send-token.ts`](https://github.com/nextauthjs/next-auth/blob/main/packages/core/src/lib/actions/signin/send-token.ts)); the default `redirect` callback allows only URLs on the same host ([core reference](https://authjs.dev/reference/core)).
- **Email-scanner prefetch**: acknowledged in the project's own guide: Outlook SafeLinks sends a `HEAD` request to each link, "in effect using it up"; the suggested fix is to answer `HEAD` with 200 before the auth handler runs ([guide](https://next-auth.js.org/tutorials/avoid-corporate-link-checking-email-provider)). That does not cover scanners that issue `GET`.
- **Email**: built-in providers for Resend, SendGrid, Postmark, Mailgun, Loops, Forward Email and Nodemailer/SMTP ([email docs](https://authjs.dev/getting-started/authentication/email)).
- **Cost**: ISC, free.
- **Maintenance**: the README says "Auth js is now part of Better Auth" and "We recommend new projects to start with Better Auth unless there are some very specific feature gaps" ([repo](https://github.com/nextauthjs/next-auth)). Last npm releases 2026-07-20 (`next-auth` 4.24.15 and 5.0.0-beta.32, `@auth/core` 0.41.3, `@auth/express` 0.12.3); repository last pushed 2026-07-22 (npm registry and GitHub API, 2026-11-27).

### `passport-magic-link` (Passport strategy)

- **Frameworks**: Passport, so Express-style middleware. `passport` itself last released 0.7.0 on 2023-11-27 (npm registry).
- **Databases**: none assumed; we supply `verifyUser` and `sendToken`. Used tokens are tracked in a `storage` that "Defaults to MemoryStorage", which is lost on restart and not shared between instances; a custom store (a Redis example is shown) is needed in production ([repo](https://github.com/vinialbano/passport-magic-link)).
- **Where data lives**: wherever we put it; the library holds no user table.
- **Sessions after the link**: not provided. Passport hands the user to whatever session middleware we add.
- **Link lifetime**: `ttl`, "defaults to 10 minutes" ([repo](https://github.com/vinialbano/passport-magic-link)).
- **Single use**: `allowReuse` "Defaults to `false`" ([repo](https://github.com/vinialbano/passport-magic-link)).
- **Token**: a token signed with `secret` (a JWT by default) rather than a random value looked up in a database ([repo](https://github.com/vinialbano/passport-magic-link)).
- **Return to the starting page**: nothing built in; ours to carry.
- **Email-scanner prefetch**: nothing documented.
- **Cost**: MIT, free.
- **Maintenance**: 4.1.0 published 2026-08-26, repository pushed 2026-10-18, but a single-maintainer project with 63 stars (npm registry and GitHub API, 2026-11-27).

### Lucia: not an option as a library

The `lucia` npm package is deprecated ("Lucia was deprecated on March 2025"); the repository now offers a single-file session implementation and points to its author's auth book ([repo](https://github.com/lucia-auth/lucia), npm registry). It is relevant only as reference material for hand-rolling.

## Hosted services

### Supabase Auth

- **Frameworks**: none assumed; called through `@supabase/supabase-js` (2.119.2, published 2026-11-19, npm registry).
- **Databases**: assumes a Supabase project. Sessions are rows in `auth.sessions` in that project's Postgres ([sessions](https://supabase.com/docs/guides/auth/sessions)). Choosing it leans on the still-open database decision.
- **Sessions after the link**: a JWT access token (default 1 hour) plus a refresh token that "never expire but can only be used once"; on a server-rendered app we store them in cookies ourselves ([sessions](https://supabase.com/docs/guides/auth/sessions)).
- **Link lifetime**: "they expire after 1 hour" by default; more than one day is "strongly discouraged" ([passwordless email](https://supabase.com/docs/guides/auth/auth-email-passwordless)).
- **Single use**: "one-time use only" ([passwordless email](https://supabase.com/docs/guides/auth/auth-email-passwordless)).
- **Return to the starting page**: `emailRedirectTo`, which must match the Site URL or the project's redirect URL allow list ([passwordless email](https://supabase.com/docs/guides/auth/auth-email-passwordless)).
- **Email-scanner prefetch**: documented as the most common cause of "expired" links; the documented workarounds are to email a code instead, or to link to a page of ours where a button completes the confirmation ([email templates](https://supabase.com/docs/guides/auth/auth-email-templates), [troubleshooting](https://supabase.com/docs/guides/troubleshooting/otp-verification-failures-token-has-expired-or-otp_expired-errors-5ee4d0)).
- **Rate limiting**: one link per user per 60 seconds; 30 sign-in requests per 5 minutes per IP ([rate limits](https://supabase.com/docs/guides/auth/rate-limits)).
- **Email**: the built-in sender is unusable for real users: it "will refuse to deliver messages to addresses that are not part of the project's team", is capped at 2 messages per hour and "is not meant for production use" ([SMTP](https://supabase.com/docs/guides/auth/auth-smtp)). A custom SMTP provider is required, so the email-provider decision is not avoided.
- **Cost**: Free plan includes 50,000 monthly active users and a 500 MB database, limited to 2 active projects, and "Free projects are paused after 1 week of inactivity"; Pro is from $25/month ([pricing](https://supabase.com/pricing)).

### Clerk

- **Frameworks**: SDK-driven; `@clerk/express` exists and is actively released (2.1.99, 2026-11-25, npm registry). A browser-side Clerk client is part of the design: it refreshes the session token about every 50 seconds ([how Clerk works](https://clerk.com/docs/guides/how-clerk-works/overview)).
- **Databases**: none of ours; users and sessions are held by Clerk. Our tables would reference Clerk user IDs.
- **Sessions after the link**: a 60-second JWT in a `__session` cookie on our domain (not `HttpOnly`), refreshed from a long-lived `HttpOnly` `__client` cookie on Clerk's domain ([how Clerk works](https://clerk.com/docs/guides/how-clerk-works/overview)). Session lifetime is "Fixed to 7 days" on the free plan ([pricing](https://clerk.com/pricing)).
- **Link lifetime**: "email links expire after 10 minutes"; not shown as configurable ([sign-up and sign-in options](https://clerk.com/docs/guides/configure/auth-strategies/sign-up-sign-in-options)).
- **Same-device rule**: a "Require the same device and browser" setting exists, described as enabled by default; a mismatch surfaces as `client_mismatch` ([email links custom flow](https://clerk.com/docs/js-frontend/guides/development/custom-flows/authentication/email-links.md)). This bears on prefetch (a scanner is a different client) and on friends who open mail on another device.
- **Return to the starting page**: a redirect URL is supplied when the link is requested ([email links custom flow](https://clerk.com/docs/js-frontend/guides/development/custom-flows/authentication/email-links.md)).
- **Email**: sent by Clerk. Custom email templates and removing Clerk branding are not on the free plan ([pricing](https://clerk.com/pricing)).
- **Cost**: free up to 50,000 monthly retained users per app; Pro $25/month ([pricing](https://clerk.com/pricing)).

### Stytch

- **Frameworks**: none assumed; backend API through the `stytch` Node SDK (14.3.0, 2026-10-27, npm registry).
- **Databases**: none of ours; users and sessions are held by Stytch.
- **Sessions after the link**: Stytch will "verify the token and issue a Session for the User" ([magic links overview](https://stytch.com/docs/consumer-auth/authentication/magic-links/overview.md)); we hold its session token in our own cookie.
- **Single use**: "Each Magic Link that is sent to the user is one-time use" ([overview](https://stytch.com/docs/consumer-auth/authentication/magic-links/overview.md)).
- **Return to the starting page**: the link target is `login_magic_link_url` / `signup_magic_link_url`, which must be on the project's redirect URL allow list ([API guide](https://stytch.com/docs/guides/magic-links/email-magic-links/api)).
- **Email-scanner prefetch**: the only option here with a documented built-in defence: "Most security scanners will inspect and click on any links present in emails", and when a scanner click is detected "we do not treat the click as an authentication attempt" ([overview](https://stytch.com/docs/consumer-auth/authentication/magic-links/overview.md)).
- **Email**: sent by Stytch. Full email customisation and brand removal is a one-time $99 add-on ([pricing](https://stytch.com/pricing)).
- **Cost**: 10,000 monthly active users free ([pricing](https://stytch.com/pricing)).

### Firebase Authentication (email link)

- **Frameworks**: the documented flow completes in the browser with the Firebase client SDK (`isSignInWithEmailLink`, `signInWithEmailLink`), with `handleCodeInApp` required to be true ([email link auth](https://firebase.google.com/docs/auth/web/email-link-auth)). A server-rendered app would add `firebase-admin` (14.5.1, 2026-11-12, npm registry) on the server.
- **Databases**: none of ours; users are held by Firebase.
- **Return to the starting page**: the continue URL in `actionCodeSettings.url` carries "any additional state", and its domain must be an authorised domain ([email link auth](https://firebase.google.com/docs/auth/web/email-link-auth)).
- **Cross-device**: if the link is opened on another device the user must type their email again, "To prevent session fixation attacks" ([email link auth](https://firebase.google.com/docs/auth/web/email-link-auth)).
- **Email**: Firebase sends it, or the Admin SDK generates the link with `generateSignInWithEmailLink` for us to send through our own provider ([admin email action links](https://firebase.google.com/docs/auth/admin/email-action-links)).
- **Cost**: the free Spark plan allows only **5 email-link sign-in emails per day**; the pay-as-you-go Blaze plan allows 25,000 per day and needs a billing account ([limits](https://firebase.google.com/docs/auth/limits)).

### Auth0

- Magic links "are not supported for Universal Login"; they work only with Classic Login, where "both the initial request and its response must take place in the same browser or the transaction will fail" ([magic link docs](https://auth0.com/docs/authenticate/passwordless/authentication-methods/email-magic-link)). A friend who requests the link on a laptop and opens the mail on a phone fails.
- Free plan: up to 25,000 monthly active users with passwordless included; the cheapest paid plan is from $35/month ([pricing](https://auth0.com/pricing)).

### WorkOS AuthKit: not an emailed link

Magic Auth is "a unique, six digit one-time-use code sent to their email inbox" that expires after 10 minutes, entered on WorkOS's hosted page ([Magic Auth](https://workos.com/docs/authkit/magic-auth)). It is free for the first 1 million monthly active users, with a custom domain at $99/month ([pricing](https://workos.com/pricing)). It answers a different question (emailed code) and is listed so it is not mistaken for a link option.

## Hand-rolling

**What it is**: a sign-in token table (token hash, email, expiry, optional return path), a session table, a route that issues a token and sends the email, a route that consumes it and sets a cookie.

- **Frameworks and databases**: assumes nothing; fits whatever is chosen. Needs only Node's `crypto`.
- **Where data lives**: our database.
- **Sessions**: ours to build, to the session requirements below. The deprecated Lucia repository publishes a single-file session implementation as a starting point ([repo](https://github.com/lucia-auth/lucia)).
- **Lifetime, single use, return path, prefetch handling**: all ours to decide, which means the expired and reused-link states on the map are designed rather than inherited.
- **Cost**: none.
- **Maintenance**: ours. No upstream security fixes; every practice in the next section is our responsibility and needs tests.

## Security practices any option must meet

Token:

- **Generation**: a "cryptographically secure random number generator" ([OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)). `Math.random`-style generators "should be avoided" ([Copenhagen Book, server-side tokens](https://copenhagen.pilcrowonpaper.com/server-side-tokens)).
- **Entropy**: OWASP says only "Long enough to protect against brute-force attacks". The Copenhagen Book gives a number: "at least 112 bits of entropy (120-256 is a good range)" ([server-side tokens](https://copenhagen.pilcrowonpaper.com/server-side-tokens)).
- **Storage**: "Stored in a secure manner" (OWASP). The Copenhagen Book: such tokens "should be hashed with SHA-256", which is sufficient because "the token is sufficiently long and random" ([server-side tokens](https://copenhagen.pilcrowonpaper.com/server-side-tokens)). A database leak must not yield usable links.
- **Single use**: "Invalidated after they have been used" (OWASP); "immediately deleted from storage after verification" ([Copenhagen Book, email verification](https://copenhagen.pilcrowonpaper.com/email-verification)). Consumption has to be atomic so two concurrent requests cannot both succeed.
- **Expiry**: "expire after an appropriate period" (OWASP). The Copenhagen Book says at least 15 minutes for email links. The defaults seen above range from 5 minutes (Better Auth) to 24 hours (Auth.js).

Link and request handling:

- **Build the link from configuration, not the request**: "Don't rely on the Host header while creating the reset URLs"; hard-code or allow-list the base URL (OWASP Forgot Password).
- **Do not leak the token by referrer**: set `Referrer-Policy` to `no-referrer` (OWASP) or `strict-origin` (Copenhagen Book) on any page whose URL contains the token.
- **No account enumeration**: "Return a consistent message for both existent and non-existent accounts", in consistent time (OWASP Forgot Password).
- **Rate limiting**: on the endpoint that sends email ("strict rate limiting", Copenhagen Book) and against token guessing ("such as rate limiting", OWASP).
- **Return destination is not an open redirect**: prefer a server-side mapping, or accept only local paths, or allow-list parsed URLs ([OWASP Unvalidated Redirects Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html)). For the invite flow this means the group path is validated or stored with the token, never taken blindly from a query string.
- **Email addresses**: normalise to lower case; do not try to fully validate by regex ([Copenhagen Book, email verification](https://copenhagen.pilcrowonpaper.com/email-verification)).

Email-scanner prefetch:

- Mail security products open links before the user does. Microsoft documents that with Safe Links "URLs are scanned prior to message delivery" and that URLs without a valid reputation "are detonated asynchronously in the background" ([Microsoft Learn](https://learn.microsoft.com/en-us/defender-office-365/safe-links-about)). Supabase, Auth.js and Stytch each document sign-in links being consumed this way (sources in their sections above).
- Consequence: a `GET` on the emailed URL must not consume the token. The documented patterns are (a) the link opens a page and the token is spent only when the user presses a button ([Supabase](https://supabase.com/docs/guides/auth/auth-email-templates)); (b) email a code to type instead (Supabase, WorkOS); (c) ignore `HEAD` requests, which is partial ([Auth.js guide](https://next-auth.js.org/tutorials/avoid-corporate-link-checking-email-provider)); (d) vendor-side scanner detection (Stytch); (e) binding the link to the requesting browser (Clerk, Auth0), which also blocks legitimate cross-device use.
- A confirm-button page also fits OWASP's "Don't automatically log the user in", written for password-reset links.

Session, once the link is used ([OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)):

- Session IDs from a CSPRNG with "at least 64 bits of entropy".
- Cookie is `Secure`, `HttpOnly`, and "`SameSite=Strict` (preferred) or `SameSite=Lax`". A link clicked in a mail client is a cross-site navigation, so an existing `Strict` cookie is not sent on that first request; Microsoft notes the same effect for Safe Links redirects ([Microsoft Learn](https://learn.microsoft.com/en-us/defender-office-365/safe-links-about)).
- A new session ID on sign-in ("renewed or regenerated ... after any privilege level change").
- An idle timeout and an absolute timeout.

## Comparison

| Option | Framework / database it assumes | Users and sessions live | Link lifetime (default) | Single use | Return to starting page | Scanner prefetch | Cost | Maintenance |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Better Auth | Any Node framework; SQL via Kysely, Drizzle, Prisma, or MongoDB | Our DB; cookie + `session` row, 7 days | 5 min, configurable | Yes, atomic | `callbackURL`, origin-checked | `GET` consumes; build own confirm page | Free (MIT) | Very active; released 2026-11-24 |
| Auth.js | Next.js first; Express adapter experimental; DB adapter mandatory | Our DB; DB session or JWT cookie, 30 days | 24 h, configurable | Yes | `callbackUrl`, same host only | Documented; `HEAD` workaround only | Free (ISC) | Now part of Better Auth; last release 2026-07-20; new projects pointed to Better Auth |
| `passport-magic-link` | Express + Passport; bring own store and sessions | Ours to build | 10 min, configurable | Yes, but default store is in-memory | Not provided | Not addressed | Free (MIT) | Single maintainer, 63 stars; released 2026-08-26 |
| Hand-rolled | Nothing | Our DB | Ours | Ours | Ours | Ours | Free | Ours entirely |
| Supabase Auth | Supabase project (its Postgres) | Supabase; JWT + rotating refresh token | 1 h, configurable | Yes | `emailRedirectTo`, allow-listed | Documented; confirm page or code | Free to 50k MAU; project pauses after 1 week idle; needs custom SMTP | Active |
| Clerk | Clerk browser client + backend SDK | Clerk; 60 s JWT cookie, 7-day session on free | 10 min | Not verified | Redirect URL on request | Same-device rule | Free to 50k users; Clerk-branded email on free | Active |
| Stytch | Backend API, any framework | Stytch | Not verified | Yes | Allow-listed link URL | Built-in scanner detection | Free to 10k MAU; $99 once for own branding | Active |
| Firebase Auth | Firebase client SDK in the browser | Firebase | Not verified | Not verified | Continue URL, authorised domains | Not verified | Free plan: 5 sign-in emails/day; otherwise billing account | Active |
| Auth0 | Classic Login only | Auth0 | Not verified | Not verified | Not verified | Same-browser only | Free to 25k MAU | Active |

Observations that fall out of the comparison, without choosing:

- The framework and database tickets constrain this one in different ways: Auth.js is only first-class on Next.js; Supabase Auth presumes Supabase as the database; Better Auth and hand-rolling fit anything.
- No library option handles scanner prefetch for us. Whatever is chosen in-process needs a confirm step designed in, which also shapes the "expired or reused link" states the map still has open.
- Every option can return the friend to the group; they differ in whether the destination is checked for us (Better Auth, Auth.js, Supabase, Stytch, Firebase) or by us (Passport, hand-rolled).
- Hosted services do not remove the email decision cleanly: Supabase needs our SMTP provider; Clerk and Stytch send from their side and charge for or restrict custom branding; Firebase's free plan caps sign-in email at 5 a day.
- Two defaults fall short of the practices above and would need changing if those options are picked: Better Auth stores tokens in plain text unless `storeToken: "hashed"` is set; Auth.js links live 24 hours.

## Not verified

- **Better Auth without a database**: the docs describe stateless sessions but do not say whether the magic link plugin, which writes a verification record, works in that mode.
- **Better Auth origin check**: I confirmed the verify endpoint applies `originCheck` to the callback URLs and that `disableOriginCheck` turns "URL validation for `callbackURL`" off, but did not read the `originCheck` implementation to confirm exactly how relative paths and trusted origins are matched.
- **Clerk**: whether an email link is single-use is not stated on the pages read; the "Require the same device and browser" default is stated as enabled in the custom-flow guide but not on the settings page; where user records are stored is inferred from the architecture page, not stated; how a return URL survives Clerk's prebuilt sign-in components was not checked.
- **Stytch**: default and maximum magic link lifetime, PKCE support and session duration bounds (the API reference page did not render for an unauthenticated fetch); the per-user price above the free tier is not shown on the pricing page.
- **Firebase**: link lifetime and single-use behaviour are not stated on the pages read; Identity Platform pricing beyond the Spark limits could not be read (page truncated).
- **Auth0**: link lifetime, single use and redirect handling are not stated on the magic link page.
- **Other scanners**: only Microsoft's own documentation was read for scanner behaviour; that other mail security products behave the same rests on the Supabase and Stytch docs, not on those vendors' own documentation.
- **Reading method**: pages were read through an automated fetch that summarises and quotes; quotations were not re-checked by eye against the rendered pages. The Copenhagen Book page on email verification carries a notice that it has been superseded by its author's newer auth book, which was not read.
- **Not surveyed**: Kinde, AWS Cognito, Descope, SuperTokens, Ory, Keycloak and other services or self-hosted servers. Self-hosted identity servers were left out because the map rules out self-managed servers.
- Pricing pages change; figures are as displayed on 2026-11-27.
