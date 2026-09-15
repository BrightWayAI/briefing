---
description: Visual dashboard of your Nucleus stack — installed plugins, recent activity, cortex memory health, outreach pipeline, time + invoicing. Renders as a Cowork artifact (HTML) you can keep open and refresh, or as a markdown snapshot in Claude Code. For a terse text status check in chat use `/status`; for troubleshooting use `/diagnose`.
---

# /dashboard

Rich visual surface for your Nucleus stack. Re-renders on demand. Single live Cowork artifact titled "Nucleus Dashboard" — open it, glance at it, refresh as needed.

Sibling to `/status` (terse text) and `/diagnose` (troubleshooting). This is the "show me what my system is doing this week" command.

---

## Step 0 — Resolve config root + load context

Resolve `<config-root>` using the shared precedence chain: explicit override,
`CORTEX_CONFIG_ROOT`, `~/.cortex/config-root`, legacy pointer, then default. Request
filesystem access only for the resolved directory when the host requires it.

- **Pointer missing** → stop with: "No plugin config root configured. Run any plugin's `/setup-*` command first."
- **Pointer exists** → read line 1 → that's `<config-root>`. Ensure access to it. Continue.

Determine `today_local` from `<config-root>/memory/me/identity.md` time zone (fallback: system local).

---

## Step 1 — Gather data (read-only, parallel where possible)

The dashboard pulls from these sources. Skip any source whose dependency is missing — that section renders with a clean "not available" placeholder rather than failing the whole render.

### Section 1: Stack overview

Same data sources as `/status` Step 1:
- Shared config files (identity, voice, DASHBOARD, decay-config, note-sources, scope-migration marker)
- Per-plugin user-context state from `<config-root>/plugins/*.user-context.md`
- Connector availability (MCP tool-name presence — no calls)
- `~/.brightway-state/agent-log.jsonl` last-run timestamps

### Section 2: This week's activity

Read `~/.brightway-state/agent-log.jsonl` filtered to last 7 days. Aggregate:

- **Total runs** across all skills
- **Top 10 skills by run count**, sorted desc
- **Daily runs histogram** — count per day for the last 7 days (for a small bar chart)
- **Skills that haven't run in 14+ days** — surface as "rusty" (signal that plugin may be misconfigured or unused)

### Section 3: Cortex memory health (v4.4+)

If `<config-root>/memory/DASHBOARD.md` exists:

- **Node count** by type (project / domain / person / archive)
- **Knowledge entry state distribution** — sample-scan up to 50 entries across nodes; classify each as Fresh / Stale / Dormant / Cold / Demoted via the decay model. Report percentages.
- **Person pages** — active count, cooling count, dormant count, archived count
- **Last `/rehearse` run** — from agent log or `.rehearse-skip-log.md`
- **Triage log** — recent entries from `<config-root>/memory/triage-log.md` (last 7 days, count of commit vs. skip decisions)

### Section 4: Relationship pipeline (if growth is installed)

If `<config-root>/relationships/pipeline.md` exists, parse it for:

- **Active signals** count
- **Drafts written this week** (from `<config-root>/relationships/sent-log.md`)
- **Drafts sent this week**
- **Drafts → sent ratio** (this week)
- **Replies this week**
- **Booked calls this week**

Skip cleanly if growth isn't installed.

### Section 5: Time + invoicing (if admin installed)

If `<config-root>/time-log.csv` exists, parse it for:

- **Hours logged this month** total
- **Hours by client** (top 5)
- **Billable vs. non-billable** split
- **Days since last invoice generated** (from `<config-root>/plugins/admin.user-context.md` invoice history, or file mtime of most recent generated invoice)

Skip cleanly if admin isn't installed.

### Section 6 (placeholder): Impact loop

Reserve a card slot for v1.1 impact metrics (drafted-to-sent ratios across all drafting plugins, mining proposal accept rate, P0 completion rate). For v1 render the card with a one-liner: "Coming after one week of telemetry. /agent-metrics has interim data."

Skip cleanly if `ops`'s agent-log doesn't yet have 7+ days of data.

---

## Step 2 — Render the artifact

Load the HTML template from `references/nucleus-dashboard-template.html` (bundled with the ops plugin source). Substitute placeholders with today's data.

Each section is a card with a header, body, and (where relevant) a small sparkline or bar chart rendered inline via SVG. No external libraries; pure inline HTML/SVG.

Decide create vs. update:

1. Call `mcp__cowork__list_artifacts` to look for an existing artifact titled exactly "Nucleus Dashboard."
2. If found → call `mcp__cowork__update_artifact` with the new HTML content. Keep the same artifact (one live dashboard, not a per-day rotation).
3. If not found → call `mcp__cowork__create_artifact` with title "Nucleus Dashboard," `artifact_type: "html"`, content = the rendered template, metadata `{plugin: "ops", last_rendered: <ISO timestamp>}`.

In Claude Code (no Cowork artifact tools available): write a markdown snapshot to `<config-root>/nucleus-dashboard.md` and print the path to chat. Same sections, markdown format, no SVG charts (just text-based bar representations like `▇▇▇▇░░░░`).

---

## Step 3 — Notify the user

Output a single short chat message:

> "Nucleus Dashboard updated. Open the 'Nucleus Dashboard' artifact to view. Re-run `/dashboard` any time to refresh."

In Claude Code:

> "Nucleus Dashboard markdown snapshot updated at `<config-root>/nucleus-dashboard.md`."

Don't dump the dashboard content into chat. The whole point is that it lives as a persistent artifact / file, not as scroll-bound chat output.

---

## Behavior rules

- **Read-only across everything.** No file writes outside the dashboard's own artifact + markdown snapshot. No telemetry log entry for this command (avoid the meta-recursion of "dashboard run" appearing in the dashboard's own activity list).
- **Graceful degradation.** Every section is independent. If a data source fails, that section renders an honest "not available" placeholder rather than failing the whole render.
- **Cap data reads.** Per-section caps: 50 memory entries (sampled), 100 agent-log lines (last 7 days), 200 time-log rows. Don't read every line of multi-month logs.
- **Fast.** Target < 5 seconds wall time. Filesystem reads + light parsing, no model calls.
- **Idempotent.** Re-running on the same minute produces the same content (modulo timestamps in the data).
- **Cost: zero model calls in v1.** Pure data + template rendering.

## What this command is NOT for

- **Quick "is everything working" check** — that's `/status`. Use this for the richer view.
- **Per-plugin troubleshooting** — that's `/diagnose`.
- **Memory triage decisions** — that's `/recall` recall-time actions or `/rehearse`.
- **Editing config** — that's per-plugin `/setup-*` commands.

## Versioning

This is v1. v1.1 will add the Impact loop section (drafted-to-sent ratios, mining accept rate, P0 completion rate) once a week of telemetry is available to populate it sensibly.
