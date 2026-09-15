---
name: dashboard
description: Rich visual dashboard of your Nucleus stack — installed plugins, this week's activity, cortex memory health, outreach pipeline, time + invoicing. Renders as a live Cowork HTML artifact you can keep open and refresh, or as a markdown snapshot in Claude Code. Auto-fires on "/dashboard", "show my dashboard", "open my nucleus dashboard", "weekly view", "what's my stack doing", or any phrase asking for the visual stack overview. Sibling to /status (terse text) and /diagnose (troubleshooting).
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->


See `commands/dashboard.md` for the full workflow.

## When this skill fires

- User runs `/dashboard` directly
- User says: "show my dashboard", "open my nucleus dashboard", "weekly view", "what's my stack doing", "give me the visual"
- After /end-week or other rhythm rituals, when the user wants the high-level view

## What this skill is NOT for

- **Quick check** — that's `/status` (terse text in chat).
- **Troubleshooting** — that's `/diagnose`.
- **Memory triage** — that's `/recall` (with recall-time actions) or `/rehearse`.
- **Editing config** — per-plugin `/setup-*` commands.

## Inputs (all read-only)

- `<config-root>/memory/me/identity.md`, `<config-root>/memory/me/voice.md`
- `<config-root>/memory/` (DASHBOARD, triage-log, decay-config, scope-migration marker, person/, sampled knowledge entries)
- `<config-root>/plugins/*.user-context.md` (per-plugin setup state)
- `<config-root>/relationships/pipeline.md` + `sent-log.md` (if growth is installed)
- `<config-root>/time-log.csv` (if admin installed)
- `~/.brightway-state/agent-log.jsonl` (this week's activity)
- Runtime MCP tool list (connector detection)
- `references/nucleus-dashboard-template.html` (bundled with ops source)

## Outputs

- Cowork artifact titled "Nucleus Dashboard" (HTML, re-rendered on each run, single persistent surface)
- Markdown fallback at `<config-root>/nucleus-dashboard.md` in Claude Code
- One-line chat notification with the artifact name or snapshot path

## Cost

Zero model calls in v1. Pure filesystem reads + template rendering. Target < 5 seconds wall time.

## Sections (v1)

1. **Stack overview** — installed plugins, configured/template/missing state, connector availability, last-run per skill
2. **This week** — activity histogram, top 10 skills by run count, rusty skills (no run in 14+ days)
3. **Cortex memory health** — entries by state (Fresh/Stale/Dormant/Cold/Demoted), person pages by state, triage decisions this week
4. **Relationship pipeline** (if growth is installed) — active signals, drafts written/sent, replies, booked calls
5. **Time + invoicing** (if admin installed) — hours by client, billable split, days since last invoice
6. **Impact loop** (v1 placeholder, v1.1 ships content) — drafted→sent ratios, mining accept rate, P0 completion rate

## Failure modes

- Pointer missing → routes user to a setup command
- Cortex memory missing → section 3 renders "Memory not initialized — install cortex"
- Agent log missing → section 2 renders "No telemetry yet — runs will appear here after first invocation"
- Cowork artifact tools unavailable → falls back to markdown snapshot at `<config-root>/nucleus-dashboard.md`
- Individual section data unreadable → that section renders honest "not available" placeholder, doesn't fail the whole render
