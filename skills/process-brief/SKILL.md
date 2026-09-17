---
name: process-brief
description: Read the actions + annotations the user logged into today's brief state (canonical v0.7.0 — tasks/annotations/outreach_actions/reflection), classify each, and route it — draft reply via Gmail, move CRM task due date, dismiss; stage task actions (done→COMPLETED, delegate→delegatee task, skip→defer) and outreach actions (sent/nudge→touch, booked→prep task, let_go→close). Auto-fires on "/process-brief", "process my brief", "act on my annotations", "follow up on my brief". Intra-day actor; durable memory write-backs + suppression learning happen overnight in cortex /listen Step 1.5, or synchronously in optional /end-day Step 2c. Drafts only — never sends.
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

- `<config-root>/briefs/<today>.state.json` — canonical v0.7.0 state (`tasks`/`annotations`/`outreach_actions`/`reflection`), populated by the artifact mirror, sync paste, or explicit ChatGPT/Codex actions in chat
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

**Task actions** (`state.tasks`, v0.7.0): `done` → CRM COMPLETED · `delegate` → delegatee CRM task · `skip` → defer to `return_on` · `not_important` → log for `/listen` suppression learning.

**Outreach actions** (`state.outreach_actions`, v0.7.0): `sent`/`nudge` → log touch (+ bucket/value-add/signal) · `skip` → defer to `return_on` · `booked` → advance stage + prep task · `let_go`/`dead` → log + remove from queue.

Durable memory write-backs, `surfacing-prefs.md` learning, snooze-ledger updates, and reflection carry-forward are owned by cortex `/listen` Step 1.5 and reviewed in `/morning`. Optional `/end-day` applies the same explicit state synchronously and writes the processed marker so `/listen` does not duplicate it.

## Two-stage triage

Classifier (Haiku-class): one cheap pass over all annotations to assign normalized actions. Synthesis (Sonnet): only items requiring drafting (`draft_reply`, `draft_outreach`) go through full synthesis. Routing-only items (`reschedule_task`, `dismiss`) and the task/outreach action write-backs skip Sonnet entirely.

## Failure modes

- No artifact or Markdown twin found → "Run `/brief` first."
- No annotations on the artifact → "Nothing to process. The brief is ready for annotation."
- Interactive artifact unavailable → read the local v0.7.0 state written from explicit chat actions; if neither it nor pasted state exists, explain how to record the action against a stable id instead of implying state was read
- Reschedule date >14 days out → asks user to confirm inline before writing to CRM
