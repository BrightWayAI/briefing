---
name: setup-plan
description: Configure the /plan-tomorrow command (now hosted in the daily-brief plugin as of v0.2.0) for your CRM, working hours, calendar conventions, and companion-plugin integrations. Auto-fires on "/setup-plan", "set up plan-tomorrow", "configure daily planning", or when /plan-tomorrow reports user-context.md is missing. The standalone plan-tomorrow plugin is deprecated; this skill lives in daily-brief now.
---

<!-- OPENAI-ADAPTER:START -->
## OpenAI host binding

Before acting, read `../../references/openai-portability.md`. That file translates
host-specific tools, agents, artifacts, scheduling, connectors, and config-root
access for ChatGPT and Codex. It overrides concrete Claude/Cowork tool names only;
the workflow, safety gates, and output contract in this skill remain canonical.
<!-- OPENAI-ADAPTER:END -->


See `commands/setup-plan.md` for the full interview.

## When this skill fires

- User runs `/setup-plan` directly
- User says: "set up plan-tomorrow", "configure my daily planning", "configure plan-tomorrow"
- The `/plan-tomorrow` command reports user-context.md is missing → auto-route here
- User installs the plugin and asks "how do I use this?"

## Quick path

If the user wants minimum-viable defaults to start: write a placeholder `<config-root>/plugins/plan-tomorrow.user-context.md` with working hours 9–5, CRM "none," no companion plugins. The planner will work but will be limited to calendar + Gmail + manual user input. Recommend running the full interview when ready.
