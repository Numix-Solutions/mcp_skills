# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`mcp_skills` is a catalog of **Agent Skills** (`SKILL.md` files) distributed by the Numix MCP server. There is no application, build, lint, or test step — each skill is a self-contained markdown file that an AI agent (Claude Desktop, or any client of the Numix MCP server) loads and follows verbatim to drive a guided workflow against the Numix backend's MCP tools.

Numix is an AI accounting firm. The skills here orchestrate accounting workflows for end clients; the actual data operations are performed by MCP tools the skill *calls*, which live in the external Numix backend (not in this repo). A skill's job is the conversation script and the business rules — not the implementation of the tools.

## Layout

- One directory per skill, each containing a single `SKILL.md`.
- `README.md` is the public-facing index (with install instructions for Claude Desktop) and **must be updated when a skill is added or renamed** — its "List of Skills" table is the canonical list.
- Current skills: `start_rnd_credit/` — step-by-step R&D Tax Credit calculation, ending in pre-filled Form 6765/3523.

## Skill file conventions

Every `SKILL.md` begins with YAML frontmatter:

```yaml
---
name: kebab-case-name        # note: directory uses snake_case, frontmatter name uses kebab-case
description: >               # one sentence; what the skill does and when to use it
  ...
user-invocable: true         # exposes it as a /command in the chat client
---
```

The body is the agent's operating manual. When editing skills, preserve these patterns that the existing skills rely on:

- **Numbered steps executed in strict order** ("Follow every step in order. Never skip a step"). Gating language ("Do not proceed until the client confirms…") is load-bearing — it prevents the agent from calling write/calculation tools before data is collected and verified.
- **Tool calls are referenced by bare name in backticks** (e.g. `get_user_info`, `fetch_wages`, `update_wages`, `calculate_credit`). These resolve to MCP tools on the Numix server. When changing a tool name or its semantics, search every skill for the old name — the contract lives only in this prose, there is no shared schema file.
- **Session resume**: skills silently call `fetch_*` tools at session start to detect already-collected data and skip completed steps. Keep new write-flows idempotent and resumable in the same way.
- **2FA gating**: write operations require TOTP; skills check via `get_user_info` and block writes until enabled.
- **Upload pattern**: clients cannot upload files through chat. Skills direct them to the dashboard (`https://app.numix.app/dashboard?action=upload`), warn about a ~15–20s processing delay, then poll with the relevant `fetch_*` tool.
- **Exact-match business rules over AI estimates**: e.g. a transaction is R&D-eligible *only* if `user_defined_r_d === "R&D Qualified Expense"`; the `r_d_eligible` boolean is an AI estimate and must be ignored. Encode such rules explicitly and flag the trap, as the existing skill does.
- **Interactive forms with table fallback**: prompt for "display an interactive form" but always specify a structured-table-in-chat fallback for clients that lack form rendering.
- Client support escalation is `office@numix.co` or the Slack channel; surface this on repeated failures rather than looping.
