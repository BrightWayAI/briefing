---
type: llm
weight: 1
---

The scenario has a missing closures.json but a present yesterday.state.json, which
`/brief` Step 0D0's fallback derivation must turn into a synthetic closures record. A
successful response:

- Drops all 7 `done` tasks and all 5 `sent` outreach items (they're closed).
- Keeps `task-8` (skip, return_on 2026-09-25) hidden/snoozed until that date rather
  than rendering it today.
- Turns the `event-10` annotation ("I still need to send the follow-up deck") into a
  carried task/proposal tied to that event id — and notes it would render as a
  priority row (reasonably describing it as P1-ish carried work is fine; exact
  priority label isn't required).
- Explicitly does NOT turn `event-11` ("nothing for me to do here, don't mine this
  relationship") into any task — recognizes it as a suppression/disposition marker,
  not an action.
- States that Center of Gravity resolves to the reflection's `one_thing`: "close the
  Acme contract."
- Ideally mentions the derived closures.json would be tagged `written_by: "/brief
  fallback"`.

A failing response treats all items as still open (fails to close done/sent items),
surfaces the skipped task before its return date, misses or wrongly stages the
"nothing to do" annotation as a task, fails to carry the real action annotation, or
gets Center of Gravity wrong (e.g. picks a P0 task instead of the reflection).
