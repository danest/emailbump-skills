# Email Bump Agent Skills

Focused email capabilities for AI agents — install once, and your agent knows how
to send email, manage audiences, run campaigns, and provision Email Bump
infrastructure the right way.

Compatible with any tool that reads the open [Agent Skills](https://agentskills.io)
`SKILL.md` format: Claude Code, Cursor, Codex, Devin, Gemini CLI, GitHub Copilot,
OpenCode, and more.

## Install

```bash
npx skills add danest/emailbump-skills
```

Or copy any skill directory into your agent's skills folder (for Claude Code:
`~/.claude/skills/`).

Every skill authenticates with an Email Bump API key:

```bash
export EMAILBUMP_API_KEY="ebk_..."
```

Create keys in the dashboard under **Settings → API keys**, or provision them
programmatically with the `emailbump-management` skill.

## Skills

| Skill | What your agent learns |
| --- | --- |
| [`emailbump`](./emailbump/SKILL.md) | Send transactional email through the REST API — immediate and scheduled sends, templates, personalization, and rate limits. |
| [`emailbump-audience`](./emailbump-audience/SKILL.md) | Manage contacts, lists, segments, consent, and behavioral events. |
| [`emailbump-campaigns`](./emailbump-campaigns/SKILL.md) | Create, A/B test, schedule, and send marketing campaigns with templates. |
| [`emailbump-management`](./emailbump-management/SKILL.md) | Provision workspaces, projects, sending domains, flows, and project API keys. |
| [`email-best-practices`](./email-best-practices/SKILL.md) | Deliverability, authentication (SPF/DKIM/DMARC), compliance, and bounce handling — grounded in Email Bump's guides. |

## Human in the loop

Sending email is irreversible. Every skill in this set instructs agents to
confirm with a human before any send that reaches a real audience, and to prefer
single-recipient test sends first. Keep it that way in your own prompts.

## Docs

The full platform is agent-readable: [llms.txt](https://emailbump.com/llms.txt),
[agents.md](https://emailbump.com/agents.md), and every docs page as Markdown at
`https://emailbump.com/docs/<slug>.md`.
