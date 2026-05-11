---
name: meeting-performance-audit
description: Weekly meeting performance brief. Pulls meeting transcripts from the connected meeting tool over the past 7 days (since last successful run, capped at 7 days), reviews each meeting against Atlas's quality dimensions using one sub-agent per meeting, applies the Meeting Worth Matrix to recurring meetings (KILL / DELEGATE / TRANSFORM / PROTECT), synthesizes a constructive leadership reflection on the executive's behavior in the meetings, writes a two-tier brief to the workspace, and sends one notification email. Run weekly — manually or via the host's scheduler.
when_to_use: Run weekly to produce the meeting performance brief. Trigger phrases — "run meeting performance audit", "weekly meeting brief", "analyze last week's meetings", "meeting performance brief", "audit my meetings". Auto-trigger as a scheduled task in the host's scheduler. Also runs when invoked directly after `setup` and `extract-meeting-preferences` are complete.
atlas_methodology: opinionated
---

# meeting-performance-audit

The weekly run. Reads the local config and preferences file, computes the audit window, pulls transcripts from the connected meeting tool, fans out per-meeting reviews, assembles the brief, sends the notification email.

## Inputs

- **Local config** at `<workspace>/client-profile/meeting-performance-analyzer.local.md` — written by `setup`. Required.
- **Preferences file** at `<workspace>/client-profile/meeting-performance-analyzer-preferences.md` — written by `extract-meeting-preferences`. Required.
- **Last-run marker** at `<workspace>/.meeting-performance-analyzer-last-run.json` — read by this skill at start, written at end of a successful run. Optional on first run.
- **Host's loaded tool list** — required. The skill calls the configured meeting-tool and email tools.

## Required capabilities

(Abstract — concrete tool names live in plugin README §4.)

- Meeting tool: list meetings in window, fetch transcript and metadata
- Email send: one notification email at end of run
- File read / write / list against the workspace
- Sub-agent fan-out (one per meeting)

## Output

- Weekly brief at `<output_folder>/<YYYY-MM-DD>-meeting-brief.md`
- Updated last-run marker at `<workspace>/.meeting-performance-analyzer-last-run.json`
- One notification email to the configured recipient

## Steps

> **Note on "workspace".** All paths in this SKILL — the local config, preferences file, last-run marker, and output folder — resolve against the user's CWD (workspace), NOT the plugin install directory. Plugin-internal references (`references/*.md`) live in this skill's own `references/` subdirectory.

### 0. Read state

**Resolve all input paths against the workspace, NOT the plugin install directory.**

1. Read `<workspace>/client-profile/meeting-performance-analyzer.local.md`. If absent → halt with: *"No setup config found. Run `setup` first."*
2. Read `<workspace>/client-profile/meeting-performance-analyzer-preferences.md`. If absent → halt with: *"No meeting preferences found. Run `extract-meeting-preferences` first."*
3. Read `<workspace>/.meeting-performance-analyzer-last-run.json` if present. Schema:

```json
{
  "last_run_at": "<ISO 8601 UTC timestamp>",
  "success": true
}
```

If the file is absent or its `success` field is `false`, treat as "no prior successful run."

4. Verify capabilities still connected. Detection logic same as `setup` Step 0 (Pattern A annotation read or fallback `MANAGE_CONNECTIONS`). If the meeting tool or email tool is no longer connected → halt with the specific missing capability and: *"Re-run `setup` to refresh the connection."*

---

### 1. Compute window

Now (UTC) = `T_now`. Last run timestamp = `T_last` (or null if first run).

| Condition | Window start | Window end | Gap flag |
|---|---|---|---|
| First run (no `T_last`) | `T_now - 7 days` | `T_now` | false |
| `T_last` within last 7 days | `T_last` | `T_now` | false |
| `T_last` more than 7 days ago | `T_now - 7 days` | `T_now` | **true** — note the gap in the executive summary |

Save the window and gap-flag to running state.

---

### 2. List and classify meetings in window

> **Discover the actions; don't hardcode them.** The connected meeting toolkit (Fathom, Granola, Fireflies, or another) exposes its own action slugs — `FATHOM_LIST_MEETINGS`, `GRANOLA_LIST_*`, etc. Don't branch on `if meeting_tool == fathom: ... elif granola: ...`. Instead, use the toolkit-discovery capability to find the right actions by use-case query.
>
> For an aggregator like Composio: call the discovery tool once with one use-case query per action this audit needs — typically *"list meetings in date window"* and *"fetch meeting transcript by recording ID"* (and optionally *"fetch meeting summary by recording ID"* for the metadata-only fallback). The response returns the toolkit-scoped action slug for whatever the user has connected, plus the input schema. Capture the returned **list-action slug** for use here, and capture the **transcript-fetch action slug** for use at Step 3 (sub-agents will call it themselves). Same flow regardless of which meeting toolkit is wired.
>
> For direct vendor MCPs (Pattern B), the action names are typically discoverable from the host's loaded tool list — match by intent (`*list*meeting*`, `*get*transcript*`) rather than by hardcoded name.

This step is **metadata-only** — do **not** fetch transcripts here. Fetching transcripts in the main agent then passing them inline to sub-agents doesn't scale: meeting-tool transcripts are often large enough that the aggregator auto-offloads the response to a remote sandbox file (Composio behavior on responses ≳ 50k tokens). Slicing per-meeting transcripts out of an offloaded sandbox file in the main agent before fan-out is messy and brittle. Instead, each sub-agent fetches its own transcript at Step 3, scoped to a single meeting.

Call the meeting tool's list-meetings action (discovered per the callout above) with the window. Use the toolkit slug from `meeting_tool` in the local config to scope the discovery. If `meeting_tool_source: composio` and `meeting_tool_account_id` is set (multi-account case captured at setup Step 3.2), pass the account ID to disambiguate which connected account to pull from — same disambiguation pattern as Step 5's email send. If `meeting_tool_account_id` is null (single-account or direct vendor MCP), call without an account-id parameter.

Required fields per meeting in the list response:

- `id` — stable ID for the meeting in the meeting tool (used by sub-agents to fetch the transcript)
- `title`
- `start_time`, `end_time` (compute `duration_minutes` from these)
- `attendees` (list of names + emails if available)
- `recurrence_marker` — boolean or enum from the meeting tool's metadata. If the meeting tool doesn't expose this, set to `"unknown"` and the per-meeting sub-agent will infer from the title (recurring meetings often have stable titles).
- `transcript_status` — whether the transcript is processed and available, or pending

For each meeting:
1. **Filter against `restricted_types` from preferences.** If the title (case-insensitive substring) matches any entry in `restricted_types`, drop the meeting from the list. Keep a count of dropped meetings — used in the brief's exec summary if all meetings are dropped.
2. **Mark recurrence.** If the meeting tool returned `recurrence_marker = true`, mark recurring. If false, mark one-off. If unknown, the per-meeting sub-agent infers from the title at review time.
3. **Mark prefs hits.** If the title matches `non_negotiables`, attach `prefs_hit: "non_negotiable"`. If it matches `already_flagged_to_cut`, attach `prefs_hit: "already_flagged_to_cut"`. Otherwise null.

Aggregator note: the list-meetings preview row may include a `transcript: null` field even when the meeting IS processed and fetchable. That null is the preview shape, not a signal of transcript availability. Don't filter on it; sub-agents will discover real transcript availability when they call the dedicated fetch action at Step 3.

Save to running state:

```json
{
  "window_start": "...",
  "window_end": "...",
  "gap_flag": false,
  "transcript_fetch_action_slug": "...",  // discovered at top of step; passed to each sub-agent
  "meetings": [
    {
      "id": "...",
      "title": "...",
      "duration_minutes": 60,
      "attendees": [...],
      "is_recurring": true,
      "prefs_hit": null
    }
  ],
  "restricted_dropped_count": 0
}
```

---

### 3. Fan out per-meeting sub-agents

For each meeting in `running_state.meetings`, dispatch one sub-agent using the prompt template at `references/sub-agent-meeting-review-prompt.md`.

Substitute placeholders per meeting:
- `<exec_name>`, `<exec_email>` from the local config
- `<meeting_title>`, `<date_time>`, `<duration_minutes>`, `<attendee_list>`, `<recurrence marker>` from the meeting record
- `<id>` (the recording ID) from the meeting record — sub-agent uses this to fetch the transcript
- `<transcript_fetch_action_slug>` from `running_state.transcript_fetch_action_slug` — sub-agent calls this action with `<id>` to fetch
- `meeting_tool_account_id` from the local config (so the sub-agent can disambiguate when the toolkit has multiple accounts)
- `non_negotiables` and `already_flagged_to_cut` from the preferences file

Each sub-agent fetches its own transcript at the start of its work using the discovered action — see the prompt template at `references/sub-agent-meeting-review-prompt.md`. Aggregator-specific note: if the transcript-fetch response is large, the aggregator may auto-offload it to a remote sandbox file (Composio writes to `/mnt/files/...` and returns a path + preview). The sub-agent reads the offloaded file via the aggregator's remote-bash or remote-workbench tool. The prompt template covers this.

Dispatch all sub-agents in parallel where the host runtime supports it, but be aware of the meeting-tool's per-key rate limits (Fathom warns of HTTP 429 on bulk parallel fetches). For windows with > 5 meetings on a rate-limited toolkit, dispatch in batches of 3-5. Collect JSON records into `running_state.reviews`.

**Failure handling.** If a sub-agent fails to return a parseable JSON record, log the failure (meeting ID + error reason) into `running_state.failed_reviews` and continue. The brief will surface failed meetings in a "Could not analyze" footnote — one failure does not abort the run.

**Sandbox-loss resilience.** Per-sub-agent fetches keep this step robust against aggregator-MCP disconnect mid-run: each sub-agent does its own discovery + fetch, so there's no shared in-memory transcript state to lose. If an aggregator session dies between Step 2 and Step 3, the next run picks up cleanly because `last-run.json` only updates on full success (Step 6).

**Empty window.** If `running_state.meetings` is empty after filtering (no meetings or all dropped by `restricted_types`):
- If 0 total meetings in the window → skip the fan-out, jump to Step 4 with the appropriate exec-summary path
- If `restricted_dropped_count > 0` and post-filter list is empty → skip the fan-out, jump to Step 4 with the all-restricted exec-summary path

---

### 4. Assemble the brief

Read all sub-agent records (`running_state.reviews`). Compose the brief as one markdown file with this structure:

#### Executive Summary

Open with totals:

```
- Window: <window_start_date> – <window_end_date>
- Meetings reviewed: <N>
- Total time spent in meetings: <total_hours_minutes>
- Reclaimable per week (if recommendations adopted): <total_reclaimable_hours>
```

If `gap_flag` is true, append:

> "You haven't run this audit in <N> days — only the last 7 are analyzed here. Re-run sooner to keep coverage continuous."

If `restricted_dropped_count > 0` (some meetings dropped by `restricted_types`):

> "<N> meeting(s) in this window were marked restricted and excluded from analysis."

Top-line verdicts (1–2 sentences) — pick the most material thing the executive should act on. Examples:
- "Two recurring meetings (the Friday standup and the Wednesday roadmap review) earned KILL verdicts. Cutting both reclaims 2.5 hours/week."
- "No high-stakes recommendations this week — the calendar is mostly running clean. Worth holding the current shape."

If `running_state.meetings` is empty (no meetings in window, or all restricted-dropped):

> "No meetings analyzed in this window. <Either: 'Nothing on the calendar.' or 'All meetings in this window were marked restricted.'>"

— skip the detail sections entirely, jump to Step 5.

#### Detail sections

**Determine layout based on count.** Let `M = len(running_state.reviews)` (excluding failed reviews):

| `M` | Layout |
|---|---|
| 1–4 | Single section "Meetings this week" — list all meetings annotated with strengths and weaknesses |
| 5–7 | Top-5 only; skip Bottom-3 section |
| 8+ | Top-5 and Bottom-3 both |

**Top-5 / Bottom-3 ranking.** Sort all reviews by `total_score` (per-meeting score from the seven dimensions, max 14). Top-5 = highest 5 scores. Bottom-3 = lowest 3 scores.

For each top/bottom meeting, write:

```
### <Meeting title>

- **What worked / what didn't:** <one sentence summarizing the dimension scores; emphasize 1-2 concrete observations>
- **Date / duration:** <date>, <N> minutes
- <If recurring AND the per-meeting verdict is KILL/DELEGATE/TRANSFORM, mention here: "Recommended verdict: <verdict>. Reason: ...">
```

**Worth Matrix verdicts.** Group all *recurring* reviews by `worth_matrix.verdict`. For each verdict in order KILL → DELEGATE → TRANSFORM → PROTECT:

```
### <Verdict>

- **<Meeting title>** — <reason>. Reclaimable: <N> min/week.
- **<Meeting title>** — <reason>. Reclaimable: <N> min/week.
```

For TRANSFORM, also include the proposed change from `worth_matrix.transform_change`. For PROTECT, "Reclaimable: 0" can be omitted.

If `prefs_hit == "already_flagged_to_cut"`, the line reads: *"<Meeting title> — Already flagged for removal by the executive. No audit needed."*

If `prefs_hit == "non_negotiable"`, the meeting goes under PROTECT regardless of dimension scores. Line reads: *"<Meeting title> — Non-negotiable per executive's preferences."*

**Leadership Reflection.** Synthesize across all reviews' `leadership_observation` fields. Apply the tone rules from `references/atlas-leadership-reflection-tone.md`:

- Strengths: 3-5 bullets, each anchored in 1-2 specific meetings
- Growth areas: 2-4 bullets, each with one concrete behavior change for next week

Write the section as:

```
## Leadership Reflection

This week, you led <M> meetings. Here's what stood out about how you showed up.

### Strengths

- **<Strength name>.** <one-sentence anchored description, second-person voice, citing specific meeting(s)>

### Growth areas

- **<Pattern name>.** <one-sentence description with specific meeting citations and one concrete suggestion>
```

**Could not analyze (footnote).** If `running_state.failed_reviews` is non-empty:

```
---

*Footnote: <N> meeting(s) could not be analyzed: <list of titles + brief reasons>.*
```

#### Save the brief

Write the assembled markdown to:

```
<output_folder>/<window_end_date>-meeting-brief.md
```

(Use `<window_end_date>` as the filename anchor — typically today's date.)

If `dry_run: true` in the local config, print the brief to terminal instead of writing the file. Skip Step 5 (notification) and Step 6 (last-run update) entirely in dry-run mode.

---

### 5. Send notification email

Skip if `dry_run: true`.

Compose a short heads-up email:

- **Subject:** `Meeting performance brief — <window_end_date>`
- **Body:**
  ```
  Your weekly meeting brief is ready.

  - Meetings reviewed: <M>
  - Total time in meetings: <total_hours>
  - Reclaimable if recommendations adopted: <total_reclaimable_hours>

  Top recommendation: <top_line_verdict_one_sentence>

  Read the full brief: <output_folder>/<window_end_date>-meeting-brief.md
  ```

Send via the email tool configured in `email_provider`. If `email_source: composio`, pass the saved `email_account_id` to disambiguate among multiple connected accounts. Recipient: `email_recipient`.

If the send fails, log the failure but DO NOT halt the run — the brief is already written. Surface the failure in the terminal so the user knows to check the brief manually:

> "Brief written to <path>. Notification email failed to send: <error>. Open the brief manually."

---

### 6. Update last-run marker

Skip if `dry_run: true` OR if Step 4 wrote no brief (empty window AND no exec-summary path was triggered — though in practice the empty-window path still writes a brief, so this branch shouldn't fire).

Write `<workspace>/.meeting-performance-analyzer-last-run.json`:

```json
{
  "last_run_at": "<T_now ISO 8601 UTC>",
  "success": true
}
```

Print a confirmation:

> "Run complete. Brief saved to <output_folder>/<window_end_date>-meeting-brief.md. Notification sent to <email_recipient>. Next run will analyze meetings since <T_now>."

If any step in this run failed, do NOT update the last-run marker — the next manual run will pick up the same window again. Two failure classes do NOT count as run failures (the run is "degraded but successful," and the marker still updates):

- **Sub-agent review failures.** Surfaced in the brief's "Could not analyze" footnote.
- **Notification email send failure.** The brief is already written to disk; missing the heads-up email isn't worth re-running the whole audit. Surfaced in the terminal at Step 5.

A "real" run failure that blocks the marker update is something like: the meeting-tool list call failed, the brief couldn't be written to disk, or the config / preferences file couldn't be read.

## Output

A weekly brief markdown file plus an updated last-run marker. One notification email sent to the configured recipient.
