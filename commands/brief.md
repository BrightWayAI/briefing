---
description: Generate or refresh today's daily brief as the persistent Cowork artifact "Today's Brief" (stable id `todays-brief`). Renders 6 fixed sections — Center of Gravity, Calendar Block (visual timeline + written list with per-meeting notes), Priority Tasks (richer actions), Outreach Queue (actions + category tags + annotation + skip duration), Yesterday's Reflection (read-only), Today's Reflection (editable). Filters everything against `memory/me/surfacing-prefs.md`, yesterday's closures, and the snooze ledger before render. Writes a markdown twin to `<config-root>/briefs/YYYY-MM-DD.md`. Run again any time to refresh; state persists in localStorage `brief-YYYY-MM-DD` (schema_version 0.7.0) and mirrors to `briefs/YYYY-MM-DD.state.json`, mined nightly by cortex `/listen`. `/brief --tomorrow [date?]` dispatches to calendar-first next-day planning (see Step -1) — same job the retired standalone `/plan-tomorrow` command did.
---

# /brief

Builds today's working surface. The output is two coordinated things:

1. A **markdown twin** for audit at `<config-root>/briefs/YYYY-MM-DD.md` (the canonical text record).
2. A live Cowork **artifact** with stable id `todays-brief` — the working surface, with richer per-item actions that persist to localStorage and get mined overnight by cortex `/listen`.

This command is **read-only across all sources**. It does not draft replies, modify CRM tasks, or send anything. Acting on annotations happens in `/process-brief`; the durable write-backs (closing source-node actions, suppression learning, snooze ledger, reflection) happen in `/listen` Step 1.5, reviewed the next morning via `/morning`. None of this depends on `/end-day`, which is optional.

**The brief is a working surface, not a read-only snapshot (v0.5.0).** Every interactive action writes to one localStorage blob keyed `brief-YYYY-MM-DD` and is mined by `/listen` Step 1.5.

---

## Step -1 — Mode dispatch

If invoked as `/brief --tomorrow` (or the user's natural-language request is clearly "plan tomorrow" / "block my day" / "plan next business day" rather than "show me today"), this is a **different job**, not a variant render of the same artifact: it writes new calendar events instead of rendering a read-only status surface. Delegate the entire request to `commands/plan-tomorrow.md` and follow that command's workflow exactly (Step 0 onward) — do not attempt to reuse this command's artifact-rendering steps for it. `commands/plan-tomorrow.md` is the canonical procedure; this dispatch is a packaging change so `/brief --tomorrow` is the single documented entry point, folding in the formerly-standalone `/plan-tomorrow` command (2026-09-15).

Otherwise (no `--tomorrow` flag, no next-day-planning phrasing), continue with today's brief below.

---

## Step 0 — Resolve plugin config root and load context

### A — Resolve `<config-root>`

Resolve explicit override → `CORTEX_CONFIG_ROOT` → `~/.cortex/config-root` →
legacy pointer → default. Request access only to the resolved root in Cowork. A
malformed higher-priority pointer is an error. If the resolved root is not
accessible, prompt the user to run `/setup-brief` and stop.

### B — Load plugin config

Read `<config-root>/plugins/briefing.user-context.md`.

- **Exists** → parse `sections_enabled`, sort defaults, empty-state behavior, annotation placeholders.
- **Missing** → ask: "I don't see your briefing config. Want to run `/setup-brief` first, or use defaults?" Default to "use defaults" if the user is in a hurry.

If `brief_enabled` is `false`, stop with: "Daily brief is disabled. Re-enable in `<config-root>/plugins/briefing.user-context.md`."

### C — Load shared identity

Read `<config-root>/memory/me/identity.md` for time zone (defines "today") and tool inventory (decides which sections will have data).

### A2 — Artifact-db preflight (v0.8.0 — moved ahead of D0)

Before reading closures/snooze state, check whether a prior publish left
`<config-root>/briefs/.artifact-runtime.json` (written by Step 3.0 on every hosted
publish). If it names an artifact with `capability: "db"`, read document
`briefs/<yesterday_local>` from that artifact's database (per the runtime file's
`collection` and `doc_id_pattern`) and write it to
`<config-root>/briefs/<yesterday_local>.state.json` before continuing — this is what
gives Step D0's closures/fallback pass real data to read when yesterday's brief lived
only in a hosted claude.ai artifact's browser tab. No-op if `.artifact-runtime.json`
is absent, names no `db` capability, or the read fails (log the failure and continue
with whatever `<yesterday_local>.state.json` already exists on disk — never block on
this). This preflight is intentionally the same one Step 3.0 documents for the
hosted-artifact case; it now runs here, in Step 0, ahead of D0, rather than being
folded into Step 3.0's own narrative, so the closures pass always has the freshest
data available before render instead of only when `/brief` happens to reach Step 3.

### D0 — Read closures + snooze ledger (v0.7.0 — REQUIRED before render)

Before pulling any live source, read three files that describe what already happened to yesterday's brief:

1. `<config-root>/briefs/<yesterday_local>.closures.json` (written by `/listen` Step 1.5, or absent if `/listen` hasn't run or found nothing). If present, parse `closed`, `carried`, `snoozed`, `suppressed`, `annotations`.
2. `<config-root>/briefs/.snooze-ledger.json` — the canonical snooze ledger (see `/listen` Step 1.5h). Entries keyed by brief item id: `{title, kind, node, return_on, skipped_on, skip_count, last_detail}`.
3. If `<config-root>/relationships/today.json` is present, it still drives the outreach queue as today's live pull (Step 1). If it's **missing**, build the outreach queue from: snooze-ledger returns (kind=outreach) + unclosed outreach items from yesterday's brief + person/bizdev node open loops, and say so in the twin footer ("today.json not found. Outreach built from carryover + snooze returns").

**Fallback derivation (v0.8.0) — when `.closures.json` is missing but state exists.** If `<yesterday_local>.closures.json` is missing but `<yesterday_local>.state.json` exists on disk (written by the Step 0.A2 artifact-db preflight below, or mirrored directly by desktop Cowork), derive the closures pass directly from that state blob instead of proceeding with empty sets — this is what keeps the brief filtered even on a night `/listen` never ran:
- `closed` = tasks with `action` in `done`/`not_important`; outreach with `action` in `sent`/`nudge`/`booked`/`let_go`.
- `snoozed` = any task/outreach entry with `return_on > today_local`.
- `carried` = annotation entries that read as containing an action (non-empty free text that isn't purely a "nothing to do" disposition — see `/listen` Step 1.5's carried-task semantics for the same judgment call); `reflection.one_thing` feeds Center of Gravity as usual (Step 1's Center of Gravity logic already reads yesterday's `## Reflection`, which this fallback does not need to duplicate).
- `reflection` = `state.reflection` verbatim.
Write `<yesterday_local>.closures.json` with this derived shape plus `"written_by": "/brief fallback"` so downstream readers (including `/listen`, which treats an existing closures.json as authoritative and will not overwrite it — see `/listen` Step 1.5e) know this record didn't come from the nightly pipeline. If the artifact-db doc for yesterday can be read instead of a local state.json, use that as the source for the same derivation.

If neither `.closures.json` nor `<yesterday_local>.state.json` (nor a readable artifact-db doc) exists at all, proceed with empty sets as before, but the Step 2 markdown twin footer must say so explicitly (see Step 2) rather than silently rendering unfiltered content — this is the signal that `nightly-listen` may not be running.

Any of these three files being absent is otherwise normal on a fresh install or the first day — proceed with empty sets, don't treat it as an error.

### D — Load surfacing preferences (v0.5.0 — REQUIRED filter)

Read `<config-root>/memory/me/surfacing-prefs.md`. This is the canonical suppression store (written by `/listen` Step 1.5 from `not_important` actions and the repeat-ignore rule, or by `/end-day` Step 2c when the user still runs it). Parse:

- **Do-not-resurface list** — explicit per-item suppressions. An item whose title/source matches a suppressed entry MUST NOT be rendered as a priority task or outreach item, unless its linked re-surface condition has flipped.
- **Surfacing rules** — noise classes (admin/finance dunning, vendor cert/onboarding nudges). Items matching a noise class are demoted (never P0/P1); route to a low-priority "admin" mention at most, or drop.

If `surfacing-prefs.md` is missing, proceed without filtering but note it once: "No surfacing-prefs.md found. Brief is unfiltered. It'll be created the first time you mark something 'not important' and `/listen` mines it overnight."

### E — Resolve dates

`today_local = current date in user's time zone from identity.md` (fallback: system local date). Format `YYYY-MM-DD`.
`yesterday_local = today_local − 1 calendar day`.

When `/end-day` invokes this command to pre-stage, it passes `target_date = tomorrow_local`; use that in place of `today_local` everywhere below (and `target_date − 1` for the reflection read).

---

## Step 1 — Pull source data (in parallel where possible), then filter

Pull each section's source, then **apply the Step 0D0 closures/ledger pass, then the Step 0D surfacing-prefs filter, before anything is rendered**.

**Closures/ledger pass (v0.7.0):** for every task/outreach candidate pulled from a live source:
- If its id appears in `closed` or `suppressed` (from yesterday's `.closures.json`) → drop it. It's done or dead; don't re-render it.
- If its id appears in the snooze ledger with `return_on > today_local` → drop it, but count it toward the footer's "N snoozed."
- If its id appears in the snooze ledger with `return_on <= today_local` → include it in the **Today** tier regardless of what the live source says (the ledger is itself a source — a live source that forgot about it doesn't override a legitimate return), tag its row sub-line "back from snooze (skipped `<skipped_on>`, `<skip_count>`x)", and clear its ledger entry once it has rendered (not before — a render failure shouldn't silently lose the return).
- If its id appears in `annotations` from `.closures.json` and the live source still carries it (carried-forward item), show the annotation text in the row sub-line.

Suppressed/noise items (surfacing-prefs) are dropped silently; log the count for the markdown twin's footer, alongside the closures/snooze counts: "N items filtered by surfacing-prefs · Closed since yesterday: `<list>` · N snoozed · N returned".

### Center of Gravity (section 1)

The single most important thing for the day, one or two sentences. Derive from, in order of preference:
1. Yesterday's reflection "the one thing tomorrow has to move" (read from `briefs/<yesterday_local>.md` `## Reflection`).
2. The top P0 priority task after filtering.
Phrase it as a directive sentence. Add upstream context if it sharpens urgency ("X start ~July 1 creates pull"). Not interactive.

### Calendar Block (section 2)

Pull today's calendar via the connected calendar MCP. For each event, capture: title, start, end, location/video link, attendees (excluding you). Cap at 12.

For each external attendee, attempt to find a cortex node and pull one line of context:
- `<config-root>/memory/person/<firstname>-<lastname>.md` → most recent interaction, open commitments, "why this matters / last touch."
- Fall back to `<config-root>/memory/client/<slug>.md` if the company appears in a client node, or the event description.

Two coordinated views are rendered (Step 3):
- **Visual timeline strip** — 8a–6p horizontal scale, one block per event positioned by time, color-coded `meeting` / `focus` / `personal` / `open`.
- **Written block list** — time, title, attendees, and **notes where available** (the one-line context above; "why this matters / last touch" for known person/client nodes).

Meetings are **read-only context cards** (no actions). If zero meetings, hide the whole calendar card.

### Priority Tasks (section 3)

Query CRM (HubSpot MCP if available; otherwise render "CRM not connected" placeholder):
- Tasks where `owner = current user`, `due_date <= today_local`, `status != completed`.
- **Keep P0 and P1 by default.** P2 and lower are excluded to reduce clutter (spec A.1 §3) — **except** P2 items the user has explicitly approved for the brief (e.g. carried forward from a prior `/end-day` tomorrow-priority approval, or reprioritized to P2 in the artifact rather than dropped). Approved P2 rows render with the `p2` tag styling. Never auto-promote unapproved P2s.
- **Filter against surfacing-prefs** (Step 0D): drop suppressed items; demote noise-class items out of the priority card.
- Sort: priority desc, then due_date asc. Cap at 12.

For each surviving task capture: stable `task-<id>` id, title, due date, related contact/deal, priority (P0/P1/approved-P2). Each row renders the richer action set (Step A.2: done / delegate / skip / not_important / annotate) plus a per-row **priority toggle** (P0/P1/P2) that writes a `reprioritize` into the same `tasks` map (v0.6.0, D3). Set `data-priority` on the row to the rendered priority so the toggle knows its baseline. Hide the card if zero tasks survive.

### Outreach Queue (section 4)

Source from Growth (relationships) (look in `<config-root>/relationships/today.json` if
present, otherwise its current queue). Existing installs may read the legacy
`<config-root>/plugins/weekly-outreach.*` fallback only during migration and must
label it as legacy in the output. <!-- LEGACY_COMPAT -->

- Tier each contact: **Today** (scheduled/flagged today), **This week** (`due_date <= today + 7d` or weekly-cadence overdue), **Backlog** (flagged, not date-scoped).
- For each contact capture: stable `outreach-<id>` id, name, title/company, last touch, a one-line "why," and — if the pipeline entry carries one — the **signal type** (one of lead-engine's 7: Engagement / Job change / Funding / Hiring / Growth-expansion / Tech-stack change / Direct intent). The signal is stored on the row's `data-signal` so it auto-fills the tag without asking.
- **Research link (v0.5.0)** — capture a `research_url` + `link_label` for each contact so the user can open them up before reaching out. Resolve in this order:
  1. A LinkedIn profile URL stored on the contact's cortex node — check `<config-root>/memory/person/<slug>.md` and `<config-root>/memory/bizdev/<slug>.md` for a `linkedin:` field, a `## Linked entities` LinkedIn link, or a LinkedIn URL anywhere in the node. If found → `research_url = <that URL>`, `link_label = "🔗 LinkedIn"`. Also accept a LinkedIn URL carried directly on the pipeline entry.
  2. Else build a LinkedIn people-search URL from name (+ company if known): `https://www.linkedin.com/search/results/people/?keywords=<URL-encoded "Name Company">`. `link_label = "🔍 Research"`.
  3. Else (no name usable) a plain web search: `https://www.google.com/search?q=<URL-encoded "Name Company">`. `link_label = "🔍 Research"`.
  Always produce a non-empty `research_url`.
- Filter against surfacing-prefs. Hide empty tiers; hide the card if all tiers empty.

Each row renders per-contact actions (Sent / Nudge / Booked / Skip / Let go) plus an **optional** category quick-tag (bucket + value-add chips), revealed after an action is logged (decision D.3: optional/skippable, not required).

### Yesterday's Reflection (section 5)

Read `<config-root>/briefs/<yesterday_local>.md` `## Reflection` (written by `/listen` Step 1.5g from the artifact's own "Today's Reflection" card, by `/morning` Step 4.6, or by `/end-day` Step 4 when the user still runs it). Show at minimum **biggest thing done** and **the one thing that has to move today** (the latter also feeds section 1).

- Exists with `## Reflection` → render read-only.
- Exists without it → "No reflection logged yesterday. Fill in Today's reflection at the bottom of the brief."
- Missing → "No brief yesterday."

Always render this card (it's required).

---

## Step 2 — Write the markdown twin

Write the assembled content to `<config-root>/briefs/<today_local>.md`. The markdown is the canonical text record; the artifact is a render of it. Section order is fixed and matches the artifact:

```markdown
# Today's Brief — <today_local>

> Generated <ISO-8601 timestamp> · <N> items filtered by surfacing-prefs · Closed since yesterday: <list or "none"> · <N> snoozed · <N> returned

If neither `<yesterday_local>.closures.json` nor `<yesterday_local>.state.json` (nor a readable artifact-db doc) exists at all — the "neither exists" case from Step 0D0 — replace the whole footer line with: "yesterday's brief state not found — nightly-listen may not be running; run ops `/status`." Do not render the normal filtered-counts footer in that case; a fabricated "0 filtered" reads as healthy when it isn't.

## 1. Center of Gravity

<one or two sentences>

## 2. Calendar

### <time> — <title>
- **With:** <attendees>
- **Notes:** <context / why this matters / last touch — if available>

(repeat per meeting; omit section if zero)

## 3. Priority Tasks

| Task | Due | Related | Priority |
|---|---|---|---|
| <title> | <due> | <related> | P0/P1 |

(P0/P1 only; omit section if zero)

## 4. Outreach Queue

### Today
- <name> — <company> · last touch <date> · <why> [signal: <type>] · [research](<research_url>)

### This week
- ...

### Backlog
- ...

(omit empty tiers; omit section if all empty)

## 5. Yesterday's Reflection

<content from yesterday's `## Reflection`, or the appropriate placeholder>
```

Create `<config-root>/briefs/` if missing. Overwrite today's file on re-run; yesterday's file is untouched.

---

## Step 3 — Render the Cowork artifact (stable id `todays-brief`)

**Artifact identity rule:** the artifact id is ALWAYS `todays-brief`. Both `/brief` and cortex `/end-day` Step 5 `update_artifact` this same surface. Never create a parallel artifact; never produce a markdown-only fallback when Cowork is available. Formatting MUST be identical regardless of which command produced it.

### Step 3.0 — Resolve the state-mirror write tool (D2a, v0.6.1; hosted path added v0.7.0)

The artifact sandbox can only call MCP tools that are (a) fully qualified `mcp__<server>__<tool>`, (b) declared in the artifact's `mcp_tools` allowlist at create/update time, and (c) actually connected to Cowork. There is **no built-in filesystem tool** — auto-sync works only when the user has connected a filesystem-capable MCP server (see `/setup-brief` § Enable brief auto-sync).

**First, determine the runtime.** Desktop Cowork and hosted claude.ai artifacts both render the same HTML, but only desktop Cowork's `window.cowork.callMcpTool` bridge can reach a filesystem MCP server — a hosted claude.ai artifact runs in a browser tab with no MCP bridge at all, only `localStorage` (which lives in that browser and is unreachable by any scheduled or headless command).

**Desktop Cowork path (unchanged):**
1. Scan the session's available MCP tools for a file-write tool (canonical: `mcp__filesystem__write_file` from the reference filesystem server; any server exposing a write-file tool qualifies). Prefer one whose allowed roots cover `<config-root>/briefs/`.
2. **Verify it this session** (the sandbox bridge requires this discipline — never declare an unverified tool): if `<config-root>/briefs/<today_local>.state.json` is missing, write an empty v0.7.0 blob (`{"schema_version":"0.7.0","tasks":{},"annotations":{},"outreach_actions":{},"reflection":{},"tasks_checked":{}}`) through the tool; if it already exists, re-write its current content verbatim. Confirm the write landed. A side benefit: `/listen` always finds a state file, even on a zero-action day.
3. On success → `{{FS_WRITE_TOOL}}` = the verified tool name, `{{STATE_MIRROR_PATH}}` = the **absolute** path `<config-root>/briefs/<today_local>.state.json`, and pass `mcp_tools=["<tool>"]` on the Step 3 `create_artifact`/`update_artifact` call. If the verified tool's args differ from `{path, content}`, adapt the `mirrorState()` call in the rendered HTML to match what you verified.
4. On no tool found (or verify failed) → `{{FS_WRITE_TOOL}}` = `""`, `{{STATE_MIRROR_PATH}}` = the absolute path anyway (harmless), omit `mcp_tools`. Fall through to the hosted/degraded behavior below.

**Hosted claude.ai path (v0.7.0):** if the render target is a hosted claude.ai artifact (not desktop Cowork), load the **artifact-capabilities skill** before assuming there is no bridge — it documents whatever shared-state capability is actually available at render time (this changes over time; don't hardcode an API surface here). Concretely:
1. Load the artifact-capabilities skill and ask it what shared-state / persistent-storage capability, if any, this hosted artifact can call to write a blob outside its own browser tab.
2. If it reports a usable capability → wire the rendered HTML's `mirrorState()` to call it (same shape as the desktop path: write the JSON blob, treat a thrown error / error response the same as `isError`), and **write (overwriting on every publish)** `<config-root>/briefs/.artifact-runtime.json`:
   ```json
   {"artifact_url": "<url>", "capability": "db", "collection": "briefs", "doc_id_pattern": "<date>", "updated_at": "<ISO-8601 now>"}
   ```
   This is the cross-plugin contract the Step 0.A2 preflight (this command), and cortex `/listen` Step 1.5a, read to know how to reach the doc store without re-discovering the capability every time.
3. If it reports no usable capability → degrade explicitly: `{{FS_WRITE_TOOL}}` = `""`, and the rendered artifact shows the persistent manual-sync banner. Do not invent an API call that hasn't been confirmed to exist.

**Preflight (all four commands, v0.7.0; keyed off `.artifact-runtime.json` as of v0.8.0):** `/brief` (Step 0.A2, above), `/process-brief`, `/end-day`, and `/listen` (Step 1.5a) each open with a preflight step that reads `<config-root>/briefs/.artifact-runtime.json` and, when it names a hosted artifact with a discovered shared-state capability, reads that store back and writes it to `<config-root>/briefs/<date>.state.json` before doing anything else — this is what turns "state lives in a browser tab" into "state lives on disk where a headless command can see it." On desktop Cowork this preflight is a no-op (the mirror already wrote the file directly, and `.artifact-runtime.json` is never written on that path). When no capability was ever discovered, the preflight is also a no-op and the existing manual-sync-banner path is the only bridge.

Load `references/brief-artifact-template.html` (v2 layout). Substitute these tokens with today's filtered data:

**Header / meta**
- `{{DATE_LONG}}` = e.g. "Tuesday, June 9". `{{DATE_ISO}}` = `<today_local>` (drives `LS_KEY = brief-<today_local>`).
- `{{HEADER_BADGE}}` = one-line day descriptor (e.g. "Light calendar · ship the B&S plan").
- `{{FOOTER_NOTE}}` = e.g. "Run /brief to refresh · /process-brief to act on annotations · actions feed /end-day".
- `{{META_DESCRIPTION}}` = one-line artifact description. `{{META_MCP_TOOLS}}` / `{{META_MCP_SERVERS}}` = JSON arrays listing exactly the tools the HTML actually calls: `["<FS_WRITE_TOOL>"]` + its server name when Step 3.0 resolved one, else `[]` / `[]`. (The meta block only drives re-grant prompts on share/import — the live allowlist is the `mcp_tools` param from Step 3.0.) In Claude Code, drop the whole `<script id="cowork-artifact-meta">` block.
- `{{FS_WRITE_TOOL}}` / `{{STATE_MIRROR_PATH}}` = from Step 3.0.

**1. Center of Gravity** — `{{CENTER_OF_GRAVITY}}` = the one-or-two-sentence directive.

**2. Calendar** — feed the visual strip via the `<script>` constants, then repeat the written `.cal-item` block:
- `{{TL_START_HOUR}}` / `{{TL_END_HOUR}}` = the day window in whole hours (default `8` / `18`). `{{TL_WINDOW_LABEL}}` = e.g. "8a–6p".
- `{{TL_BLOCKS_JSON}}` = a JS array literal, one object per event, **decimal hours**: `[{s:9.5,e:10,label:'Automation chat',cls:'meeting'},{s:12,e:13,label:'Focus',cls:'focus'}]`. `cls` ∈ `meeting` | `focus` | `personal`. Emit `[]` if no events. (The template's `buildTimeline()` positions blocks against the window — no manual `left%`/`width%`.)
- Repeat the `.cal-item` block per event: `{{EVENT_ID}}` (stable per-event id, e.g. the calendar event id — drives the per-meeting note textarea's annotation key `event-<id>`), `{{EVENT_TIME}}` (e.g. "9:30–10:00"), `{{EVENT_TITLE}}`, `{{EVENT_WHO}}` (attendees + location), `{{EVENT_NOTES}}` (the one-line context / why-this-matters / last-touch; omit the `.cal-notes` div if no notes). If a note already exists for this event (from `annotations["event-<id>"]` in yesterday's or today's own carried-forward state), the template pre-fills the textarea on load — no extra token needed, the JS reads `state.annotations` directly.

**3. Priority Tasks** — repeat the `.task-row` block per task (P0/P1 only, post-filter):
- `data-id="task-<id>"`, `{{TASK_ID}}` = same id, `{{TASK_NAME}}` = title, `{{TASK_PRIORITY}}` = `P0`/`P1`, `{{TASK_PRIORITY_CLASS}}` = `` for P0 or ` p1` for P1, `{{TASK_SUB}}` = one-line context, `{{HINT_TASK}}` = annotation placeholder from config.

**4. Outreach Queue** — repeat the `.outreach-row` block per contact (post-filter, ordered today → this week → backlog):
- `data-id="outreach-<id>"`, `{{CONTACT_ID}}` = same id, `{{CONTACT_NAME}}`, `{{CONTACT_SUB}}` = title/company · last touch · one-line why. If the item is back from snooze (Step 0D0), append the "back from snooze" sub-line here.
- **Signal auto-fill:** in the `.oc-signal` select, emit the contact's pipeline signal as the FIRST `<option>` so it's pre-selected (fall back to `—` first if no signal). Bucket/value-add stay at their template defaults (optional — the user can change them; they're captured on action).
- `{{CONTACT_LINK}}` = the `research_url` resolved in Step 1; `{{CONTACT_LINK_LABEL}}` = `🔗 LinkedIn` when a real profile URL is known, else `🔍 Research`.
- Each row also carries an annotation textarea (same `annotations` map, keyed by `outreach-<id>`) and a Skip action that opens an inline duration mini-form (v0.7.0) instead of committing immediately — see the localStorage contract below.

**5. Reflection** — `{{YESTERDAY_LABEL}}` = e.g. "Mon 6/8"; `{{REFLECT_BIGGEST}}` / `{{REFLECT_BLOCKED}}` / `{{REFLECT_ONE_THING}}` from yesterday's `## Reflection` (use "—" for blanks). This card is read-only. **Today's Reflection** (the second card, v0.7.0) has no tokens — it's an empty editable form on first render; if today's brief already has a `state.reflection` from a prior same-day re-run, the template restores it from `state` on load like every other field.

Hide any non-required card whose content is empty by omitting its `<div class="card" data-section="...">`. Center of Gravity and Yesterday's Reflection always render. To hide the calendar strip cleanly when there are no events, emit `{{TL_BLOCKS_JSON}}` = `[]` and omit the calendar card.

### localStorage contract (canonical — schema_version 0.7.0)

ONE JSON blob per day at key `brief-<today_local>`:

```json
{
  "schema_version": "0.7.0",
  "tasks":            { "<task-id>":    { "action": "done|delegate|skip|not_important", "detail": "", "return_on": "YYYY-MM-DD (skip only)", "priority": "P0|P1|P2", "reprioritized": true, "ts": "", "name": "" } },
  "annotations":      { "<item-id>":    "free text" },
  "outreach_actions": { "<contact-id>": { "name": "", "action": "sent|nudge|skip|let_go|booked|dead", "bucket": "", "signal": "", "value_add": "", "detail": "", "return_on": "YYYY-MM-DD (skip only)", "ts": "" } },
  "reflection":       { "biggest": "", "blocked": "", "one_thing": "", "ts": "" },
  "tasks_checked":    { "<task-id>": true },
  "last_interaction_at": "ISO8601"
}
```

`tasks_checked` is a back-compat mirror — the template sets `tasks_checked[id] = (action === "done")` whenever a task action fires, so v0.4.x readers keep working. New readers use `tasks`. The UI emits outreach actions `sent|nudge|skip|let_go`; `booked`/`dead` are reader-accepted synonyms (`dead` = `let_go`) but not rendered by default. `annotations` now also includes outreach-row annotations (keyed by `outreach-<id>`, same map) and per-meeting calendar notes (keyed by `event-<id>`) — readers that only expected task/inbox ids must not assume a fixed id prefix set.

**New in 0.6.0 (D3 reprioritize):** a task row can carry an on-the-fly priority change. `reprioritize` merges `{ priority, reprioritized: true }` into the existing `tasks[id]` entry **without clobbering its `action`**. An entry may therefore have `reprioritized: true` + `priority` and **no `action`** (priority changed, no disposition yet) — readers must tolerate a missing `action`.

**New in 0.6.0 (D2a state mirror), fixed in 0.6.1, extended in 0.7.0:** because Cowork exposes no widget-context handle for *persisted* artifacts, the template mirrors this full blob to `<config-root>/briefs/<date>.state.json` on every action. **The mirror is not free:** the sandbox bridge (`window.cowork.callMcpTool`) only reaches a fully-qualified MCP tool that Step 3.0 verified and declared in the artifact's `mcp_tools` allowlist. When no writable tool resolves (desktop, no filesystem MCP connected — or hosted claude.ai with no discovered shared-state capability), the artifact shows a persistent banner steering the user to the manual **🔄 Sync brief state** button (renamed in 0.7.0 — it no longer belongs only to `/end-day`; it copies the blob and any of `/process-brief`, `/end-day`, or `/listen`'s preflight can accept the paste). This file is the canonical read path for every downstream reader — see below.

**New in 0.7.0:** `tasks[id].return_on` / `outreach_actions[id].return_on` — an absolute date computed client-side (`skipDuration()` in the template) when the user picks a skip duration (1d / 3d / next week / 2 weeks / 1 month / pick a date). 1d/3d land on a weekday. `reflection` — the editable "Today's Reflection" card's three fields, autosaved on input like annotations. Outreach rows gained an `annotations["outreach-<id>"]` textarea and a Skip mini-form (previously Skip committed immediately with no detail).

**Sandbox invariant (D1):** the template contains **no** `prompt()`/`confirm()`/`alert()` — Cowork's artifact iframe silently blocks native dialogs, which made Delegate/Skip/Not-important dead buttons before 0.6.0. All detail-capture and confirmation is inline (`.mini-form` + two-tap confirm).

Reader pattern (used by `/process-brief` and `/listen`):

```javascript
const state = JSON.parse(mirrorFile || widget_context["brief-<date>"] || "{}");
const tasks       = state.tasks || {};               // {task_id: {action, detail, return_on, ts, name}}
const annotations = state.annotations || {};          // {item_id: free-text} — task/inbox/outreach/event ids all share this map
const outreach    = state.outreach_actions || {};     // {contact_id: {name, action, bucket, signal, value_add, detail, return_on, ts}}
const reflection  = state.reflection || {};            // {biggest, blocked, one_thing, ts}
```

**Migration from 0.4.x–0.6.0:** earlier briefs stored `tasks_checked: {id: bool}` with no `tasks`; treat each `true` as `{action: "done"}`. 0.4.x–0.6.0 blobs are missing `reflection` and `return_on` — treat both as absent, not an error. The template's `save()` always writes `schema_version: "0.7.0"` going forward; every version bump so far has been additive-only, so a 0.7.0 reader can read any older blob and an older reader simply ignores fields it doesn't know about.

**State is read by `/listen` Step 1.5** (mine yesterday's brief), `/process-brief` Step 1, and `/end-day` Step 2c/4.0 when the user still runs it. Read order: the state-mirror file `<config-root>/briefs/<date>.state.json` **first** (written directly by desktop Cowork, or by the hosted-runtime preflight reading back the discovered shared-state capability — see Step 3.0), then `mcp__cowork__read_widget_context(artifact_id="todays-brief")` as a legacy fallback, then the **paste path** (ask the user to click 🔄 Sync brief state and paste the blob, write it to the state file), then — for `/end-day` only — its multi-select fallback gate. `/listen` is unattended and never prompts; if no source yields a blob it logs one line and runs its inference pass only (see `/listen` Step 1.5f).

### Decide create vs. update (race-aware)

1. `mcp__cowork__list_artifacts` → look for id `todays-brief`.
2. If found, check `metadata.target_date`:
   - `== intended-date` (`today_local` for `/brief`, `tomorrow_local` for `/end-day` Step 5) → `mcp__cowork__update_artifact(artifact_id="todays-brief", content=<HTML>, mcp_tools=[<FS_WRITE_TOOL from Step 3.0, when resolved>], metadata={target_date, plugin:"briefing", schema_version:"0.7.0"})`. Preserve matching annotation/task state by item id (Step 3a).
   - `!= intended-date` → DO NOT silently overwrite. Surface: "⚠ `todays-brief` exists with target_date `<existing>`; about to write `<new>`. This is a `/brief`↔`/end-day` race. Proceed (last-write-wins) or abort?" On proceed → update; on abort → exit Step 3.
   - older than `today_local` → update with fresh content (the old day's final state is in its markdown twin).
3. If not found → `mcp__cowork__create_artifact(id="todays-brief", artifact_type="html", content=<HTML>, mcp_tools=[<FS_WRITE_TOOL from Step 3.0, when resolved>], metadata={target_date, plugin:"briefing", schema_version:"0.7.0"})`.

Always pass `mcp_tools` when Step 3.0 resolved a tool — the allowlist lives in the artifact manifest, and an `update_artifact` that omits it keeps the prior grant (safe), but a `create_artifact` without it leaves the artifact unable to auto-sync until the next update.

### Step 3a — Preserve state across same-day re-runs

When the artifact already exists for the same date:
1. `mcp__cowork__read_widget_context(artifact_id="todays-brief")` to read the current blob.
2. The template restores `state.tasks`, `state.annotations`, and `state.outreach_actions` automatically on load (keyed by item id) — so re-rendering with the same item ids preserves everything.
3. Items dropped from today's data (e.g., a cancelled meeting) leave their prior annotation only as a line in the markdown twin under "Dropped from today's brief at <regen time>."

In Claude Code (no Cowork artifact tools): Step 3 degrades to writing the markdown twin, printing "Cowork artifact tools not available in this runtime — brief saved as markdown at `<path>`", and stopping.

---

## Step 4 — Notify the user

Output exactly one chat message:

> "Brief ready for <today_local>. Open the 'Today's Brief' artifact. Act on tasks/outreach inline (those actions get mined overnight by `/listen`), fill in Today's reflection at the bottom of the brief, then run `/process-brief` for anything you want drafted now. Markdown twin at `<config-root>/briefs/<today_local>.md`. <N> items filtered by surfacing-prefs."

If Step 3.0 found no write tool, append one sentence: "Note: brief auto-sync is unavailable (no filesystem MCP server connected). Click 🔄 Sync brief state in the artifact before you close out today, or run `/setup-brief` to enable auto-sync."

Don't dump the brief content into chat.

---

## Behavior rules

- **Read-only across all data sources.** No drafts, no task updates, no sends.
- **Always filter against `surfacing-prefs.md`** before rendering tasks/outreach (Step 0D). Suppressed items never appear; noise-class items never reach P0/P1.
- **Fixed section order**, identical formatting whether produced by `/brief` or `/end-day` pre-stage.
- Same-day re-run = refresh + preserve action/annotation state by item id. Different day = new content; yesterday's file preserved.
- If a source MCP isn't connected, that section renders "[Section]: source not connected — connect the MCP and re-run." No silent failure.
- Cost: ~3-5K tokens of synthesis on top of connector payloads. Per-section caps: 12 meetings, 12 tasks, 10 outreach.
- Telemetry: optionally log one line to ops `/log-agent-run` (skill: brief, item_counts, filtered_count, runtime_ms). Skip if ops isn't installed.
