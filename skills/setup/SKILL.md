---
disable-model-invocation: true
name: setup
description: One-time interview that configures the briefing plugin — section order, sort defaults, empty-state behavior, annotation placeholder hints, optional auto-run schedule. Auto-fires on "/setup-brief", "configure briefing", "set up the brief", "configure my brief", or when the user runs `/brief` and no config file exists. Writes results to `<config-root>/plugins/briefing.user-context.md`. Re-run anytime to update.
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->


See `commands/setup-brief.md` for the full setup workflow.

## When this skill fires

- User runs `/setup-brief` directly
- User says: "configure briefing", "set up the brief", "configure my brief"
- User runs `/brief` and no `<config-root>/plugins/briefing.user-context.md` exists — `/brief` offers to run setup or fall back to defaults

## What this skill is NOT for

- **Capturing identity / voice / company info.** Those are already in `<config-root>/memory/me/identity.md` and `<config-root>/memory/me/voice.md` from cortex's `/setup-identity` and Comms Desk's `/setup-voice`. This setup reads them; it doesn't re-ask.
- **Connecting MCPs.** This is a config-only interview. If a connector isn't wired, the relevant section will simply render "source not connected" in `/brief`.
- **Picking a config root.** If the config root pointer doesn't exist yet, Step 0 prompts for it once — but that's a marketplace-wide convention, not a brief-specific decision.

## Inputs

- `<config-root>/memory/me/identity.md` (read-only) — time zone, tool inventory
- Existing `<config-root>/plugins/briefing.user-context.md` (if present) — to update a subset rather than start over
- `references/user-context.template.md` (bundled with plugin source) — structure for the output file

## Outputs

- `<config-root>/plugins/briefing.user-context.md` — captured preferences
- `<config-root>/briefs/` directory created if missing
- (Optional) registered scheduled task via ops `/register-schedules` if user opts in

## Behavior rules

- Idempotent. Re-running updates only the sections the user wants to update.
- Default behavior is conservative: all sections on, no auto-run, master toggle on.
- Don't bombard. One interview section at a time, confirm before moving on.
- If the user is in a hurry, accept "use defaults" and skip ahead.
