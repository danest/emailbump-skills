---
name: emailbump
description: Send transactional email through the Email Bump REST API — immediate and scheduled sends, cc and bcc copies, file attachments, reusable templates, Liquid personalization, and account limits. Use when the user wants to send an email, receipt, alert, invoice, or notification programmatically via Email Bump.
license: MIT
---

# Email Bump — transactional sending

Email Bump is an email platform for marketing, transactional email, and customer
journeys. This skill covers the transactional sending API.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` (keys start with `ebk_`)
- All requests and responses are JSON.

If `EMAILBUMP_API_KEY` is not set: ask the user for a key from the dashboard
(Developers → API Keys), or `emailbump login` for the browser flow. If they have
no account at all, you can make one from here — see
[emailbump-management](../emailbump-management/SKILL.md). All-access keys must
add `X-Project-Id: <uuid>` on these endpoints. Never print the key back.

## First call: `GET /v1/me`

Before the first send, ask what the key can actually do. Four separate things
refuse a send, and this names all of them at once:

```bash
curl https://emailbump.com/api/v1/me -H "Authorization: Bearer $EMAILBUMP_API_KEY"
```

```json
{
  "project": { "name": "Main", "object": "project" },
  "sending": {
    "ready": true,
    "email_confirmed": true,
    "verified_domains": ["mail.acme.com"],
    "default_from": "hello@mail.acme.com"
  },
  "footer": { "address": "12 Bridge Street, Austin TX", "is_your_own_address": true }
}
```

- `sending.email_confirmed: false` — the account holder hasn't entered the code
  we emailed. Nothing sends until they do.
- `verified_domains: []` — mail goes from the shared domain and looks like it.
  Add theirs with `POST /v1/domains`.
- `footer.is_your_own_address: false` — the legally-required postal address in
  the footer is *Email Bump's*, not the customer's. Fix before marketing sends.

Do not discover any of this by sending and reading the error. `GET /v1` lists
the whole API and needs no key at all.

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
- `to` (required) — a single recipient address. One message, one `to`: each
  send is metered and tracked on its own, so five *different* people is five
  calls (or a campaign). A one-element array is accepted; more than one is a
  400.
- `cc` / `bcc` — copies of the **same** message, as one address or an array,
  up to 50 between them. Everyone sees the `cc` addresses; nobody sees the
  `bcc` ones. Each copy is a send of its own: its own envelope and provider
  message id (so its bounces and opens are attributable to the person who got
  it), its own row in the log, and its own line against the monthly allowance
  — one `to` plus two copies is three emails. The response carries a `copies`
  array, one entry per copy with an `id` you can pass to `GET /v1/emails/{id}`.
  A copy to an address already hard-bounced or marked spam is dropped, named
  in `warnings`, and left out of the `Cc` header; the message still goes to
  `to`. Not accepted with `stream: "marketing"` — each marketing recipient
  needs their own consent check and unsubscribe link.
- `subject` — required unless a template supplies it.
- `html` / `text` — body content. Provide both when possible; `text` improves
  deliverability and accessibility.
- `reply_to` — optional reply address.
- `template_id` — send a stored template instead of inline `html` (see
  Templates below). Template variables are rendered with Liquid.
- `attachments` — files to send along (see below).
- `scheduled_at` — RFC 3339 timestamp in the future. The response returns
  `status: "scheduled"` and Email Bump delivers it when due. Omit to send now.
- `stream` — `transactional` (default) or `marketing`. A marketing send is a
  promotional one-off: it is **refused if the recipient has unsubscribed**, the
  unsubscribe footer and one-click headers are added server-side, and it is
  classified under the marketing stream in analytics. A recipient who isn't a
  contact yet is created as one. `scheduled_at` isn't supported with
  `marketing` — schedule promotional mail with a campaign. Never use the
  default transactional stream for promotional content just to skip the
  consent check.

Personalization uses Liquid, e.g. `{{ contact.first_name }}`; contact
attributes come from the project's contact record for the recipient.

## Did it arrive?

`POST /v1/emails` returns an id; `GET /v1/emails/{id}` says what became of it.

```bash
curl https://emailbump.com/api/v1/emails/$EMAIL_ID \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY"
```

`status: "sent"` means accepted for delivery, **not** delivered. The `events`
array is the truth — `delivery`, `open`, `click`, `bounce`, `complaint`. Mail to
a domain with no MX record reports `sent` and then bounces seconds later, so
when a user says "it never arrived", read the events before believing `status`.

## Fields we ignore, and say so

A field this endpoint doesn't read is ignored, and the response carries a
`warnings` array naming it. `from_name` is the common one: it's real on
campaigns and flows, but here the display name goes inside `from`
(`"Acme <hi@acme.com>"`). Read `warnings` — it is how a silent no-op announces
itself.

## Attachments

Up to 20 files, 25 MB in total, each base64-encoded:

```json
"attachments": [
  { "filename": "invoice-4821.pdf", "content": "JVBERi0xLjQK..." },
  { "filename": "logo.png", "content": "iVBORw0KGgo...", "content_id": "logo" }
]
```

- `filename` (required) — what the recipient sees when they save it.
- `content` (required) — the bytes, base64. A whole `data:` URI is accepted too.
- `content_type` — guessed from the filename when omitted.
- `content_id` — set it to embed the file in the HTML rather than list it at the
  bottom, and reference it as `<img src="cid:logo">`.

A PDF invoice or a photo is normal and delivers fine. Executables, archives and
macro-enabled documents are what spam filters look for — link to those instead of
attaching them.

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
