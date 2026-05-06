---
name: energy-audit
description: Audits the executive's trailing 14 days across all connected data sources (calendar / task tool / meeting transcripts / selected Slack channels — any combination, at least one). Calendar and tasks are read inline; meetings and Slack fan out to sub-agents that extract candidates per batch. Applies Atlas's automation-readiness criteria filtered through the executive's energy profile, ranks candidates, takes the top one or two, writes a structured report, hands off each top candidate to build-automation-plan, and sends one notification email when complete. Detects run-overlap with the previous successful run and offers to look only at the new days.
when_to_use: Run after `setup` and `extract-energy-profile` have completed. Trigger phrases — "let's run an energy audit", "run the energy audit", "what can we automate", "find automation opportunities". Also triggered by the recurring schedule registered at the end of `setup`. Do not run if the profile or the local config does not exist — point the user at `extract-energy-profile` or `setup` respectively.
atlas_methodology: opinionated
---

# energy-audit

Trailing 14-day audit across whichever sources the user connected. Identifies the top 1–2 automation candidates, hands each to `build-automation-plan`, saves one durable report file, and sends one notification email when done.

## Purpose

The audit produces actionable automation plans, not analysis. It reads the trailing 14 days from connected data sources, applies the readiness criteria from `references/atlas-energy-audit-methodology.md` filtered through the executive's energy profile, ranks the survivors, takes the top 1–2 (hard cap of 2), and produces ready-to-build plans for each via `build-automation-plan`.

If the previous successful run was less than 14 days ago and the invocation is manual, the audit detects the overlap and surfaces three options to the user (look only at the new days / run the full window anyway / skip). This prevents near-duplicate reports.

## Inputs

- **Energy profile** at `<workspace>/client-profile/exec-energy-profile.md` (written by `extract-energy-profile`).
- **Local config** at `<workspace>/.claude/energy-audit.local.md` (written by `setup`).
- **Run log** at `<workspace>/.claude/energy-audit.run-log.jsonl` — JSON Lines, one entry per audit run. Used for overlap detection.
- **Run date** = today, in the executive's local timezone.
- **Skill argument** (optional) — `--dry-run` or matching config flag suppresses the notification send and the file write side effects.

## Required capabilities

(Abstract list — no specific tool names. The host runtime maps these to its actual tools per the plugin README §4.)

- File read + write (profile, config, run log, output report)
- For each connected source declared in the local config:
  - Calendar read (list events for a date range, per account)
  - Task tool read (list tasks assigned to an identifier, for a date range)
  - Meetings read (list transcripts for a date range, OR list files in a Notion DB / Drive folder)
  - Slack read (list channel messages for a date range, resolve channel IDs)
- Sub-agent dispatch (for meetings + Slack fan-out)
- Notification email send (Gmail OR Outlook — channel chosen at `setup`)
- Skill invocation (calling `build-automation-plan`)

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The energy profile (`client-profile/exec-energy-profile.md`), the local config (`.claude/energy-audit.local.md`), the run log (`.claude/energy-audit.run-log.jsonl`), and the output report folder (per config) all resolve relative to this directory. Do NOT read or write these from the plugin's own install directory. Plugin-internal references (`references/atlas-energy-audit-methodology.md`, `references/sub-agent-extraction-prompt.md`, the build-automation-plan skill) use `../../` from this skill folder to mean plugin root.

### 0. Pre-flight

**Resolve all input paths against the workspace, NOT the plugin install directory. If a required file is missing in the workspace, do NOT fall back to the plugin folder — surface the specific error and the recovery skill.**

1. Read `<workspace>/client-profile/exec-energy-profile.md`. If absent, halt:
   > "No energy profile found at `client-profile/exec-energy-profile.md`. Run `extract-energy-profile` first."

2. Read `<workspace>/.claude/energy-audit.local.md`. If absent, halt:
   > "No setup config found at `.claude/energy-audit.local.md`. Run `setup` first."

3. For each source block present in the config (`calendar`, `tasks`, `meetings`, `slack`), verify the corresponding capability is still callable in the host's loaded tool list (re-detection per the `setup` skill's pattern). If any connected source has lost its tool, halt:
   > "The audit is set up to read from [source] but I can't find a connected [source] tool in your AI right now. Run `setup` to refresh the wiring."

4. Verify notification: if `notification_channel` is `gmail` or `outlook`, confirm that capability is still callable. If absent, halt with the same setup-pointer message scoped to the notification.

5. Detect dry-run: skill argument `--dry-run`, OR `dry_run: true` in the config. If either, set `dry_run = true`; the audit still runs but skips the file write side effects in Steps 9 + 13 and the notification send in Step 12 (printed to terminal instead).

### 1. Determine audit window — overlap-aware

Read the most recent successful run from `<workspace>/.claude/energy-audit.run-log.jsonl` (last line; if file absent or empty, treat as no prior run).

Branch:

- **Gap ≥ 14 days, OR no prior run** → window = trailing 14 days. Mode = `full`.
- **Gap < 14 days AND invocation source is the recurring schedule** → window = trailing 14 days. Mode = `full`. (The schedule itself is bi-weekly; running the full window is the right behavior — note in the report that there's overlap with the previous run.)
- **Gap < 14 days AND invocation is manual** → surface this prompt to the user (verbatim, second-person, plain language):

> "Heads up — your last energy audit ran [N] days ago ([date]). The full audit normally looks at 14 days of activity, so most of that data is already in the previous report. Three options:
>
> 1. **Look only at the new days** — review just the [N] days since the last audit. Less data to spot patterns, but it'll catch anything new.
> 2. **Run the full audit anyway** — look at all 14 days again. Most of it overlaps with the last report, but you'll get a fresh ranking.
> 3. **Skip this one** — wait until the next scheduled audit ([next scheduled date], or whenever you trigger it manually if no recurring schedule is set)."

User picks. If "look only at the new days," set mode = `short_window`, window = `[last_run + 1, today]`. If "full audit anyway," mode = `full`, window = trailing 14 days. If "skip," halt cleanly with no further work.

**Gap-too-small abort:** if mode = `short_window` AND `(today - last_run) < 3 days`, halt:
> "Last run was [N] days ago. Wait at least 3 days for new signal."

### 2. Quarter-staleness check

Compute the run quarter (Q1 = Jan-Mar, Q2 = Apr-Jun, Q3 = Jul-Sep, Q4 = Oct-Dec) from the run date. Read `Quarterly priorities last updated: YYYY-MM-DD` from the energy profile and compute its quarter.

If the run quarter differs from the profile's last-updated quarter, set:
- `quarterly_stale = true`
- `last_updated_quarter = <profile's last-updated quarter>`

These get surfaced in the report header banner (Step 9) and passed through to `build-automation-plan` invocations (Step 10).

If quarters match, `quarterly_stale = false`; no banner.

### 3. Pull data per connected source

Branch per source. Calendar + tasks read inline; meetings + Slack fan out to sub-agents. Skip any source whose block is absent from the config.

#### 3a. Calendar (inline) — only if `calendar` block in config

> **Discover the list-events action.** Don't hardcode `<TOOLKIT>_LIST_EVENTS` per provider. For Composio: call the discovery capability with the use-case query *"list calendar events in date window"* scoped to the account's toolkit slug (`googlecalendar` or `outlook` from the account identifier). For direct vendor MCPs: match by intent against the host's loaded tool list (`*list*event*`). Capture the discovered slug to running state once per (toolkit × account) and reuse for each calendar in the per-account selection.

For each account in `calendar.accounts`:

- Look up the account's calendar selection in `calendar.calendar_ids[<account-id>]`. If the field is absent (config from a pre-`calendar_ids` version), fall back to `["primary"]` and add a one-line note to the report header: *"Calendar IDs not captured at setup — defaulted to each account's primary. Re-run `setup` to pick specific calendars."*
- For each calendar ID in the selection:
  - If the ID is the literal string `primary`, call the calendar tool's "list events" function with the account identifier alone (the tool treats unspecified calendar as primary).
  - Otherwise pass the calendar ID alongside the account identifier.
- Tag each returned event with `source_calendar = <provider>:<account-identifier>:<calendar-id>`.

After all (account × calendar) combinations are pulled, **de-duplicate** events that appear in multiple calendars:
- Primary key: `iCalUID` (events have this when synced across accounts or calendars)
- Fallback: `(start, end, title)` tuple

**Filter** per methodology:
- Drop events the executive declined (response status = `declined`)
- Drop events with no other attendees (typically personal blocks — not work signal)
- Drop events shorter than 15 minutes

The remaining event list is the calendar signal for this audit.

#### 3b. Tasks (inline) — only if `tasks` block in config

Provider-agnostic — the audit reads the captured `container_ids` + assignee filter from config and queries each container via the toolkit's "query container with filter" capability. Provider-specific differences live in the captured `assignee_property_type` (which the audit uses to construct the right filter shape for the toolkit).

Required config fields: `tasks.container_ids` (list of container IDs, or `["all"]` for flat-list providers), `tasks.assignee_property` (the field name as the toolkit returns it), `tasks.assignee_property_type` (normalized to one of `people`, `select`, `multi_select`, `status`, `relation`, `rich_text`, `text`, `enum`, `unknown`), `tasks.assignee_value` (string for most types; array for `multi_select`).

**Backward-compat halt.** If the config has the OLD shape — `tasks.assignee_identifier` present AND any of the new fields absent — halt rather than guess:
> "Your task config uses the old single-identifier shape and doesn't have enough detail to query your task tool correctly (the assignee question changed in a recent update so the audit can handle tools where assignment isn't a simple email field). Run `setup` to refresh — it'll figure out where your tasks live and which field labels assignment."

Don't attempt to translate the old field — it's too lossy in providers where assignment isn't a `people`/email field.

**Filter construction.** Build a filter object based on `assignee_property_type`. The shape varies per toolkit; the audit constructs the toolkit-appropriate shape from the captured type. Common shapes (Composio Notion API used as a representative example — adapt the field names to whatever the connected toolkit's filter API expects):
- `people` → predicate: *the field contains the user*. The captured value may be an email or a user ID; if the toolkit needs an ID and the user pasted an email at setup, resolve via the toolkit's user-search call first.
- `select` / `status` / `enum` → predicate: *the field equals the captured value*.
- `multi_select` → predicate: *the field contains all of the captured values* (combine with logical AND if the array has more than one value).
- `relation` → predicate: *the field contains the captured page/row UUID*.
- `rich_text` / `text` → predicate: *the field equals the captured string*.
- `unknown` → halt with a setup-refresh pointer; don't guess at the filter shape.

**Per-container override.** If `tasks.per_container` exists in config, the listed container IDs use the per-container `assignee_property` / `assignee_property_type` / `assignee_value` instead of the top-level fields.

> **Discover the query/list action.** Don't guess `<TOOLKIT>_QUERY_*` from the toolkit slug. For Composio: call the discovery capability with the use-case query *"query container with filter and date range"* (or *"list tasks in flat list"* if `container_ids` is `["all"]`) scoped to `tasks.provider`. The response returns the toolkit-scoped slug + filter schema; build the filter object from the captured `assignee_property_type` per the type-shape table below using the schema as the source of truth. For direct vendor MCPs: match by intent (`*query*`, `*list*tasks*with*filter*`, `*search*`).

**Query.** For each container in `tasks.container_ids`:
- If the ID is the literal string `"all"`, call the toolkit's flat list-tasks function with the captured filter and the window dates (no container parameter).
- Otherwise call the toolkit's "query container with filter" function — Composio: `<TOOLKIT>_QUERY_*` or `<TOOLKIT>_LIST_*_WITH_FILTER` — with the container ID, the built filter, and a date filter bounded by the audit window.
- Tag each returned task with `source_container = <container-id>`.

Concatenate task results across all queried containers.

**Filter** per methodology (applies after the toolkit query returns results):
- Drop tasks with status `blocked` or `wontfix` that have no work logged in the window
- Drop tasks the executive watches but does not own (assignee match is the gate; watchers don't pass)

The remaining task list is the task signal for this audit.

#### 3c. Meetings (sub-agent fan-out) — only if `meetings` block in config

> **Discover the actions; don't hardcode them.** The connected meeting toolkit (Fathom, Granola, Fireflies, Notion, Google Drive, or another) exposes its own action slugs. Don't branch on `if meetings.provider == fathom: ... elif granola: ...` — the toolkit slug captured at setup is a scope hint, not a decision tree.
>
> For an aggregator like Composio: call the toolkit-discovery capability with use-case queries scoped to `meetings.provider`. The audit needs two discovered actions:
> - **list-action** — query: *"list meetings in date window"* (for direct providers) OR *"list files in folder"* (for `notion` / `googledrive` provider — the location string from `meetings.location` is the folder/DB scope). The response returns the toolkit-scoped slug + input schema. If discovery returns multiple matching slugs (Composio ranks by relevance), take the top-ranked match. If none look right for the intent, narrow the query (e.g., add "by date range" or include the toolkit name verbatim) and re-run.
> - **transcript-fetch action** — query: *"fetch meeting transcript by recording ID"* (for direct providers) OR *"read file content by ID"* (for `notion` / `googledrive`). Each meetings sub-agent will call this action itself at Step 3c's fan-out, so capture the slug here for pass-through.
>
> For direct vendor MCPs (Pattern B), match by intent against the host's loaded tool list (`*list*meeting*` / `*list*transcript*` for the list-action; `*get*transcript*` / `*fetch*transcript*` / `*read*file*` for the fetch-action) rather than by hardcoded name.
>
> Save both slugs to running state as `meetings_list_action_slug` and `meetings_transcript_fetch_action_slug`. The fetch slug gets passed to each meetings sub-agent at the fan-out (Step 3c continuing below).

Call the discovered list-action with the audit window dates. For `notion` / `googledrive` providers, scope the call by `meetings.location` (the folder or DB ID captured at setup). The response is the meetings list for the window — same downstream shape regardless of which toolkit is wired.

**Cluster, then batch, then dispatch.** Same-pattern meetings share extraction patterns — splitting them across random batches fragments evidence and forces sub-agents to detect a recurring pattern from a partial slice. Group by pattern first, then batch within groups with a small cap so no single sub-agent is overwhelmed.

**Tier 1 — exact/normalized title cluster.** Group meetings whose titles match after normalization:
- Lowercase
- Strip trailing/leading date markers and ordinal markers ("Standup - Wed Apr 24" → "standup"; "Product Huddle #14" → "product huddle")
- Strip leading articles ("the/a/an")
- Collapse internal whitespace

For each unique normalized title with ≥2 meetings in the window, form a Tier-1 cluster named after the normalized title (e.g., `cluster: standup (6 meetings)`).

**Tier 2 — category cluster** for Tier 1 leftovers (meetings with unique titles or single occurrences in the window). Classify each leftover into ONE of these categories using title + invitee + recurrence-flag heuristics:
- **`one_on_one`** — exactly 2 attendees (executive + one other) AND recurrence flag set, OR title contains `1:1` / `1on1` / `weekly with` / `bi-weekly with`
- **`team_meeting`** — 3–8 attendees AND recurrence flag set, AND the title doesn't fall into the more specific buckets below
- **`board_or_leadership`** — title contains `board` / `exec` / `executive` / `leadership` / `strategy` / `offsite`
- **`external`** — at least one attendee from outside the executive's primary email domain (or outside the workspace's known internal domains, when known)
- **`ad_hoc`** — anything that doesn't fit the above (default bucket)

Form one Tier-2 cluster per non-empty category, named after the category (e.g., `cluster: one_on_one (5 meetings)`).

**Batch within each cluster** with a cap of **3 meetings per sub-agent**. A 6-meeting `standup` cluster becomes two batches of 3; a 5-meeting `one_on_one` cluster becomes one batch of 3 + one batch of 2; a 2-meeting `board_or_leadership` cluster becomes one batch of 2; a single-meeting cluster becomes one batch of 1. The cap is hard — never exceed 3, even if a cluster is larger.

**Dispatch one sub-agent per batch in parallel.** With a typical 14-day window of 15–25 meetings, this produces roughly 8–12 sub-agents (vs. the prior random-batches-of-3 shape which produced the same count but with incoherent slices).

Each sub-agent receives the prompt loaded from `references/sub-agent-extraction-prompt.md` with these placeholders populated:
- `[INPUT_BLOCK]` — the transcript references for the batch — recording IDs (or file IDs for `notion` / `googledrive` providers), plus any access metadata returned by the discovered list-action (e.g., `meeting_tool_account_id` for Composio multi-account toolkits). The sub-agent fetches transcript content itself at run time using the slug from `[TRANSCRIPT_FETCH_ACTION_SLUG]`. Do NOT inline transcript bodies here — large transcripts trigger aggregator auto-offload mid-fan-out which is brittle.
- `[TRANSCRIPT_FETCH_ACTION_SLUG]` — the transcript-fetch action slug discovered at the top of Step 3c. Each sub-agent calls this slug to fetch its own transcript when needed (per the prompt template's "Working from summaries first" + "Null-summary fallback" subsections). Pass-through verbatim — do not branch on the slug value.
- `[CLUSTER_CONTEXT]` — the cluster name + cluster total size + this batch's position (e.g., `cluster: standup, total cluster size: 6 meetings, this batch: 3 of 6 (batch 1 of 2)`). Tells the sub-agent it's working on a coherent slice and how much of the cluster it's seeing.
- `[PROFILE_CONTENT]` — the full energy profile content (read from Step 0)
- `[START_DATE]` and `[END_DATE]` — the window dates from Step 1
- Mode flag (`full` or `short_window`) — affects how strictly criterion 1 (repetitive) is interpreted (see methodology's Short-window mode subsection)

Each sub-agent returns: an array of candidate briefs with the shape defined in `references/sub-agent-extraction-prompt.md` (`task_name`, `frequency_signal`, `time_estimate`, `profile_alignment`, `manual_workflow`, `pattern_notes`, `source: "meetings"`, `source_refs`, `evidence_strength`).

Collect candidates from all meetings sub-agents into a single list. The light-dedup pass at Step 5 will merge any candidates the same cluster surfaced across two batches (e.g., both standup batches identifying the same daily-standup-summary pattern).

If a sub-agent fails (timeout, error, malformed response), log the failure and continue with whatever sub-agents did return. Don't silently retry.

#### 3d. Slack (sub-agent fan-out) — only if `slack` block in config

> **Discover the channel-history action.** Don't hardcode `SLACK_CONVERSATIONS_HISTORY` or any other vendor-specific slug. For Composio: call the discovery capability with the use-case query *"fetch channel message history for date range"* scoped to `slack`. For direct vendor MCPs: match by intent (`*channel*history*`, `*conversations*history*`, `*list*messages*`). Capture the slug once and reuse across the fan-out.

For each channel in `slack.channels`, dispatch one sub-agent (cap of 1 channel per sub-agent — Slack histories can be large).

Pre-filter messages before sending to the sub-agent:
- Drop bot messages
- Keep messages where the executive is the author OR is mentioned (`@<exec>` or in a thread the exec participated in)
- Include parent-thread context for threads where any kept message lives

Each sub-agent receives the prompt loaded from `references/sub-agent-extraction-prompt.md` with placeholders populated:
- `[INPUT_BLOCK]` — the channel ID, name, and the filtered message history for the window
- `[PROFILE_CONTENT]` — full energy profile
- `[START_DATE]` and `[END_DATE]` — window dates
- Mode flag (`full` or `short_window`)

Returns: array of candidate briefs (same shape as meetings, with `source: "slack"` and `source_refs` = message permalinks).

Collect Slack candidates into the candidate list.

#### Failure handling for the whole step

If every connected source fails to return data (every inline call errors AND every sub-agent fails), halt:
> "None of your connected sources returned data for [date range]. Check that [list of connected sources] are still authorized."

Do not generate a report from nothing. The audit prefers an honest failure to a fabricated empty report.

### 4. Signals breakdown

For each connected source, produce one line summarizing volume + shape. Examples:

- *"Calendar: 47 events, ~23 hours, concentrated mid-week."*
- *"Tasks: 31 assigned, 19 completed, 12 open."*
- *"Meetings: 14 transcripts pulled, ~21 hours of recorded time."*
- *"Slack: 312 messages across 3 selected channels (#exec-team, #board-prep, #ops)."*

Plus a 2–4 line prose summary of the period's shape — what concentrated, anything notable, anything missing. In `short_window` mode, the prose summary acknowledges the narrower view explicitly (*"This is a 5-day window, so patterns will be lighter than a full audit."*).

### 5. Identify candidates (light dedup)

Concatenate all candidate briefs from all sources (calendar / tasks / meetings / Slack) into a single list.

**Light dedup pass:** two candidates merge if their `task_name` values match (case-insensitive + simple normalization — strip leading "the/a/an", collapse whitespace). Merged candidates carry combined source attribution (e.g., `source: "meetings + slack"`) and the higher of the two `evidence_strength` values.

No heavier similarity matching for v1 — see methodology for the rationale (hard cap of 2 plans makes minor dedup misses harmless).

### 6. Apply readiness criteria

For each candidate, check all five criteria from `references/atlas-energy-audit-methodology.md`:

1. Repetitive OR triggered by a clear event
2. AI can execute end-to-end OR handle a substantial chunk
3. Clear inputs and clear outputs
4. Profile alignment (drain or priority weighting)
5. Buildable with current AI tooling

A candidate must pass ALL FIVE to be eligible.

**Short-window mode adjustment** to criterion 1: in `short_window` mode, a single observation passes only if the pattern is *clearly* recurring — recurring calendar event marker, weekly Slack thread title, or explicit "every Monday" / "biweekly check-in" / "weekly sync" language in the source. Otherwise drop. (See methodology's Short-window mode subsection.)

### 7. Rank and select

Apply ranking weights from `references/atlas-energy-audit-methodology.md`:

1. Drain alignment (highest)
2. Frequency
3. Priority alignment (tiebreaker)
4. Cost-of-task (second tiebreaker)

Take top 1–2. **Hard cap of 2.**

If zero candidates pass Step 6, jump to Step 11 (zero-candidate handling).

### 8. Drain alignment

For each drain in the energy profile, list observed signals across all connected sources that match. Examples:

- *"Drain: 'investor updates' — 4 Slack threads (#exec-board), 2 calendar events (board prep), 1 meeting mention. No matches in tasks."*
- *"Drain: 'expense approvals' — no observed signals this period."*

If a drain has no observed match, note it explicitly. This section is the bridge between what the executive said in onboarding and what the data shows.

### 9. Write the report file

Output path: `<output_folder>/<YYYY-MM-DD>_energy-audit.md` where `<output_folder>` comes from the config (`output_folder` field) and the date is the run date.

The report has **six required sections in this exact order**:

1. **Header** — run date, audit window (start to end), mode (`full` or `short_window`), connected sources used, profile completeness (`complete` or `partial — [missing fields]`), quarterly-stale banner if `quarterly_stale = true`.
   - In short-window mode, include this line: *"Audit window: [start] – [end] ([N] days). This is a shorter window than the standard 14 days because the previous audit ran on [last_run]."*
   - Quarterly-stale banner: *"⚠ Quarterly priorities haven't been updated this quarter (last updated [last_updated_quarter]). The audit is filtering through stale priorities — refresh via `extract-energy-profile` for sharper targeting next time."*

2. **Signals breakdown** (the lines + prose summary from Step 4).

3. **Drain alignment** (the per-drain observation list from Step 8).

4. **Automation candidates table** — every candidate evaluated, with these columns: name, source(s), evidence strength, score (out of 5 — number of readiness criteria passed). Ends with an explicit selection line: *"Selected for Automation Plans (cap: 2): [name 1] and [name 2]."* (or just *"[name 1]"* if only one passes, or the zero-candidate text per Step 11).

5. **Top candidates** — one `### Candidate N: [Task name]` subsection per top candidate (capped at 2). Each subsection contains the full structured brief:
   - Source(s)
   - Evidence strength
   - Frequency signal
   - Estimated time per occurrence
   - Profile alignment (cited drain/pattern/priority text)
   - Manual workflow today
   - Pattern notes

6. **`## Automation Plans`** — section header with the literal text `(pending)` underneath. **The marker is critical** — `build-automation-plan` looks for this section and replaces the `(pending)` text as plans are appended.

After writing, **confirm the file was written successfully and contains all required sections** before continuing — six sections in normal mode, five in zero-candidate mode (Step 11 skips section 6 entirely). If any required section is missing, fix the report first; do not proceed to handoff.

In dry-run mode, print the report content to terminal but do NOT write the file.

### 10. Hand off to build-automation-plan

For each top candidate (1 or 2, never more):

1. Invoke `build-automation-plan` with these inputs:
   - The full candidate brief (all fields, including `source`, `source_refs`, `evidence_strength`)
   - The full energy profile content
   - The path to the report file (so build-automation-plan can append the plan)
   - `quarterly_stale: true` and `last_updated_quarter` if Step 2 set them

2. Wait for the invocation to complete.

3. **Read the report file back** and verify the `## Automation Plans` section now contains a subsection whose title matches the candidate's `task_name`. If the section still shows `(pending)` or is missing the candidate's plan, halt:
   > "Plan generation incomplete for `<candidate>`. The audit report at `<path>` is saved with the candidates section but without the plan."

   Do NOT send a misleading "audit complete" notification in this case.

4. After the read-back confirms success, move to the next candidate.

**Serial invocation only.** Do NOT parallelize the build-automation-plan calls — plans appended out of order can produce confusing output, and any failure mid-flow is easier to diagnose serially.

In dry-run mode, log "would invoke build-automation-plan for [candidate]" for each top candidate and skip the actual invocation.

### 11. Zero-candidate handling

If Step 7 found zero candidates passing the readiness criteria:

1. Write the report file with sections 1–3 (Header, Signals breakdown, Drain alignment) as normal.
2. For section 4 (Automation candidates table): include the table of all evaluated candidates with their scores, but the selection line reads: *"No candidate met the automation-readiness bar this period."*
3. Replace section 5 (Top candidates) with this note:
   > "No candidate met the automation-readiness bar this period. The audit prefers to surface nothing rather than pad with weak candidates. To increase signal, ensure your connected sources are being used consistently and the energy profile is up to date."
4. Skip section 6 entirely (no `## Automation Plans` section needed when zero plans).
5. Skip the build-automation-plan handoff (Step 10).
6. Continue to Step 12 with a "no plans this time" notification body.

A zero-plan audit is correct output when the data does not support a confident recommendation.

### 12. Send notification

Channel = `notification_channel` from config. If `notification_channel: none`, **skip this step entirely** — the report file is still written; the email is just the heads-up.

Recipient = `notification_target` from config.

Body (plain language, second-person, no jargon):

- **Full audit, N plans (1 or 2):** *"Energy audit complete ([window]). [N] automation plan[s] ready. View: [path]"*
- **Short-window audit:** *"Energy audit complete — shorter window because the last audit was recent ([window], [N] days). [N] automation plan[s] ready. View: [path]"*
- **Zero plans:** *"Energy audit complete. Nothing met the bar this time. View: [path]"*
- **Quarterly-stale append (when `quarterly_stale = true`):** add as a second line — *"Heads up: your quarterly priorities haven't been updated this quarter. Run `extract-energy-profile` to refresh for sharper targeting next time."*

The notification is a **real send** (not a draft). Use the available email tool's "send" function for the configured channel (Gmail or Outlook).

If `notification_account_id` is present in config (set by `setup` Step 5 when the chosen toolkit had multiple accounts connected), pass it as the `account` / `user_id` parameter on the send call so the message goes from the right account. If absent, let the tool default — single-account toolkits work without it.

In dry-run mode, print the body and the report path to terminal but do NOT send.

### 13. Persist run log

Append a one-line JSON entry to `<workspace>/.claude/energy-audit.run-log.jsonl`:

```json
{"timestamp":"<ISO 8601>","window_start":"<YYYY-MM-DD>","window_end":"<YYYY-MM-DD>","mode":"<full|short_window>","sources_used":["calendar","tasks"],"candidates_evaluated":42,"candidates_passed":3,"plans_generated":2,"channel":"gmail","send_success":true,"dry_run":false}
```

The log is for debugging; the most-recent timestamp drives Step 1's overlap detection on subsequent runs.

In dry-run mode, write a dry-run-tagged entry to terminal but do NOT append to the log file.

## Output

A markdown report at `<output_folder>/<YYYY-MM-DD>_energy-audit.md`, optionally followed by 1 or 2 appended automation plans (added by `build-automation-plan`). One notification email to the configured recipient if `notification_channel ≠ none`. One JSON Lines entry appended to the run log.

In dry-run mode: terminal output of the report content, the notification body that would have been sent, and the run-log entry that would have been written. No files written, no email sent.

## Customization

- **Audit window override** — pass `--window-start` and `--window-end` skill arguments to override the trailing-14-day default. Skips Step 1's overlap detection entirely. Mode is set to `full` regardless of gap.
- **Sub-agent prompt** — the meetings + Slack extraction prompt lives in `references/sub-agent-extraction-prompt.md`. Edit there to change extraction behavior; the skill body stays stable.
- **Methodology** — readiness criteria, ranking weights, signals-breakdown rules, short-window-mode rule, and report structure all live in `references/atlas-energy-audit-methodology.md`. Edit there to fork the methodology; the skill body stays stable.
- **Output path** — `output_folder` field in `.claude/energy-audit.local.md` (written by `setup`).
- **Profile path** — `profile_path` field in `.claude/energy-audit.local.md` (defaults to `client-profile/exec-energy-profile.md`).
- **Dry run** — `dry_run: true` in config, OR `--dry-run` skill argument.

## Why opinionated

Five readiness criteria, ranking weights (drain > frequency > priority > cost), the hard cap of 2 plans, the conservative-output rule (zero plans is fine), the requirement that candidates pass ALL FIVE criteria — these are Atlas's specific approach. The methodology is the substrate; the skill body is the orchestration.

The hard cap of 2 plans isn't arbitrary. Atlas's experience: more than 2 plans per audit dilutes execution focus. The cap forces the audit to actually rank rather than enumerate, which makes the output decision-grade rather than survey-grade.

The plain-language report structure (six sections, signals breakdown, drain alignment, candidates table) is also opinionated — the audit is meant to be readable on its own as a 14-day operational snapshot, not just a feeder for plans. Clients who want different report sections fork `references/atlas-energy-audit-methodology.md` and adjust the report-structure spec there.
