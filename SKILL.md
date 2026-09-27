---
name: uptime-weekly-report
description: "Generate a weekly Uptime Kuma product report (Work vs Personal, exec infographic, Red Flags summary, 24-week trend appendix) as a published artifact. Use when the user asks to run/refresh their uptime report."
---

# Weekly Uptime Report

Produces a weekly product-uptime report from the user's self-hosted Uptime
Kuma instance, split into a **primary** section (default label "Work") and a
**Personal** section (personal projects must never appear in the primary
section), plus a rolling **24-week trend appendix**. Published as a Claude
Artifact (HTML page) so it can run **unattended** with no local file writes
and no OS-level folder-permission dialogs.

Run this whenever the user asks to run, refresh, or generate their uptime
report, or on a schedule they've set up.

## First-time setup (do this before anything else)

This skill's folder (the same directory as this SKILL.md) should contain a
`config.json`. If it does not exist, or is missing required fields, run
setup before doing anything else:

1. Ask the user, conversationally, for:
   - The base URL of their Uptime Kuma instance (e.g. `https://uptime.example.com`).
   - What to call the non-personal category in report headings (default:
     `Work` — this could also be a company or team name).
   - The tag name used in Uptime Kuma to mark personal monitors/groups
     (default: `Personal`, case-sensitive).
2. Write those into `config.json` in this skill's folder:
   ```json
   {
     "uptime_kuma_url": "https://uptime.example.com",
     "primary_category_name": "Work",
     "personal_tag_name": "Personal",
     "artifact_url": ""
   }
   ```
3. Leave `artifact_url` empty. It gets filled in automatically the first
   time the report is published (see "Building and publishing" below) —
   never ask the user for it, and never invent one.

**Never put the API key in `config.json` or anywhere else in this skill
folder.** It is a secret and is handled per the "Data source" section below:
ask the user for it in the session (or use it from session context if
already provided) and never hardcode it into the skill or into generated
code.

If `config.json` already has all four fields, skip setup and go straight to
generating the report. If the user wants to change a setting later (a new
instance URL, a renamed category, republishing to a fresh artifact), update
`config.json` accordingly rather than asking every time.

## Why an Artifact, not a local file

Users typically don't want this skill to ever trigger an OS-level "grant
folder access" dialog, and want it to run unattended (e.g. from a scheduled
task). Writing to a connected folder on the user's computer requires a
one-time human approval dialog on that device — that breaks unattended
runs. So this skill **never** uses `device_bash` / `device_commit_files` /
local file storage for the report or its history. Instead:

- The report itself is a **published Artifact** (`Artifact` tool), which
  needs no device permissions at all.
- The 24-week trend history is stored in **that artifact's own database**
  (`Artifact` tool, `read_db` / `write_db` actions, collection `weeks`,
  one document per ISO date `YYYY-MM-DD`). This is server-side, durable
  across sessions, and requires no local storage or device link whatsoever.

Always republish to the artifact URL stored in `config.json` (pass it as
`url` to the `Artifact` tool) so the link stays stable week over week —
never create a new artifact for a routine run once one exists.

## Data source

Uptime Kuma at the `uptime_kuma_url` from `config.json`. Two ways to pull
data, in order of preference:

1. **Browser pane (preferred — gives full detail).** If the user is normally
   already logged in to Uptime Kuma in the built-in browser pane (device
   bridge, `mcp__remote-devices__Claude_Browser__*` tools), and site access
   has been granted with `scope: "site"` (persists indefinitely, no
   re-prompt), open the dashboard (`Claude_Browser__preview_start` with the
   root URL), then use `Claude_Browser__javascript_tool` to read the live
   Vue app state directly — this is far richer than the metrics endpoint:
   ```js
   const inst = document.querySelector('#app').__vue_app__._container._vnode.component.proxy;
   const r = inst.$root;
   // r.monitorList        -> {id, name, type, parent, active, tags}
   // r.uptimeList["<id>_24"] / ["<id>_720"]  -> uptime fractions (24h / 30d)
   // r.avgPingList[id]    -> average response time ms
   // r.importantHeartbeatList[id] -> [{status, time, msg}] state-change events
   ```
   Build the monitor table (id, group name via `parent`, name, type, active,
   up24, up720, avgping, tags) and the incident list (filter
   `importantHeartbeatList` to the last 7 days, pair consecutive down→up
   events per monitor to get downtime duration in minutes).
   For certificate expiry, fetch `/metrics` from within the same page
   context (same-origin `fetch`, avoids CORS) using HTTP Basic auth with the
   API key as the username-less password:
   `Authorization: Basic ' + btoa(':<API_KEY>')`, and parse the
   `monitor_cert_days_remaining{monitor_name="..."} N` lines.
   The API key is not stored in this skill — ask the user for it if it's
   not already in this session's context, and never hardcode it in the
   skill or in generated code.
   Close the tab when done (`Claude_Browser__tabs_close`).

2. **Metrics endpoint only (fallback, if the browser pane / device link is
   unavailable).** `GET <uptime_kuma_url>/metrics` with HTTP Basic auth
   (empty username, API key as password) returns Prometheus-format text:
   `monitor_status`, `monitor_response_time`, `monitor_cert_days_remaining`,
   `monitor_cert_is_valid`, one series per monitor labeled by
   `monitor_name`. This is a point-in-time snapshot only — no 24h/30d
   uptime %, no incident history. Note this limitation to the user when
   using this path. Try this directly with `Bash`/`curl` first; if the
   cloud sandbox's egress blocks the instance host, it will fail with a
   proxy/connect error — that's expected and not worth retrying, fall back
   to the browser pane.

## Category split

Tag-driven. In Uptime Kuma, monitors and groups tagged with the
`personal_tag_name` from `config.json` (default `Personal`, case-sensitive)
belong in the Personal section; a tag on a **group** does not automatically
propagate to children in the data model, so treat a monitor as Personal if
EITHER it carries the tag itself OR its parent group does (recurse if
groups can nest). Everything else — including all ungrouped monitors — goes
in the primary section (labeled with `primary_category_name`). If it's
unclear whether an ungrouped, untagged monitor is personal, ask the user
rather than guessing — never let a personal project leak into the primary
section.

## Report structure

One HTML page, sections in this order:

1. **Exec infographic** (top of page, primary category only — Personal
   never appears here). A screenshot-friendly card meant to be pasted into
   a slide or an email: title "`{primary_category_name}` Portfolio — Weekly
   Status" with the week date, an SVG ring gauge for the primary category's
   avg 24h uptime (color by status: >=99.5% good, 98–99.5% warn, <98% bad —
   fixed status palette `#0ca30c` / `#fab219` / `#d03b3b`, never re-themed
   for dark mode) with the 30d avg shown as a small secondary figure, a row
   of stat tiles (monitor count, incidents this week, downtime minutes this
   week, certs <45 days — the last colored red when nonzero), and a pill
   row listing only the product groups currently in warning/critical status
   (each pill: colored dot + group name + the reason — cert days, incident
   count, or 24h % — icon/color never carries meaning alone, always paired
   with text). Groups fully green are omitted from the pill row (or a
   single "all groups within target" pill shown if none are flagged). No
   interactivity needed — it's a static card designed to be screenshotted.
2. **Red Flags** — directly below the infographic, before the detailed
   sections. Heading literally "Red Flags" (styled prominently, left border
   accent in the bad/critical color). One paragraph, plain prose, generated
   from the data: call out only monitors/groups currently in warn or bad
   status (uptime dip with incident count and downtime minutes, certs <=45
   days with urgency — "renew now" at <=30 days, "schedule renewal"
   otherwise, grouping monitors that share one certificate). Do not mention
   anything healthy — no "all other systems nominal" filler beyond the one
   line used when there is nothing to flag ("None this week. All
   `{primary_category_name}` monitors are within target uptime and no
   certificates expire in the next 45 days."). Keep it to one paragraph,
   not a list.
3. **Primary category section** (heading = `primary_category_name`) — stat
   row (monitor count, avg 24h uptime, avg 30d uptime, incidents this week,
   downtime minutes this week, certs expiring within 45 days), incident
   list (monitor, start time, duration, message), cert-expiry watchlist
   (<=45 days, soonest first), then a collapsible (`<details>`) breakdown
   per product group with a per-monitor detail table (24h uptime, 30d
   uptime, avg response time, cert days).
4. **Personal** — identical structure to the primary section, only tagged
   monitors/groups.
5. **Appendix — 24 week trend** — one row per week: date, primary-category
   24h/30d uptime, primary-category incidents, primary-category downtime,
   Personal 24h uptime, Personal incidents. This table is built from the
   artifact's `weeks` database collection (see below), not recomputed from
   scratch each time.

Color thresholds for uptime %: >=99.5% good (green), 98-99.5% warn (amber),
<98% bad (red). Cert days: >45 good, 31-45 warn, <=30 bad. These are the
fixed status-palette colors (`#0ca30c` / `#fab219` / `#d03b3b`), defined
once as CSS custom properties and used unchanged in light and dark mode
(status colors are never re-themed, per the dataviz skill — only
surface/text tokens swap for dark). Style the rest of the page to support
both light and dark themes (CSS custom properties, no hardcoded colors
outside `:root` blocks) per standard Artifact design guidance. Before
shipping any new chart element beyond the ring gauge and stat tiles,
consult the `dataviz` skill.

## Updating the 24-week trend appendix

On each run:

1. Compute this week's summary row for both the primary category and
   Personal:
   `{date, primary_up24, primary_up720, primary_incidents, primary_downtime,
   primary_certs45, personal_up24, personal_up720, personal_incidents,
   personal_downtime}` where `date` is the ISO date the report runs
   (`YYYY-MM-DD`), uptime figures are the simple average of active
   monitors' 24h/30d uptime %, incidents/downtime are summed over the last
   7 days.
2. `Artifact` tool, `action: "write_db"`, `db_op: "set"`,
   `collection: "weeks"`, `doc_id: "<that date>"`, `data: <the row>` — on
   the existing artifact URL. Using the date as the doc id makes reruns on
   the same day idempotent (overwrites rather than duplicates).
3. `Artifact` tool, `action: "read_db"`, `db_op: "query"`,
   `collection: "weeks"`, `query: {"order_by": {"field": "date",
   "direction": "desc"}, "limit": 24}` to fetch the last 24 weeks (newest
   first; reverse for chronological display if preferred). This is the
   full appendix data — no separate pruning step is needed since only the
   last 24 are ever queried, but old rows beyond 24 can be left in place
   (harmless, just unused) or deleted with `db_op: "delete"` if the user
   wants the store kept trim.

## Building and publishing

Generate the full HTML (report body + infographic + Red Flags + trend
appendix, inline CSS, no external assets) to a file in the working
directory, then `Artifact` tool with that `file_path`.

- If `config.json`'s `artifact_url` is empty, this is the first run: call
  `Artifact` with no `url` so it creates a new artifact, then immediately
  write the returned URL into `config.json`'s `artifact_url` field so every
  future run republishes to the same page. Tell the user the link once it
  exists.
- Otherwise, pass `url: <artifact_url from config.json>` so it republishes
  in place.

Keep `capabilities: {"db": {}}` on every publish (omit the field on a plain
redeploy to carry it forward, but don't accidentally clear it by passing
`{}`). Before telling the user it's done, render the HTML with Playwright
(chromium at `/opt/pw-browsers/chromium` is preinstalled) and screenshot
the infographic card in both light and dark color schemes to sanity-check
layout — catch label collisions or overflow before they see it. Tell the
user in one or two sentences what changed this week (headline incidents,
any new cert warnings) rather than re-describing the whole report — they
can open the artifact.

## Running unattended

For a scheduled weekly run: use `create_trigger` (never local cron) with a
prompt that tells the fresh session to invoke this skill. If the user wants
one set up, ask them what day/time and timezone they want it to run, then
create it — don't assume a schedule. The browser-pane site access is
usually granted with `scope: "site"`, so it should not re-prompt on future
runs from the same linked device — but the run still requires the desktop
app to be open and that computer linked, since the built-in browser only
runs there. If the device is not reachable when the scheduled task fires,
fall back to the `/metrics`-only path (still fully unattended, no dialogs)
and note in the report that it's a partial data pull. Never fall back to
writing local files or requesting folder/device access — both can trigger
an OS-level permission prompt, which defeats unattended running. If a
scheduled task for this report already exists, update it with
`update_trigger` rather than creating a duplicate.
