---
name: process-brief
description: Read the actions + annotations the user logged into today's "Today's Brief" artifact (v0.5.0 blob — tasks/annotations/outreach_actions), classify each, and route it — draft reply via Gmail, move CRM task due date, dismiss; stage task actions (done→COMPLETED, delegate→delegatee task, skip→defer) and outreach actions (sent/nudge→touch, booked→prep task, let_go→close). Auto-fires on "/process-brief", "process my brief", "act on my annotations", "follow up on my brief". Intra-day actor; durable memory write-backs + suppression learning happen in cortex /end-day Step 2c. Drafts only — never sends.
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->


See `commands/process-brief.md` for the full workflow.

## When this skill fires

- User runs `/process-brief` directly
- User says: "process my brief", "act on my annotations", "process today's annotations", "follow up on my brief"
- User clicks the "Process all annotations" button inside the "Today's Brief" artifact (which calls `sendPrompt("Process the annotations in Today's Brief")`)

## What this skill is NOT for

- **Generating the brief.** That's `/brief`. This command only acts on the brief once it exists and has annotations.
- **Sending or finalizing anything.** Every action is reversible: Gmail drafts (not sends), CRM date/status updates, idempotent node touches.
- **Cross-day cleanup.** If you missed yesterday's brief, the annotations live on in `briefs/<yesterday>.md`; this command operates strictly on today's brief.

## Inputs

- Cowork artifact "Today's Brief" — v0.5.0 localStorage blob `tasks`/`annotations`/`outreach_actions` (via `mcp__cowork__read_widget_context`)
- `<config-root>/briefs/<today>.md` — the markdown twin to append action records to
- `<config-root>/memory/me/voice.md` — for drafting replies in the user's voice
- Gmail MCP, HubSpot MCP — for the side-effects (Gmail drafts, CRM task date/status updates, delegatee tasks)

## Outputs

- Gmail drafts (one per `draft_reply` annotation)
- CRM updates: due-date reschedules (`reschedule_task` / `skip`), COMPLETED (`done`), delegatee tasks (`delegate`)
- Touchpoints / pipeline updates on person/bizdev nodes (outreach `sent`/`nudge`/`booked`/`let_go`)
- `### Processed annotations` block appended to the markdown twin
- Append-only line in `<config-root>/plugins/briefing.dismissed-log.md` for each `dismiss` / `not_important`

## Routing table

**Annotations** (free-text per item):

| Annotation pattern | Action | Tool |
|---|---|---|
| "draft reply" / "reply: ..." | `draft_reply` | growth `/draft-touchpoint` or lead-engine + Gmail MCP |
| "draft outreach" | `draft_outreach` | growth `/draft-touchpoint` (fallback: lead-engine) |
| "move to tomorrow" / "move to <date>" | `reschedule_task` | HubSpot MCP |
| "skip" / "dismiss" / "I'll handle this" | `dismiss` | log only |
| Free-text / ambiguous | `clarify` | batched follow-up question in chat |

**Task actions** (`state.tasks`, v0.5.0): `done` → CRM COMPLETED · `delegate` → delegatee CRM task · `skip` → defer due date · `not_important` → log for `/end-day` suppression.

**Outreach actions** (`state.outreach_actions`, v0.5.0): `sent`/`nudge` → log touch (+ bucket/value-add/signal) · `booked` → advance stage + prep task · `let_go`/`dead` → log + remove from queue.

Durable memory write-backs + `surfacing-prefs.md` suppression learning are owned by `/end-day` Step 2c, not this command.

## Two-stage triage

Classifier (Haiku-class): one cheap pass over all annotations to assign normalized actions. Synthesis (Sonnet): only items requiring drafting (`draft_reply`, `draft_outreach`) go through full synthesis. Routing-only items (`reschedule_task`, `dismiss`) and the task/outreach action write-backs skip Sonnet entirely.

## Failure modes

- No artifact found → "Run `/brief` first."
- No annotations on the artifact → "Nothing to process. The brief is ready for annotation."
- Cowork artifact tools not available (Claude Code) → stops with a Claude-Code-specific path: edit the markdown snapshot directly, invoke draft / update commands manually
- Reschedule date >14 days out → asks user to confirm inline before writing to CRM
