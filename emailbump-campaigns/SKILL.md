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
