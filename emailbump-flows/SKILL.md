---
name: emailbump-flows
description: Build automated email journeys with Email Bump — welcome series, onboarding, win-backs — from a trigger and a list of steps (send, wait, branch, tag, webhook). Use when the user wants email sent automatically in response to a signup, an event, or joining a list.
license: MIT
---

# Email Bump — automated flows

A flow is a trigger and a list of steps. Someone enters, and the steps run in
order: send an email, wait, branch on what they did, tag them, call a webhook.

## Setup

- Base URL: `https://emailbump.com/api/v1`
- Auth: `Authorization: Bearer $EMAILBUMP_API_KEY` (project key, or an
  all-access key plus `X-Project-Id: <uuid>`).

## Check these first

One request each. Skipping them is how a welcome series goes out from the wrong
address with no way to unsubscribe. `GET /v1/me` answers all three at once —
project, sender readiness, and whether the footer carries the customer's own
postal address — so start there and use the longer forms below to fix whatever
it reports.

1. **Which project?** `GET /v1/projects` lists them flat with the workspace each
   belongs to. A workspace and its first project share a name by default — read
   the `object` field rather than guessing from the name.
2. **Does the project have a real postal address?** `GET /v1/projects/{id}`
   returns `email_footer`. If `is_your_own_address` is `false`, every email
   prints Email Bump's address as the sender's. Set theirs with
   `PATCH /v1/projects/{id}` (`company_name`, `address_line1`, `city`, `state`,
   `postal_code`, `country`) before sending.
3. **Is there a verified sending domain?** `GET /v1/domains` with a project
   key, or `GET /v1/projects/{id}/domains` with an all-access key. If not, add
   it (`POST /v1/domains`) and publish the DNS records first — that's its own
   job, not a step to rush inside this one.

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

## Create a flow

`POST /v1/flows`

```bash
curl -X POST https://emailbump.com/api/v1/flows \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Welcome series",
    "trigger": { "type": "list_join", "list_id": "LIST_UUID" },
    "steps": [
      {
        "type": "send",
        "subject": "You are in",
        "from_name": "Acme",
        "from_email": "hello@mail.acme.com",
        "html": "<p>Thanks for joining, {{ contact.first_name }}.</p>"
      },
      { "type": "wait", "days": 2 },
      {
        "type": "send",
        "subject": "The one thing worth knowing",
        "from_name": "Acme",
        "from_email": "hello@mail.acme.com",
        "html": "<p>Here it is.</p>"
      }
    ]
  }'
```

A flow is created active unless you pass `"status": "draft"`. `GET /v1/flows/{id}`
returns it with its steps; `activate` and `pause` change its state.

## Always set the sender

**Set `from_email` and `from_name` on every send step.** They are optional, but
a step without them sends from the project's verified domain with no display
name — and from Email Bump's shared domain if the project has verified nothing
at all. Neither is what a customer wants their audience to see.

- `from_email` must be on a domain verified for that project. Check with
  `GET /v1/domains` (project key) or `GET /v1/projects/{team_id}/domains`
  (all-access key).
- `from_name` is what appears in the inbox. Use the brand, not "noreply".
- Set `reply_to` when a person will actually read replies.

If the domain isn't verified yet, say so and set it up first — sending from an
unverified domain is what a spam folder is for.

## Do not write your own unsubscribe

Every flow email gets a CAN-SPAM footer and a one-click `List-Unsubscribe`
header automatically: the opt-in reason, the sender's physical address, and an
unsubscribe link tied to that contact. Don't add your own — you'll get two, and
yours won't be wired to the suppression list.

## Steps

| Step | What it does |
|---|---|
| `send` | Sends an email. `subject` plus one of `html`, `mjml`, or `template_id`. |
| `wait` | Holds for `seconds` / `minutes` / `hours` / `days`. |
| `wait_until` | Holds until a time of day, e.g. `"09:30"`. |
| `wait_event` | Holds until a tracked event arrives, or a timeout passes. |
| `branch` | Splits the path on a condition; each leg is its own list of steps. |
| `split` | Random split for A/B: `branches`, each `{ key, weight?, steps }`. Needs at least two. |
| `add_list` / `remove_list` | `list_id`. Puts the contact on a list, or takes them off it. |
| `set_attribute` | `key`, `value`. Writes a contact attribute — the field is `key`, not `attribute`. |
| `notify` | `headline`, optional `message`. Emails your own team, not the contact. |
| `webhook` | `url`, optional `method`. The URL must resolve to a public address. |
| `exit` | Ends the journey. |

That is the whole list, and the names are exact. There is no `tag`, `untag` or
`goal` step — use `add_list`/`remove_list` to group people and `exit` to end a
journey. Anything else comes back as
`unknown variant \`x\`, expected one of ...`, which names the valid set; read it
rather than guessing again.

Steps after a `branch` aren't allowed — put them inside the legs.

## Triggers

`list_join` (`list_id`, or omit it to catch every list), `segment_enter`
(`segment_id`), `event` (a tracked behavioural event), `email_event`,
`contact_created`, `date_based` (`attribute`, `offset_days`), and `manual`
(enrol through the API).

These names are exact. `list_joined` and `segment_entered` are refused with a
400 that lists the valid ones — read it rather than guessing again. Full shapes:
the Flows API reference below.

## Enrol someone

`POST /v1/flows/{id}/enroll` with `{"email": "..."}` or `{"contact_id": "..."}`
— the way to drive a `manual` flow, and useful for testing any other.

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

## Links get rewritten, and whose domain they show

Every link in a campaign is rewritten so clicks can be recorded. By default they
go through Email Bump's shared tracking domain, which works and is invisible to
most recipients — but someone hovering a link sees our domain, not the
customer's.

If that matters to them, branded link tracking sends links through
`links.theirdomain.com` instead:

```bash
curl -X POST https://emailbump.com/api/v1/domains/DOMAIN_ID/tracking \
  -H "Authorization: Bearer $EMAILBUMP_API_KEY"
```

It provisions in two steps, and `dns_records` only ever holds the record that
exists yet — publish the `step: "certificate"` one, poll
`GET /v1/domains/{id}`, then publish the `step: "routing"` one that appears
after it validates. Tens of minutes. Links keep working through the shared
domain the whole time, so it is safe to start and leave.

Do not tell a customer their links are unbranded as though it were a fault — it
is the default, it tracks correctly, and turning it on is optional.

## Guardrails

- A flow sends real email to real people, repeatedly, without anyone watching.
  Show the user the steps, the timing, and the sender before activating one.
- Create it as a draft when the user hasn't seen the content yet.
- `pause` stops it without losing enrollments. `delete` only works on a flow
  nobody has ever been through — once a single contact has been enrolled it
  returns `409` and stays for good, because the emails it sent would otherwise
  be left in the reports with nothing to attribute them to. For a live flow,
  pause is the answer; don't offer deletion as a way to stop one.

## Reference

- Flows API: https://emailbump.com/docs/flows-api.md
- Automated flows guide: https://emailbump.com/docs/flows.md
- Sending domains: https://emailbump.com/docs/domains.md
