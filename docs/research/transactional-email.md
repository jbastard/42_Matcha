# Transactional email in dev and on the VPS

Research for issue #7. Sources checked on 2026-11-24. Pricing and limits change often: re-check before relying on them.

## Question

How do we send the verification email (R-AUTH-03) and the password-reset email (R-AUTH-05) in local development and from the public VPS? Which Node mail libraries are acceptable under R-GEN-02?

## Short answer

- One code path everywhere: Nodemailer over SMTP, configured only through `.env` (R-GEN-11). Only the SMTP host, port and credentials change between environments.
- Dev: Mailpit in Docker Compose catches every message (SMTP 1025, web UI 8025) and has a REST API that integration tests can query.
- VPS: do not run our own MTA. Relay through a provider's free-tier SMTP (Brevo 300/day, Resend 100/day, Mailgun 100/day), from a domain we control with the provider's SPF/DKIM records published, over port 587 or 465.

## Library: Nodemailer

- Nodemailer is a sending library, not a framework, with zero runtime dependencies and MIT-0 license. SMTP is its default transport, configured with `host`, `port`, `secure`, `auth` or a `smtp://` URL; `transporter.verify()` checks the connection. Source: https://nodemailer.com/
- Latest npm release: 10.6.1, license `MIT-0`, no `dependencies` field. Source: `curl https://registry.npmjs.org/nodemailer/latest` (run 2026-11-24).
- Nodemailer 10 needs Node.js 20 or later and ships its own types: do not install `@types/nodemailer` with v10 (conflicting declarations). Source: https://nodemailer.com/
- R-GEN-02 fit: it does no routing, ORM, validation or account management, so it sits under R-GEN-03 ("any library needed"). Account verification tokens, reset tokens and expiry remain our own code, which R-GEN-02 requires anyway.
- Provider HTTP SDKs (e.g. Resend's) would also be libraries, but they tie the code to one vendor and do not work against Mailpit. SMTP keeps dev and prod identical. This is a judgment, not a sourced fact.

## Local development: Mailpit

- Image `axllent/mailpit`, default ports: SMTP 1025, web UI 8025. Source: https://mailpit.axllent.org/docs/install/docker/
- Latest release v1.34.0 (2026-11-17). Source: GitHub API `repos/axllent/mailpit/releases/latest`.
- REST API for tests: `GET /api/v1/messages`, `GET /api/v1/search`, `GET /api/v1/message/{ID}`, `DELETE /api/v1/messages`. Source: https://raw.githubusercontent.com/axllent/mailpit/master/server/ui/api/v1/swagger.json and https://mailpit.axllent.org/docs/api-v1/
- Shortcut views: `/view/latest.html`, `/view/latest.txt`, optionally `?query=<search>`, which make "click the link in the email" end-to-end tests simple. Cypress helper `cypress-mailpit` exists. Source: https://mailpit.axllent.org/docs/integration/
- A "Chaos" feature simulates SMTP failures, useful to test error handling. Source: https://mailpit.axllent.org/docs/integration/
- Mailpit never delivers mail, so dev cannot spam real addresses even with seeded profiles (R-GEN-06).

## VPS: why not send directly

- Hetzner Cloud blocks outbound ports 25 and 465 by default; unblocking needs an account older than a month with a paid invoice. Port 587 is open. Source: https://docs.hetzner.com/cloud/servers/faq/
- DigitalOcean blocks 25, 465 and 587 on Droplets and recommends a third-party email provider. Source: https://docs.digitalocean.com/support/why-is-smtp-blocked/ . A relay on a non-standard port (e.g. Resend 2465/2587, below) works around this.
- OVH VPS policy: not verified (help page redirected to a generic docs index).
- Gmail requirements for all senders: SPF or DKIM on the sending domain, PTR record for the sending IP matching forward DNS, TLS, spam rate under 0.3%, RFC 5322 format. Bulk senders (over 5,000/day) also need DMARC and alignment. DKIM keys at least 1024 bits, 2048 recommended. Source: https://support.google.com/a/answer/81126
- Self-hosting means owning PTR, IP reputation, DKIM signing and port unblocking. A relay provider covers PTR, IP reputation and TLS; we only publish its DNS records.

## VPS: free-tier SMTP relays

| Provider | Free limit | SMTP on free | Domains | Source |
|---|---|---|---|---|
| Brevo | 300 emails/day, after Brevo approves the account | Yes ("forever free plan with a sending limit of 300 emails per day") | not verified | https://www.brevo.com/pricing/ , https://www.brevo.com/free-smtp-server/ (FAQ text read via curl; WebFetch got only the title) |
| Resend | 3,000/month, 100/day | Yes | 3 custom domains | https://resend.com/pricing |
| Mailgun | 100/day | Yes | 1 custom sending domain | https://www.mailgun.com/pricing/ |
| SendGrid | not verified (pricing page redirects and renders client-side) | - | - | https://sendgrid.com/pricing/ |

- Resend SMTP: host `smtp.resend.com`, user `resend`, password = API key; ports 465/2465 (implicit TLS), 25/587/2587 (STARTTLS). Requires a verified domain. Source: https://resend.com/docs/send-with-smtp
- All three need a domain we control to publish SPF/DKIM records. Whether Brevo still allows sending from a single verified address without domain authentication is not verified; even if it does, mail From a `@gmail.com` address through a third-party relay fails Gmail's alignment and is likely to land in spam (inference from the Gmail guidelines above).
- Volume: an evaluation sends a handful of messages; 100/day is enough. The 500 seeded profiles (R-GEN-06) must never trigger real sends: seed with verified flags set directly in the database.

## Recommendation for the decision ticket (#18)

1. Nodemailer 10 with the SMTP transport, a single `sendMail` wrapper, config from `.env` (`SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `MAIL_FROM`), with `.env.example` committed without secrets (R-GEN-11, R-GEN-12).
2. Mailpit as a Docker Compose service for dev, and its API in integration tests for R-AUTH-03 and R-AUTH-05 links.
3. On the VPS: Resend or Brevo free SMTP over 587 (or 2587 if the host blocks 587), from a subdomain like `mail.<our-domain>` with the provider's SPF, DKIM and a `p=none` DMARC record. Pick by which provider the team can sign up for; Brevo has the highest daily cap but an approval step.
4. Do not self-host Postfix on the VPS.

## Not verified

- SendGrid's current free tier.
- OVH outbound SMTP policy.
- Brevo's number of sender domains on the free plan and single-sender mode.
- Gmail SMTP with an app password as a no-domain fallback (limits and current availability not checked).
