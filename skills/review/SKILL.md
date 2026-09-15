---
name: review
description: Generate a synthesized weekly review of activity and learning across all projects — what moved forward, what was learned, what's stuck, what's coming up. Auto-fires on "/review", "weekly review", "summarize my week", or any phrase asking for a synthesized digest (as opposed to a raw chronology — that's `timeline`). Moved here from the `cortex` plugin 2026-09-15.
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->

See `commands/review.md` for the full workflow.

## When this skill fires

- User runs `/review` directly
- User says: "weekly review", "summarize my week", "what moved forward this week", "give me the digest"
- `cortex`'s `/end-week` Step 3 invokes it (as `briefing:review`) when `briefing` is installed

## What this skill is NOT for

- **Raw chronology** — that's this plugin's `timeline` skill.
- **Today's working surface** — that's `/brief`.
- **Stack health / visual dashboard** — that's this plugin's `dashboard` skill.
- **Memory hygiene** — that's cortex's `/cleanup`.

## Inputs

- `<config-root>/memory/DASHBOARD.md` — overview (cortex)
- Node files updated within the review period — LOG entries, knowledge entries, SIGNAL entries, current SUMMARYs

## Outputs

- Rendered weekly digest in chat: Progress / Learned / Stuck / Decided / Coming Up / Connections

## Failure modes

- `<config-root>/memory/` not accessible → explain the review can't be generated and stop
- Cortex not installed / memory empty → sections render "not available" rather than failing the whole digest
