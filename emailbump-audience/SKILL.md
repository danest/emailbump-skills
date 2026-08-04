---
name: emailbump-audience
description: Manage an Email Bump audience — create and update contacts, organize lists, read live segments, track behavioral events, and handle subscribe/unsubscribe consent. Use when the user wants to add or update contacts, sync users into email lists, or track product events that drive automations.
license: MIT
---

# Email Bump — audience management

Contacts, lists, segments, consent, and events for an Email Bump project.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` (keys start with `ebk_`)

## Contacts

- `GET /v1/contacts` — list; supports `limit`, `offset`, `status`, `search`,
  and `email` (exact-match lookup: one contact or an empty set — URL-encode
  the address, a literal `+` in a query string decodes as a space).
- `POST /v1/contacts` — create **or update**: an email you already have is
  updated, not rejected. `201` for a new contact, `200` for an existing one, and
  the response's `created` says which. To add to lists in the same call the
  field is `list_ids` (an array); unknown fields like `list_id` or `tags` are
  a 400 naming the valid ones.
- `GET /v1/contacts/{id}` — fetch one.
- `PATCH /v1/contacts/{id}` — change only the fields you send.
- `DELETE /v1/contacts/{id}` — remove entirely (destructive — confirm first).
- `POST /v1/contacts/{id}/subscribe` / `POST /v1/contacts/{id}/unsubscribe` —
  change marketing consent.

```bash
curl -X POST https://emailbump.com/api/v1/contacts \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "jane@example.com", "first_name": "Jane", "attributes": {"plan": "pro"}}'
```

### Attributes merge — but know what you're writing

Keys you send are added or overwritten; keys already on the contact that you
don't mention are kept. To remove one, send it as `null`.

That matters because attributes are often a **record of something**: when
someone entered a giveaway, that they accepted the rules, which form they came
from. You will frequently be updating a contact for an unrelated reason —
setting a plan, tagging a source — with no idea those fields exist. They stay.

Two habits worth keeping anyway:

- **Read before you write** when you're about to overwrite a key you didn't
  set. `GET /v1/contacts/{id}` costs one call and tells you what's there.
- **Don't invent timestamps.** If a contact already has `signed_up_at` or
  similar, leave it. Re-stamping the moment you happened to run destroys the
  answer to "when did this actually happen".

Full field reference: https://emailbump.com/docs/contacts-api.md

## Lists (audiences)

- `GET /v1/lists` / `POST /v1/lists` — list and create.
- `GET /v1/lists/{id}` / `PATCH` / `DELETE` — manage one list.
- `GET /v1/lists/{id}/contacts` / `POST /v1/lists/{id}/contacts` — membership.
- `DELETE /v1/lists/{id}/contacts/{contact_id}` — remove a member.

Reference: https://emailbump.com/docs/lists-api.md

## Segments (read-only)

Segments are saved, live-evaluated audience queries defined in the dashboard.

- `GET /v1/segments` — list segments with current match counts.
- `GET /v1/segments/{id}` — one segment.

Use a segment's id as a campaign audience (`segment_id`). Reference:
https://emailbump.com/docs/segments-api.md

## Events

`POST /v1/events` tracks customer behavior (e.g. `order_completed`). Events
power segments and trigger automation flows.

```bash
curl -X POST https://emailbump.com/api/v1/events \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "jane@example.com", "event": "order_completed", "properties": {"total": 4999}}'
```

Reference: https://emailbump.com/docs/events-api.md

## Guardrails

- Never re-subscribe a contact who unsubscribed unless the user confirms the
  contact gave fresh consent — this is a legal (CAN-SPAM/GDPR) matter, not a
  data fix.
- `DELETE` on contacts and lists is irreversible; state what will be deleted
  and get confirmation.
- When importing many contacts, verify with the user that the list is
  permission-based (no purchased/scraped lists).
