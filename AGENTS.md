# Daily Brief — OpenAI host entrypoint

This repository supports Claude Code/Cowork, ChatGPT desktop Local Work, and Codex
from one canonical workflow source.

Read in this order:

1. `references/openai-portability.md` for host capability and degradation rules.
2. The matching `skills/<name>/SKILL.md` for workflow instructions.
3. Any `commands/<name>.md` and references named by that skill.

Treat `commands/` and authored `skills/` as canonical. OpenAI alias skills are thin
entrypoints and must not fork workflow behavior. Treat the installed plugin directory
as read-only at runtime. User state belongs under the shared `<config-root>` resolved
by the precedence chain in the portability reference.
Never fabricate connector data. Keep drafts as drafts, and confirm any external write,
send, schedule registration, or destructive action at the point of action.
