# Email providers for sign-in links

Research for [#13](https://github.com/clnnn/bill-splitter/issues/13), on the map [#11](https://github.com/clnnn/bill-splitter/issues/11).
Checked against provider docs and pricing pages on 2026-11-27. This is a comparison, not a choice.

**Question.** Which transactional email providers can deliver a sign-in email to arbitrary recipients from a domain the developer controls, within a free or near-free tier, and what does each require?

**Constraints (from the map).** Free or near-free (a few dollars a month at most), real sign-in emails to arbitrary recipients, sender on a domain the developer controls, TypeScript on Node, a handful of real users. Expected volume is therefore tens of emails a month, a few per day at peak.

## Short answer

Seven providers were checked. Six have a standing free or pay-per-use tier that fits the cost ceiling; SendGrid no longer does (60-day trial only). All six require a verified domain or sender before mail goes to arbitrary recipients, and all six are callable from Node through an npm package or plain HTTPS. They differ mainly in three things: the free volume, whether a human review gates sending to strangers, and how many DNS records are needed.

| Provider | Free tier | Gate before arbitrary recipients | DNS records for the domain | Node |
| --- | --- | --- | --- | --- |
| Resend | 3,000/month, 100/day, 3 domains | Domain verification only | MX + SPF TXT on `send.` subdomain, DKIM TXT; DMARC optional | `resend` SDK, REST, SMTP |
| Postmark | 100/month, no expiry, no overage | Domain verification **and** manual account approval (under 24h on weekdays) | DKIM TXT + Return-Path CNAME; DMARC recommended | `postmark` SDK, REST, SMTP |
| Amazon SES | No standing free quota; $0.10 per 1,000 (new accounts get AWS credits) | Domain verification **and** production-access request (initial response within 96h) | 3 DKIM CNAMEs; custom MAIL FROM (MX + SPF TXT) optional; DMARC optional | `@aws-sdk/client-sesv2` |
| Mailgun | 100/day, 1 custom domain, 1-day logs | Domain verification (sandbox domain is limited to 5 authorized recipients) | SPF TXT, DKIM TXT, MX x2, tracking CNAME, all marked required | `mailgun.js` SDK, REST |
| MailerSend | 500/month per pricing page (see conflict below); card required at signup | Domain verification **and** account approval (2 recipients until approved) | SPF TXT, 2 DKIM CNAMEs, Return-Path CNAME; DMARC optional | `mailersend` SDK, REST, SMTP |
| Brevo | 300/day "once we approve your account" | Registered and verified sender; account approval | Not verified (help page blocked) | `@getbrevo/brevo` SDK, REST, SMTP |
| Mailjet | 6,000/month, 200/day | Validated sender or domain | Not verified (help page blocked) | `node-mailjet` SDK, REST v3.1, SMTP |
| SendGrid | None: free trial, 100/day for 60 days | n/a | n/a | n/a |

## Per provider

### Resend

- **Free tier.** 3,000 emails/month, "limited to 100 emails per day", 3 domains, 30-day data retention, 1 webhook endpoint. Cheapest paid plan: Pro, $20/month for 50,000 emails. [pricing](https://resend.com/pricing)
- **Domain verification.** Records live on a `send` subdomain plus a DKIM selector: an MX record at `send` (priority 10), an SPF TXT at `send` (the guide's example value is `v=spf1 include:amazonses.com ~all`), and a DKIM TXT at `resend._domainkey`. An extra MX at `inbound` is only for receiving. [Cloudflare guide](https://resend.com/docs/dashboard/domains/cloudflare)
- **DMARC.** Optional, presented as a step after verification; suggested starting record is `v=DMARC1; p=none; rua=mailto:...` at `_dmarc`. [DMARC guide](https://resend.com/docs/dashboard/domains/dmarc)
- **Before verification.** The shared `resend.dev` sender works for testing only: "You can only send testing emails to your own email address". Any other recipient returns 403 until a domain is verified. [403 article](https://resend.com/docs/knowledge-base/403-error-resend-dev-domain), [domains intro](https://resend.com/docs/dashboard/domains/introduction)
- **Node.** `resend` npm package (ships type declarations), `resend.emails.send({ from, to, subject, html })` returning `{ data, error }`. [Node guide](https://resend.com/docs/send-with-nodejs). SMTP at `smtp.resend.com`, ports 25/465/587/2465/2587, username `resend`, password is the API key. [SMTP guide](https://resend.com/docs/send-with-smtp)
- **Limits.** "The default maximum rate limit is 10 requests per second per team." [API intro](https://resend.com/docs/api-reference/introduction)
- **Caveats.** No human review step was found in the docs. The example SPF include points at Amazon SES, which suggests mail leaves through SES infrastructure (an inference from that record, not a statement in the docs).

### Postmark

- **Free tier.** 100 emails/month, "No overages allowed in this plan", "it doesn't expire". Cheapest paid plan: Basic, $15/month for 10,000. [pricing](https://postmarkapp.com/pricing)
- **Domain verification.** A DKIM TXT record (required for domain verification) and a custom Return-Path CNAME (value `pm.mtasv.net`; the default host is given as `pm-bounces` in one article and `pm_bounces` in another, so copy it from the dashboard). No separate SPF record is asked for: the Return-Path record "allows for messages sent by Postmark to pass SPF authentication". DMARC recommended, starting at `p=none`. A single address can instead be verified as a Sender Signature. [verify a domain](https://postmarkapp.com/support/article/1046-how-do-i-verify-a-domain), [getting started](https://postmarkapp.com/support/article/1002-getting-started-with-postmark)
- **Before approval.** New accounts are in test mode: "you won't be able to send to any email address outside the domains you've added to your account and verified". Approval is a manual review requested by the account owner, "completed in less than 24 hours on weekdays". [approval process](https://postmarkapp.com/support/article/1084-how-does-the-account-approval-process-work)
- **Node.** `postmark` npm package (ships types), `ServerClient.sendEmail`. REST is `POST https://api.postmarkapp.com/email` with `X-Postmark-Server-Token`. [API guide](https://postmarkapp.com/developer/user-guide/send-email-with-api). SMTP at `smtp.postmarkapp.com`, ports 25/2525/587. [SMTP guide](https://postmarkapp.com/developer/user-guide/send-email-with-smtp)
- **Caveats.** Transactional and broadcast mail go through separate message streams, with transactional as the default. 100/month with no overage is a hard ceiling: once hit, sign-in stops until the month rolls over or the plan becomes $15/month, which is above the cost ceiling.

### Amazon SES

- **Free tier.** The pricing page lists no standing free email quota. New AWS customers get "up to $200 in AWS Free Tier credits" usable on SES. Pay-as-you-go outbound is "$0.10 / 1,000 emails", so this app's volume costs about a cent a month or less. [pricing](https://aws.amazon.com/ses/pricing/)
- **Domain verification.** Easy DKIM: three CNAME records, and verification completes when SES detects them (DNS changes "can take up to 72 hours to propagate"). A custom MAIL FROM subdomain (MX + SPF TXT) is optional. Identities are per AWS Region. [creating identities](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- **DMARC.** Passes through DKIM alignment once Easy DKIM is set up. SPF alignment additionally needs the custom MAIL FROM domain. [DMARC guide](https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dmarc.html)
- **Before production access.** Every new account starts in the sandbox: "You can only send mail to verified email addresses and domains", "a maximum of 200 messages per 24-hour period", "1 message per second". Leaving it means a request stating mail type and website URL; "The AWS Support team provides an initial response to your request within 96 hours", and the page does not promise approval. [production access](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html)
- **Node.** AWS SDK for JavaScript v3; the current SES API is v2, published as `@aws-sdk/client-sesv2` (ships types). Needs an AWS access key pair and a Region rather than a single API key. [SDK guide](https://docs.aws.amazon.com/ses/latest/dg/send-an-email-using-sdk-programmatically.html)
- **Caveats.** The most setup of the group: AWS account, credentials, Region choice, sandbox exit. The production-access form asks for a website URL, so the app may need to be deployed before the request.

### Mailgun

- **Free tier.** "100 emails/day included", 1 custom sending domain, 1 day of log retention, 1 API key, 1 user. Cheapest paid plan: Basic, from $15/month for 10,000. [pricing](https://www.mailgun.com/pricing/)
- **Domain verification.** Four record types, all marked required: SPF TXT (`v=spf1 include:mailgun.org ~all`), DKIM TXT, a CNAME to `mailgun.org` for open/click tracking, and two MX records (`mxa.mailgun.org`, `mxb.mailgun.org`) for receiving. DMARC is not mentioned on the page. [verify a domain](https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-verify)
- **Before verification.** "Sandbox domains can only send to authorized recipients", up to 5. [sandbox domain](https://documentation.mailgun.com/docs/mailgun/user-manual/domains/domains-sandbox)
- **Node.** `mailgun.js` npm package, `mg.messages.create(domain, { from, to, subject, text })`. [Node SDK](https://documentation.mailgun.com/docs/mailgun/sdk/nodejs_sdk)
- **Caveats.** Publishing MX records on the sending name conflicts with an existing mailbox on that name, which is why a subdomain is the usual place for them (an inference from the record list, not a quoted recommendation). One-day log retention leaves little to debug a missed sign-in email with.

### MailerSend

- **Free tier.** Pricing page: 500 emails/month, 100 API requests/day, 1 sending domain, 1 API token, 1-day activity retention, and a credit card is required at signup ("You won't be charged for anything on the Free plan"). Cheapest paid plan: Hobby, $7/month ($5.60 billed yearly) for 5,000. [pricing](https://www.mailersend.com/pricing)
- **Conflict between first-party sources.** The help centre describes a "Sandbox mode" of "100 emails a month, with a cap of 100 API requests a day" with "no payment method needed" for approval. [sandbox article](https://www.mailersend.com/help/sandbox-mode-and-the-professional-trial). Whether an approved free account gets 100 or 500 a month, and whether a card is needed, was not resolved.
- **Domain verification.** SPF TXT at the root (merged into any existing SPF record), two DKIM CNAMEs ("missing either one will keep your domain from verifying"), and a Return-Path CNAME (default host `mta`, pointing to `mailersend.net`). A tracking CNAME is optional. [verify a domain](https://www.mailersend.com/help/how-to-verify-and-authenticate-a-sending-domain)
- **Before approval.** "You can send to any 2 recipients in Sandbox mode, until your account is approved." A trial domain is available immediately. [sandbox article](https://www.mailersend.com/help/sandbox-mode-and-the-professional-trial)
- **Node.** `mailersend` npm package (ships types). REST base `https://api.mailersend.com/v1/`; SMTP relay also offered. [developer docs](https://developers.mailersend.com/)

### Brevo

- **Free tier.** "Free forever, no credit card needed"; "Once we approve your account for sending, you can start sending up to 300 emails per day." All plans include "transactional email features, including RESTful APIs, SMTP". Cheapest paid plan: Starter, $9/month from 5,000 emails/month. [pricing](https://www.brevo.com/pricing/)
- **Domain verification.** The sender must be "registered and verified". The domain-authentication help page returned 403 to automated fetches, so the exact DNS records are not verified here. [send guide](https://developers.brevo.com/docs/send-a-transactional-email)
- **Node.** `@getbrevo/brevo` npm package (ships types). REST is `POST https://api.brevo.com/v3/smtp/email` with an `api-key` header. SMTP relay uses a separate SMTP key on port 587, 2525 or 465. [SMTP guide](https://developers.brevo.com/docs/smtp-integration)
- **Caveats.** The pricing page leads with marketing features, and transactional is one capability among several. Whether free-plan transactional mail carries Brevo branding was not verified.

### Mailjet

- **Free tier.** 6,000 emails/month, 200/day, "APIs, SMTP Relay, Webhooks" included. Cheapest paid tier shown: Starter, $8.10/month billed annually for 8,000/month. [pricing](https://www.mailjet.com/pricing/)
- **Domain verification.** "Verify the domain or subdomain you will use for sending emails"; the From address "must be a previously validated and active sender". The SPF/DKIM help page returned 403, so the exact records are not verified here. [getting started](https://dev.mailjet.com/email/guides/getting-started/), [Send API v3.1](https://dev.mailjet.com/email/guides/send-api-v31/)
- **Node.** `node-mailjet` npm package (ships types). REST is `POST https://api.mailjet.com/v3.1/send` with basic auth using a public/private key pair.
- **Caveats.** The pricing page lists "No Mailjet Logo" starting at a paid tier, which implies free-tier mail may carry branding; whether that applies to API-sent transactional mail was not verified.

### SendGrid (Twilio)

- No standing free plan: "Free Trial", "$0/mo for 60 days", "100 emails/day for 60 days". Paid starts at Essentials, "starting at $19.95/mo". [pricing](https://www.twilio.com/en-us/products/email-api/pricing)
- Outside the cost ceiling after the trial, so not examined further.

## Deliverability notes for one-off sign-in emails

- **Mailbox provider baselines.** Gmail requires of all senders: "Set up SPF or DKIM email authentication for your sending domains", TLS, valid forward and reverse DNS, and spam rates "below 0.3%". DMARC, From-domain alignment and one-click unsubscribe are additional requirements only for senders of 5,000+ messages a day. [Gmail sender guidelines](https://support.google.com/a/answer/81126). Yahoo is the same shape: "Implement SPF or DKIM at a minimum" for everyone, and it "strongly urges all senders to publish a DMARC policy". [Yahoo sender best practices](https://senders.yahooinc.com/best-practices/)
- **What that means here.** For the five providers whose DNS setup was read (Resend, Postmark, SES, Mailgun, MailerSend), the standard domain setup includes DKIM or SPF and so meets the mandatory bar at this volume. A `_dmarc` TXT record at `p=none` is optional in each of them and costs one DNS record.
- **Review gates trade against speed.** Postmark, SES, MailerSend and Brevo put a review or approval between signup and sending to strangers; for Resend, Mailgun and Mailjet the pages read mention only DNS or sender verification. The gates add up to a day (Postmark) or up to four days for a first response (SES) before the first real sign-in email.
- **Hard ceilings matter more than volume.** A sign-in email that is not sent locks a user out. Postmark's 100/month with no overage is the cap a handful of users could plausibly meet; the daily caps (100 at Resend and Mailgun, 200 at Mailjet, 300 at Brevo) are well above expected use but are still hard stops.
- **Tracking.** Mailgun and MailerSend include a tracking CNAME in domain setup (required at Mailgun, optional at MailerSend). Whether click tracking, which rewrites links, is on by default at each provider was not checked, and it matters for a sign-in link.
- **Not found in primary sources.** No provider page read here publishes inbox-placement figures, says whether free tiers send from shared IPs, or quantifies how a domain with no sending history fares. This document therefore does not rank the providers on deliverability.

## Not verified

- **Mailgun, custom domain on the free plan.** The pricing page lists 1 custom domain on Free, and the sandbox doc limits only the sandbox domain. The help-centre article that states custom domains may send to anyone, and that free accounts without a card on file are restricted to authorized recipients, returned 403 and was seen only through a search summary.
- **MailerSend free volume and card requirement.** Pricing page and help centre disagree (500/month with a card, against 100/month with no payment method).
- **Brevo and Mailjet DNS records.** Both help-centre pages returned 403.
- **Branding on free-tier mail** for Brevo and Mailjet.
- **SMTP for SES and Mailgun.** No page fetched in this session stated whether either offers an SMTP interface.
- **`mailgun.js` types.** The npm metadata for the latest version declares no top-level `types` field; the other six packages do. Whether types ship another way was not confirmed.
- **Approval outcomes.** Postmark, SES, MailerSend and Brevo document that review happens, not what gets a small hobby app approved or rejected.
- **Click-tracking defaults** per provider.
- **Prices** are as displayed to an automated fetch on 2026-11-27 and may differ by region or change.

## Where this file lives

The repo had no convention for research notes, so this sits in `docs/research/`. The branch `research/email-providers` is throwaway and is not meant to be merged.
