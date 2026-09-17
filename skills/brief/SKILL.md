---
name: brief
description: Generate or refresh today's daily brief as the persistent "Today's Brief" working surface. Auto-fires on "/brief", "morning brief", "today's brief", "what's on today", "what am I working on today", "give me my brief", or any phrase asking for today's working surface. Renders 6 fixed sections — Center of Gravity, Calendar Block, Priority Tasks, Outreach Queue, Yesterday's Reflection, Today's Reflection. Filters against surfacing preferences, yesterday's closures, and the snooze ledger. Explicit actions persist in the canonical v0.7.0 state and are mined overnight by cortex /listen. Read-only across sources — no drafting, no sends. `/brief --tomorrow` dispatches to calendar-first next-day planning — see `commands/brief.md` Step -1.
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->


See `commands/brief.md` for the full generation workflow.

## When this skill fires

- User runs `/brief` directly
- User says: "morning brief", "today's brief", "give me my brief", "what's on today", "what am I working on today", "start my day"
- Scheduled task fires (if user opted in during `/setup-brief`)

## What this skill is NOT for

- **Drafting or sending anything.** This is read-only. Drafting happens in `/process-brief` after the user annotates.
- **Multi-day planning.** This is today only. For tomorrow, use `/brief --tomorrow`, which dispatches to the calendar-first planning workflow.
- **Weekly summaries.** Use `growth` (relationships, absorbed referral-engine), or this plugin's own `/review` for week-level work.
- **Replacing the dashboard.** Cortex's `DASHBOARD.md` is the always-on memory index. The brief is a daily working surface that includes today's slice of dashboard context.

## Inputs

- `<config-root>/memory/me/identity.md` — time zone, tool inventory
- `<config-root>/plugins/briefing.user-context.md` — section toggles, sort defaults, empty-state behavior, annotation placeholder hints
- `<config-root>/memory/me/surfacing-prefs.md` — **required filter**: do-not-resurface list + noise rules applied to tasks/outreach before render
- `<config-root>/memory/DASHBOARD.md` — active project context (cortex)
- Calendar MCP — today's events (timeline strip + written list with per-meeting context)
- HubSpot MCP — priority tasks (owner=you, due today / overdue; P0/P1 only)
- growth pipeline (`<config-root>/relationships/today.json`) / lead-engine — outreach queue (legacy weekly-outreach fallback)
- `<config-root>/briefs/<yesterday>.md` — yesterday's reflection (`## Reflection`)
- `<config-root>/briefs/<yesterday>.closures.json` — explicit closures and carry-forward annotations from cortex `/listen`
- `<config-root>/briefs/.snooze-ledger.json` — hidden and returning tasks/outreach

## Outputs

- `<config-root>/briefs/<today>.md` — markdown twin (canonical record)
- Cowork artifact "Today's Brief" (stable id `todays-brief`) when supported, or the Markdown working surface on OpenAI hosts
- `<config-root>/briefs/<today>.state.json` — canonical v0.7.0 state mirror when the user records task/outreach actions, annotations, reprioritizations, snoozes, or today's reflection; mined by cortex `/listen` Step 1.5
- One short chat message confirming the brief is ready, with the twin path + filtered-item count

## Cost profile

- ~3-5K tokens of synthesis on top of raw connector payloads
- Per-section caps: 12 meetings, 12 tasks
- Cheap to run; designed to be invoked daily without breaking the bank

## Failure modes

- No `<config-root>` set → routes user to `/setup-brief`
- No calendar / inbox / CRM MCP available → that section renders as "source not connected"; other sections still populate
- Interactive artifact unavailable → render stable item ids in Markdown; accept explicit actions in chat and merge them into the canonical v0.7.0 state file per `references/openai-portability.md`
- `brief_enabled: false` in user-context → stops cleanly with re-enable instructions
