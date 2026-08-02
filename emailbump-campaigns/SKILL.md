---
name: emailbump-campaigns
description: Create, A/B test, schedule, and send Email Bump marketing campaigns — audience targeting by list or segment, Liquid-personalized HTML or stored templates, gradual/timezone send strategies, and A/B winner selection. Use when the user wants to run a newsletter, product announcement, or any marketing send.
license: MIT
---

# Email Bump — campaigns & A/B testing

Marketing sends to lists and segments, with A/B testing built in.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` (keys start with `ebk_`)

## Create a campaign (draft)

`POST /v1/campaigns`

```bash
curl -X POST https://emailbump.com/api/v1/campaigns \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "July product update",
    "subject": "New in July: three features you asked for",
    "preview_text": "Three features you asked for",
    "from_name": "Acme",
    "from_email": "hello@updates.acme.com",
    "segment_id": "SEGMENT_UUID",
    "html": "<h1>Hi {{ contact.first_name }}</h1><p>...</p>"
  }'
```

Key fields:

- `name` (required) — internal name.
- `subject`, `preview_text`, `from_name`, `from_email` — envelope. The from
  domain must be verified.
- Audience: exactly one of `list_id`, `segment_id`, or `contact_id` (single
  recipient).
- Body: `html` (Liquid personalization, e.g. `{{ contact.first_name }}`) or
  `template_id` for a stored template.
- Campaigns are created as **drafts** — nothing sends until the send call.

Manage drafts with `GET /v1/campaigns` (filter by `status`: draft | scheduled |
sending | sent | paused), `GET /v1/campaigns/{id}`, `PATCH /v1/campaigns/{id}`.

## Send or schedule

`POST /v1/campaigns/{id}/send` — empty body sends **immediately**. Optional
body:

- `scheduled_at` — future RFC 3339 time to schedule instead of sending now.
- `send_strategy` — `fixed` | `gradual` | `smart`; with `gradual`, set
  `batch_percent` + `batch_interval`.
- `timezone_mode` — `specific` (with `send_timezone`) or `local` (deliver in
  each contact's local time; `local_past_behavior` controls already-past
  slots).
- `recipients_at_send_time` — re-evaluate the audience at send time rather
  than freezing it now.

## A/B testing

Enable on create with `ab_enabled` (plus test share, winning metric, and wait
window fields), or manage variants directly:

- `POST /v1/campaigns/{id}/variants` — add a variant (2–4 total). Omitting
  `variants` on create with `ab_enabled` seeds a default "A"/"B" pair.
- `PATCH` / `DELETE /v1/campaigns/{id}/variants/{variant_id}` — edit variants.
- `GET /v1/campaigns/{id}/ab` — live A/B results per variant.
- `POST /v1/campaigns/{id}/ab/winner` — pick the winner manually (otherwise
  the winning metric decides after the wait window).

Full reference: https://emailbump.com/docs/campaigns-api.md

## The sender's postal address

Anti-spam law wants a real postal address for whoever is sending, and it appears
in the footer of every marketing email. A project that hasn't set one prints
Email Bump's address instead — legally fine for Email Bump, wrong for the
customer, and the sort of thing nobody notices until a recipient asks.

Check it before any marketing send:

```bash
curl https://emailbump.com/api/v1/projects/PROJECT_ID \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY"
```

`email_footer.is_your_own_address: false` means it's ours. To fix it:

1. **Ask the human** for the business address that should appear on their email.
2. If they'd rather you find it: look at their website — the page footer, a
   contact page, terms, or privacy policy usually carries it — and **show them
   what you found and get a yes before writing it.**
3. Then set it:

```bash
curl -X PATCH https://emailbump.com/api/v1/projects/PROJECT_ID \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "company_name": "Acme Inc", "address_line1": "12 Bridge Street",
        "city": "Austin", "state": "TX", "postal_code": "78701", "country": "USA" }'
```

Never invent an address, and never use one you only half-recognise. A wrong
address in a compliance footer is worse than a missing one.

## Two things the inbox will show you

**Set `from_name` on every send.** Without it the recipient sees a bare address
— `hello@mail.acme.com` — where the brand should be. It is the first thing shown
in an inbox list and the main reason a legitimate email looks like spam.

**Don't write your own unsubscribe.** Every campaign and flow email gets an
unsubscribe link, the sender's postal address, a permission reminder and a
one-click `List-Unsubscribe` header added at send time. Writing your own puts
two of each in front of the recipient, and only the platform's is wired to the
suppression list — someone clicking yours may not actually be unsubscribed.

Both are reported back: `POST /v1/campaigns` and `POST /v1/flows` return a
`warnings` array when a send has no `from_name` or the content mentions
unsubscribing. Read it.

## Guardrails — always human in the loop

- **Never call `/send` on a real audience without explicit confirmation.**
  Before sending: report the audience (list/segment name and size), subject,
  from address, and schedule, then wait for the user's go-ahead.
- Prefer a rehearsal first: create the same campaign with `contact_id` set to
  the user's own contact, send that, and let them review the rendered email.
- Sent campaigns cannot be recalled. Scheduled campaigns can still be edited
  or paused before their send time.

## Related

- Templates API (reusable MJML): https://emailbump.com/docs/templates-api.md
- Analytics & activity: https://emailbump.com/docs/analytics.md
- Webhooks for engagement events: https://emailbump.com/docs/webhooks-api.md
