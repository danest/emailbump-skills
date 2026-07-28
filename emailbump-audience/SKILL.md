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

- `GET /v1/contacts` — list; supports `limit`, `offset`, `status`, `search`.
- `POST /v1/contacts` — create a contact (email + attributes).
- `GET /v1/contacts/{id}` — fetch one.
- `PATCH /v1/contacts/{id}` — update attributes.
- `DELETE /v1/contacts/{id}` — remove entirely (destructive — confirm first).
- `POST /v1/contacts/{id}/subscribe` / `POST /v1/contacts/{id}/unsubscribe` —
  change marketing consent.

```bash
curl -X POST https://emailbump.com/api/v1/contacts \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email": "jane@example.com", "first_name": "Jane", "attributes": {"plan": "pro"}}'
```

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
  -d '{"email": "jane@example.com", "name": "order_completed", "properties": {"total": 4999}}'
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
