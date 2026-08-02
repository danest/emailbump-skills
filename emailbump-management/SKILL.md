---
name: emailbump-management
description: Provision Email Bump account structure programmatically with an all-access key — workspaces, projects, sending domains with DNS verification, and project keys. Use when setting up Email Bump for a new app, adding a sending domain, or when an agent needs to bootstrap email infrastructure end to end.
license: MIT
---

# Email Bump — account API

Provision the account structure that the other Email Bump skills operate
inside: workspaces → projects → domains, and per-project API keys.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` using an **all-access key**
  (created with the "all access" scope — via `emailbump login` or the dashboard).
  Project-scoped keys cannot provision. An all-access key can also act in any
  project its owner admins by adding an `X-Project-Id: <uuid>` header to the
  regular project endpoints (sending, contacts, campaigns).

## Typical bootstrap flow

1. `GET /v1/workspaces` — see existing workspaces; `POST /v1/workspaces` to
   create one.
2. `POST /v1/projects` — create a project inside a workspace.
3. `POST /v1/projects/{team_id}/domains` — add a sending domain. The response
   includes the DNS records (SPF/DKIM) to publish.
4. Publish the DNS records (with the user, or via their DNS provider's tooling
   if authorized), then `GET /v1/projects/{team_id}/domains/{domain_id}` to
   check/trigger verification. Poll until verified — propagation can take
   minutes to hours; don't tight-loop.
5. `POST /v1/projects/{team_id}/api-keys` — mint a project-scoped key (`ebk_…`)
   for application sending. Show it to the user once and tell them to store it as
   a secret; it is not retrievable later.
6. Optionally `POST /v1/flows` with the new project key (or this key plus an
   `X-Project-Id: {team_id}` header) — create a starter automation, e.g. a
   welcome series. Flows are a project resource, not part of provisioning.

Full reference: https://emailbump.com/docs/api-reference.md — workspaces,
projects, and domains each have their own page.

## Guardrails

- Creating workspaces/projects may have billing implications — state what you
  are about to create and on which account before doing it.
- Treat minted API keys as secrets: never write them into code, logs, or chat
  history beyond the one-time handoff; put them in the user's secret store or
  environment.
- Do not send real email from a domain until it verifies; sending from
  unverified domains fails or hurts reputation.

## Related

- API keys guide: https://emailbump.com/docs/api-keys.md
- Sending domains guide: https://emailbump.com/docs/domains.md
- Workspaces API: https://emailbump.com/docs/workspaces-api.md
- Projects API: https://emailbump.com/docs/projects-api.md
- Domains API: https://emailbump.com/docs/domains-api.md
- Flows API: https://emailbump.com/docs/flows-api.md
