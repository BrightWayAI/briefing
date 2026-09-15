# OpenAI portability contract — Today's Brief

This file binds the plugin's canonical Claude-oriented examples to ChatGPT and Codex.
It changes tool names and unavailable-host behavior, not the workflow's business logic
or safety gates.

## Shared config root

Resolve `<config-root>` with the same chain used by Cortex, in this exact order:

1. `CORTEX_CONFIG_ROOT` environment variable, when set to a non-empty path.
2. First non-empty line of `~/.cortex/config-root`.
3. First non-empty line of legacy `~/Documents/.claude-plugin-config-root`.
4. Default `~/Documents/Claude`.

An explicit path supplied for the current workflow may be used for that invocation,
but do not create a GPT-specific pointer. Expand `~`, use an absolute path, and ask for
filesystem permission when the resolved root is outside the host's writable roots.
If a new OpenAI-host user chooses a persistent root, configure it through Cortex or
write `~/.cortex/config-root` only after confirmation. Never overwrite a pointer that
targets a different root without a second explicit confirmation, and do not create or
update the legacy Claude pointer from an OpenAI host.
Identity and voice remain shared files at `<config-root>/memory/me/identity.md` and
`<config-root>/memory/me/voice.md`. Plugin state belongs under `<config-root>`—normally
`<config-root>/plugins/`—never inside the installed plugin directory.

## Invocation

- ChatGPT desktop: enable the plugin in a chat, then ask naturally or use `@Today's Brief <request>`.
- Codex: ask naturally or invoke the namespaced Agent Skill shown by the client.
- Claude slash commands remain aliases in prose. `/example` means the matching skill
  workflow; it does not require an OpenAI slash-command feature.

## Capability translation

| Canonical intent | ChatGPT / Codex behavior |
|---|---|
| Read or write local files | Use Local Work or sandbox filesystem access scoped to `<config-root>`. Request the smallest additional writable root needed. |
| Read a connector | Use an installed ChatGPT app or MCP server. Check availability first; never infer that a named Claude/Cowork connector exists. |
| Write through a connector | Preview the exact mutation and obtain confirmation immediately before it. Keep draft-only steps as drafts. |
| Web research | Use current web search with citations. If unavailable, ask for sources or stop the fresh-research portion. |
| Delegate to an agent | Use the matching read-only `.codex/agents` role when available; otherwise follow the role inline. |
| Create a Cowork artifact | Produce the same information as Markdown or a supported document artifact. Do not claim Cowork interactivity or localStorage state. |
| Register a schedule | Use a host scheduler only when exposed and after confirmation. Otherwise return a schedule definition for manual setup. |
| Select a Claude model tier | Preserve the intent (fast/low-cost vs. deep synthesis) using the host's available model; ignore Claude model names. |

ChatGPT desktop Local Work and Codex can share the exact same local files as Claude
when they resolve the same `<config-root>`. ChatGPT web/cloud sessions cannot access a
private local folder merely because this plugin is installed. They need an approved
remote app/MCP bridge; otherwise local-memory operations are unavailable, not silently
redirected to another store.

## Plugin-specific degradation

Cowork's interactive artifact is not portable. In ChatGPT and Codex, write a Markdown snapshot and use pasted or local annotation state. Missing connectors are listed and skipped; no mail, CRM, or calendar mutation occurs without confirmation.

Always report unavailable or skipped capabilities in the result. A degraded run must
remain useful where possible, but it must never imply that missing data was read or an
external action happened.
