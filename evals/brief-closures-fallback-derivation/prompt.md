---
max_turns: 10
allowed_tools: [Read, Glob, Grep, Skill]
---

Yesterday's `<config-root>/briefs/2026-09-17.closures.json` is missing (nightly-listen
didn't run), but `<config-root>/briefs/2026-09-17.state.json` exists with this shape:

```json
{
  "schema_version": "0.7.0",
  "tasks": {
    "task-1": {"action": "done"}, "task-2": {"action": "done"}, "task-3": {"action": "done"},
    "task-4": {"action": "done"}, "task-5": {"action": "done"}, "task-6": {"action": "done"},
    "task-7": {"action": "done"},
    "task-8": {"action": "skip", "return_on": "2026-09-25"}
  },
  "outreach_actions": {
    "outreach-1": {"action": "sent"}, "outreach-2": {"action": "sent"}, "outreach-3": {"action": "sent"},
    "outreach-4": {"action": "sent"}, "outreach-5": {"action": "sent"}
  },
  "annotations": {
    "event-10": "I still need to send the follow-up deck to them",
    "event-11": "nothing for me to do here, don't mine this relationship"
  },
  "reflection": {"biggest": "shipped the SOW draft", "blocked": "waiting on legal", "one_thing": "close the Acme contract"}
}
```

Walk `/brief` Step 0D0's fallback derivation against this state and tell me: what
gets dropped, what stays hidden until 2026-09-25, what becomes a carried task and
from which annotation, what does NOT become a task, and what Center of Gravity
resolves to.
