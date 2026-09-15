---
type: llm
weight: 1
---

The user asked for today's rundown in plain language, never typing `/brief`. A successful response recognizes this as a request for the briefing workflow and builds or references today's brief — calendar, priority tasks, outreach queue, or the working-surface artifact — without asking the user to type the explicit command first. A failing response ignores the request, asks the user to run `/brief` themselves, or gives a generic answer with no reference to calendar/tasks/outreach content.
