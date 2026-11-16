---
max_turns: 10
allowed_tools: [Read, Glob, Grep, Skill]
append_system_prompt: |
  Eval fixture: the Nucleus config root for this run is the `cortex-root` folder in the
  current working directory. It was resolved for you; a shell is not available, so do
  not stop because `root-status` cannot be run. Read what you need from that folder.
---

What's on today? Give me the rundown.
