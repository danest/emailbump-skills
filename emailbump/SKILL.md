---
name: emailbump
description: Send transactional email through the Email Bump REST API — immediate and scheduled sends, reusable templates, Liquid personalization, and account limits. Use when the user wants to send an email, receipt, alert, or notification programmatically via Email Bump.
license: MIT
---

# Email Bump — transactional sending

Email Bump is an email platform for marketing, transactional email, and customer
journeys. This skill covers the transactional sending API.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` (keys start with `ebk_`)
- All requests and responses are JSON.

If `EMAILBUMP_API_KEY` is not set, ask the user to create one in the Email Bump
dashboard (Settings → API keys) and export it. Never print the key back.

## Send an email

`POST /v1/emails`

```bash
curl -X POST https://emailbump.com/api/v1/emails \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "Acme <hello@updates.acme.com>",
    "to": "customer@example.com",
    "subject": "Your receipt",
    "html": "<p>Thanks for your order, {{ contact.first_name }}!</p>",
    "text": "Thanks for your order!"
  }'
```

Body fields:

- `from` (required) — display name + address. The domain must be a verified
  sending domain on the project.
- `to` (required) — a single recipient address.
- `subject` — required unless a template supplies it.
- `html` / `text` — body content. Provide both when possible; `text` improves
  deliverability and accessibility.
- `reply_to` — optional reply address.
- `template_id` — send a stored template instead of inline `html` (see
  Templates below). Template variables are rendered with Liquid.
- `scheduled_at` — RFC 3339 timestamp in the future. The response returns
  `status: "scheduled"` and Email Bump delivers it when due. Omit to send now.

Personalization uses Liquid, e.g. `{{ contact.first_name }}`; contact
attributes come from the project's contact record for the recipient.

## Templates

Reusable MJML templates can be managed via `GET/POST /v1/templates` and
`GET/PATCH/DELETE /v1/templates/{id}`, then referenced from sends with
`template_id`. Full schema: https://emailbump.com/docs/templates-api.md

## Limits

`GET /v1/limits` returns the account's current sending limits and usage. Check
it before bulk work and back off on HTTP 429.

## Guardrails

- Emails cannot be unsent. Before sending to a real customer address, confirm
  the exact recipient, subject, and body with the user.
- For anything repeated or bulk, send one test email to the user's own address
  first and wait for their approval.
- Never invent sender addresses — use a from-domain the user has verified.

## Reference

- Transactional API guide: https://emailbump.com/docs/transactional-api.md
- API overview (auth, errors, pagination): https://emailbump.com/docs/api-reference.md
- Webhooks for delivery/open/click/bounce events: https://emailbump.com/docs/webhooks-api.md
