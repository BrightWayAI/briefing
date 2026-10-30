# OpenAI portability contract — Today's Brief

This file binds the plugin's canonical Claude-oriented examples to ChatGPT and Codex.
It changes tool names and unavailable-host behavior, not the workflow's business logic
or safety gates.

## Shared config root

The authority is Cortex `references/core-contract.md` §1 and its
`scripts/lib/config_root.py` resolver. Locate the installed Cortex plugin rather
than assuming it is the current directory. The resolution order is:

1. Explicit workflow override or a loaded project `.cortex.json` `config_root`.
2. `CORTEX_CONFIG_ROOT` environment variable, when set to a non-empty path.
3. `~/.cortex/config-root` pointer file.
4. Legacy `~/Documents/.claude-plugin-config-root` pointer file.
5. `~/cortex-root`, only when materialized (contains `.cortex-repos.json`).
6. `~/Documents/Claude`, the backward-compatible runtime fallback.

New-user setup recommends `~/cortex-root`; that setup default does not change
runtime precedence. A malformed higher-priority pointer is an error, not a reason
to fall through. After resolution, verify that the root is reachable before reading
or writing state. An unreachable root is not an empty memory or a first-run install:
report it and stop dependent steps. With Cortex 4.30.0+, use its read-only
`python3 scripts/cortex_cli.py root-status` (pass `--root` for an explicit workflow
override). Otherwise verify that the resolved directory and its `memory/` exist.
Never create a replacement root, copy a Mac path
into a cloud filesystem, or change the pointer to hide an access failure.

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

## Cloud bootstrap and sync

Follow Cortex `references/core-contract.md` §11B and the `workspace.bootstrap` /
`sync.git` binding in `references/capability-matrix.md`. On a filesystem-capable
cloud host with no reachable root, discover the designated org and private checkouts
by their `.cortex-role` markers and use the existing Cortex `scripts/cortex_cli.py
bootstrap --print-env`. Use an explicit invocation root for that host if a pointer
refers to another machine; do not silently overwrite the persistent pointer.
If the code, designated repositories, credentials, or filesystem permissions are
unavailable, report the missing capability and leave dependent work unapplied.

Pull before approved writes and sync afterward through Cortex's existing CLI.
Inspect both checkouts first: `sync` commits outstanding changes in both repos;
do not include unrelated changes in a test or a narrowly authorized write.
Preserve canonical mutation gates, shared locking/atomic writes, and §11A's
multi-actor proposal boundary. Keep private paths in the private checkout; never
force-push or resolve a sync conflict by overwriting memory. Report sync failures.
A verified local host sync job may perform the push when the agent lacks network
access; distinguish a saved local write from confirmed remote synchronization.

An MCP-only host must use operations actually exposed by its Cortex bridge.
Do not assume shell access or invent bootstrap/sync MCP tools. If lifecycle
operations are unavailable, require host-side bootstrap/sync and report them
unverified until a real receipt is available. A scheduled job must verify its own
root, tools, and synchronization; another chat's successful setup is not proof.

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

ChatGPT desktop Local Work and local Codex can share the same files as Claude when
they resolve the same `<config-root>`. A cloud filesystem host needs its own synced
checkout; an MCP-only host needs an authorized bridge to a reachable root. Installing
a plugin alone grants neither access to a Mac folder nor GitHub write permission.

## Plugin-specific degradation

Cowork's interactive artifact is not portable. In ChatGPT and Codex, write a Markdown snapshot and preserve user-supplied actions in the canonical local v0.7.0 state file so cortex /listen can mine the same round-trip. Missing connectors are listed and skipped; no mail, CRM, or calendar mutation occurs without confirmation.

### Chat-native brief state on OpenAI hosts

When no interactive artifact is available, render the Markdown brief with the same
stable task, outreach, and event ids used by the canonical artifact. If the user
marks an item done, delegates it, snoozes it, changes its priority, adds an
annotation, or supplies today's reflection in chat, merge only those explicit
choices into `<config-root>/briefs/<YYYY-MM-DD>.state.json` using the canonical
v0.7.0 shape from `commands/brief.md`. Preserve existing keys and update
`last_interaction_at`; never infer a click or disposition from silence.

This local state file is the OpenAI-host equivalent of the artifact's state mirror.
`/process-brief` may act on it during the day, and cortex `/listen` mines it
overnight before `/morning` reviews durable memory proposals. If no explicit state
was recorded, `/listen` may still run its evidence-based inference pass against the
Markdown twin and available archive sources.

Always report unavailable or skipped capabilities in the result. A degraded run must
remain useful where possible, but it must never imply that missing data was read or an
external action happened.
