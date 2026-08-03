---
name: emailbump-management
description: Provision Email Bump account structure programmatically with an all-access key — workspaces, projects, sending domains with DNS verification, and project keys. Use when setting up Email Bump for a new app, adding a sending domain, or when an agent needs to bootstrap email infrastructure end to end.
license: MIT
---

# Email Bump — account API

Provision the account structure that the other Email Bump skills operate
inside: workspaces → projects → domains, and per-project API keys.

## If they have no account yet

Make one from here. It takes an email address and nothing else — no password, no
browser:

```bash
curl -X POST https://emailbump.com/api/v1/signup \
  -H "Content-Type: application/json" \
  -d '{ "email": "them@company.com", "name": "Dana", "accept_terms": true }'
```

That returns **two** keys: `api_key`, scoped to the new project, for sending;
and `account_key`, for creating further workspaces, projects and keys. Between
them there is nothing in provisioning that needs a browser. Over MCP it's
`create_account`; over the CLI, `emailbump signup --email … --accept-terms`.

Two things are theirs to give, not yours:

1. **The address.** Ask for it. Never guess it from a domain or a git config.
2. **`accept_terms`.** Creating an account means agreeing to the terms at
   https://emailbump.com/terms. Ask them, in words, and only send `true` if they
   said yes.

Then verify the address — the account can read the API immediately but **cannot
send a single message until it's verified**. The email carries a six-character
code:

- **MCP:** ask them to read the code out, then `verify_email_code`.
- **CLI:** `emailbump verify-email --email them@company.com PL8FJD`

Nothing here can read their inbox, and nothing should. They read it out.

A password is never required. If they want to sign in to the dashboard later,
"Forgot password" sets one on an account that never had one.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` using an **all-access key** —
  from `POST /v1/signup`, `emailbump login`, or the dashboard. An all-access key
  can also act in any project its owner admins by adding an
  `X-Project-Id: <uuid>` header to the regular project endpoints.
- A **project key** can't create workspaces or projects, but it *can* set up its
  own project's sending: `GET/POST /v1/domains` and `GET /v1/domains/{id}`. If
  all you're doing is getting one project sending from its own domain, you don't
  need the account key at all.
- `GET /v1/me` says which project a key is in and what it can do; `GET /v1`
  lists the API and needs no key.

## Typical bootstrap flow

1. `GET /v1/workspaces` — see existing workspaces; `POST /v1/workspaces` to
   create one.
2. `POST /v1/projects` — create a project inside a workspace.
3. `POST /v1/projects/{team_id}/domains` — add a sending domain. The response
   includes the DNS records (SPF/DKIM) to publish. (With a project key:
   `POST /v1/domains`, same body, no project id needed.)
4. Publish the DNS records (with the user, or via their DNS provider's tooling
   if authorized), then `GET /v1/projects/{team_id}/domains/{domain_id}` to
   check/trigger verification (project key: `GET /v1/domains/{domain_id}`). Poll
   until verified — propagation can take minutes to hours; don't tight-loop.
4b. Optional, and worth offering: `POST /v1/domains/{domain_id}/tracking` turns
   on branded link tracking, so links in their mail read
   `links.theirdomain.com` instead of Email Bump's shared tracking domain.
   Until it's on, clicks are still tracked and nothing is broken — a recipient
   hovering a link just sees our domain rather than theirs.

   It provisions in two steps and **`dns_records` only ever contains the record
   that exists yet**, so don't tell the user "publish the records" and hand them
   an empty list. Publish the one with `purpose: "Link tracking"` and
   `step: "certificate"`, poll `GET /v1/domains/{domain_id}` (that call is what
   advances provisioning), then publish the `step: "routing"` record that
   appears once the certificate validates. Tens of minutes end to end — much
   slower than domain verification. `DELETE` on the same path turns it off.
5. `POST /v1/projects/{team_id}/api-keys` — mint a project-scoped key (`ebk_…`)
   for application sending. Show it to the user once and tell them to store it as
   a secret; it is not retrievable later.
6. Optionally `POST /v1/flows` with the new project key (or this key plus an
   `X-Project-Id: {team_id}` header) — create a starter automation, e.g. a
   welcome series. Flows are a project resource, not part of provisioning.

7. `PATCH /v1/projects/{team_id}` — set the project's company name and postal
   address. Do this as part of provisioning, not later: without an address every
   marketing email that project sends prints Email Bump's address as the
   sender's own. `GET /v1/projects/{team_id}` shows what the footer will say
   under `email_footer`, including `is_your_own_address`.

Reading the account: `GET /v1/workspaces` returns workspaces with the projects
inside them; `GET /v1/projects` returns projects flat with the workspace each
belongs to; `GET /v1/projects/{team_id}/domains` lists a project's sending
domains with their verification state — check it before setting any From
address, because mail from an unverified domain doesn't land. Every object carries `"object": "workspace"` or `"object": "project"`
— a workspace and its first project share a name by default, so the name alone
doesn't say which one you're holding.

Removing a project: `DELETE /v1/projects/{team_id}` with `{"confirm_name": "..."}`
matching its name exactly. It takes the contacts, campaigns, flows, domains,
keys and received mail with it, and the workspace too if it was the last project
in it. Confirm with the human first — there is no undo.

Full reference: https://emailbump.com/docs/api-reference.md — workspaces,
projects, and domains each have their own page.

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

## Guardrails

- Creating workspaces/projects may have billing implications — state what you
  are about to create and on which account before doing it. When someone wants another
  sending unit, ask whether they mean a project inside an existing workspace
  (shared bill) or a separate workspace (its own bill) — the words get used
  interchangeably and the billing consequence is different.
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
