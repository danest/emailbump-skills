---
name: email-best-practices
description: Email deliverability and compliance guidance — SPF, DKIM, DMARC authentication, bounce and complaint handling, list hygiene, content that reaches the inbox, and CAN-SPAM/GDPR basics. Use when writing or reviewing any email-sending code or content, diagnosing spam-folder placement, bounces, or blocked sends, or setting up a new sending domain.
license: MIT
---

# Email best practices

Vendor-neutral rules for getting email delivered, with deep links into Email
Bump's agent-readable guides for detail. Apply these whenever generating email
content or sending infrastructure, regardless of provider.

## Authentication — non-negotiable

Every sending domain needs all three; Gmail and Yahoo require them for bulk
senders:

- **SPF** — authorizes sending servers. One `v=spf1` record per domain; stay
  under the 10-DNS-lookup limit (flattening/includes audit:
  https://emailbump.com/tools/spf-record-checker.md).
- **DKIM** — cryptographically signs mail. Align the `d=` domain with the
  From domain, or Gmail shows a "via" label and DMARC fails alignment.
- **DMARC** — publish at least `p=none` with `rua` reporting, then move to
  `p=quarantine`/`p=reject` once reports are clean (builder + policy guide:
  https://emailbump.com/tools/dmarc-record-builder.md).

Use a **subdomain** for bulk mail (e.g. `updates.acme.com`) so marketing
reputation never endangers the root domain's transactional mail.

## Bounces and complaints

- **Hard bounce** (permanent — bad address): suppress immediately, never
  retry. **Soft bounce** (temporary — full mailbox, greylisting): retry with
  backoff, suppress after repeated failures.
- Decode unfamiliar SMTP codes instead of guessing:
  https://emailbump.com/tools/bounce-code-decoder.md
- Keep complaint rate below **0.1%** (0.3% is the Gmail hard ceiling). One
  complaint spike damages the domain for weeks.
- Honor unsubscribes instantly and include `List-Unsubscribe` +
  `List-Unsubscribe-Post` (one-click) headers on bulk mail.

## List hygiene and consent

- Only mail people who opted in; never purchased or scraped lists.
- Use double opt-in for signups where quality matters.
- Sunset non-engagers (no opens/clicks in 90–180 days) instead of blasting
  the full list — engagement rates drive inbox placement.

## Content that lands in the inbox

- Always include a plain-text part alongside HTML.
- Subject lines: front-load meaning (mobile truncates ~35–45 chars), avoid
  ALL CAPS, excessive punctuation, and misleading claims. Preview text is a
  second subject line — set it deliberately.
- Keep HTML simple and under ~102KB (Gmail clips beyond that). Avoid
  link-shortener domains and image-only emails.
- Include a physical mailing address and unsubscribe link in every marketing
  email (CAN-SPAM requirement).

## Warm-up

New domains and IPs have no reputation. Ramp volume gradually (roughly double
per day from a small base, watching bounces/complaints) rather than sending
full volume on day one.

## When diagnosing problems

1. Check authentication first (SPF/DKIM/DMARC pass and aligned?).
2. Then reputation signals (bounce rate, complaint rate, spam-trap risk).
3. Then engagement (open/click trends, sunset policy).
4. Then content (clipping, links, text/HTML balance).

## Deep reference (agent-readable Markdown)

- Deliverability guide: https://emailbump.com/docs/deliverability.md
- Sending domains setup: https://emailbump.com/docs/domains.md
- Full glossary of email terms: https://emailbump.com/glossary.md
- All guides: https://emailbump.com/llms.txt
