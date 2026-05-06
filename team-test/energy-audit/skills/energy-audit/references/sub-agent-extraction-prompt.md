# Sub-agent extraction prompt template

This template is loaded by `energy-audit/SKILL.md` Step 3c (meetings) and Step 3d (Slack), populated at dispatch time, and sent to each sub-agent. The orchestrator passes:

- The transcript references / message history for the batch (recording IDs + access metadata for meetings; filtered messages inline for Slack)
- For meetings only: the discovered transcript-fetch action slug (`[TRANSCRIPT_FETCH_ACTION_SLUG]`) — sub-agent calls this slug to fetch its own transcripts when summary metadata is insufficient
- The cluster context (meetings only — tells the sub-agent it's working on a coherent group, not a random batch)
- The full energy profile content
- The 5 readiness criteria
- The audit window dates
- The mode flag (`full` / `short_window`)

The sub-agent returns an array of candidate briefs in the structured shape defined below. Editing this file is how clients change extraction behavior — keep `energy-audit/SKILL.md` body stable.

---

You are extracting automation candidates from [meetings | Slack messages] for an energy audit.

## Context

- Energy profile of the executive: [PROFILE_CONTENT]
- Audit window: [START_DATE] to [END_DATE]
- Mode: [full | short_window]
- Cluster context (meetings only): [CLUSTER_CONTEXT — e.g., "cluster: standup, total cluster size: 6 meetings, this batch: 3 of 6 (batch 1 of 2)"]
- Transcript-fetch action slug (meetings only): [TRANSCRIPT_FETCH_ACTION_SLUG] — call this slug with a recording ID from the input block when you need full transcript content (see "Working from summaries first" below).

The cluster context tells you whether you're seeing all of a recurring pattern, part of one, or a one-off. Use it to set your confidence: a single-meeting `ad_hoc` batch supports `weak`/`moderate` evidence at best; seeing 3 of 6 meetings in a `standup` cluster supports `strong` evidence for any pattern that recurs across the batch.

## Your input

[INPUT_BLOCK — populated at dispatch time with transcripts or filtered messages]

## What you're looking for

A candidate is anything the executive does — once or repeatedly — that produces a definable output, AND meets ALL FIVE of these criteria:

1. **Repetitive OR triggered by a clear event.**
   In `short_window` mode: a single observation counts ONLY if the pattern is CLEARLY recurring — recurring calendar event marker, weekly thread title, or explicit "every Monday" / "biweekly" / "weekly sync" language in the source. Otherwise drop.

2. **AI can execute end-to-end OR handle a substantial chunk.**

3. **Clear inputs and clear outputs.**

4. **Profile alignment.** Maps to one of the executive's stated drains, recurring patterns, or quarterly priorities. Cite the exact text from the profile.

5. **Buildable with current AI tooling.**

## Output shape

Return zero or more candidate briefs as a JSON array. Each must have ALL of these fields filled:

- `task_name` (string): short descriptive name
- `frequency_signal` (string): how often this happens (e.g., "weekly", "twice a month", "every board meeting")
- `time_estimate` (string): rough time per occurrence
- `profile_alignment` (string): which drain / pattern / priority this maps to (cite exact text)
- `manual_workflow` (string): step-by-step description of what's done today
- `pattern_notes` (string): any cross-occurrence patterns observed (e.g., "always Monday morning", "always after fundraising calls")
- `source` (string): "meetings" or "slack"
- `source_refs` (array of strings): transcript IDs (meetings) or message permalinks (slack)
- `evidence_strength` (string): "strong" / "moderate" / "weak"
  - **strong** — pattern observable in multiple places or explicit recurrence markers
  - **moderate** — pattern appears recurring but only in one source
  - **weak** — single-mention pattern with limited evidence

## Quality bar

Return ZERO candidates if nothing meets the bar. Don't pad. The audit's hard cap of 2 plans means quality matters more than coverage. A weak candidate dilutes the report.

## Working from summaries first (meetings only)

For meeting transcripts, work from the meeting summary + agenda + action items first (typically present in the list-action's response and inlined in `[INPUT_BLOCK]`). Only dive into the full transcript if the summary doesn't reveal a clear pattern — and when you do, call `[TRANSCRIPT_FETCH_ACTION_SLUG]` with the recording ID rather than guessing an action name. Most meeting platforms auto-summarize, so most batches won't need full-transcript fetches. This keeps context window usage manageable when the orchestrator dispatches multiple sub-agents in parallel.

**Null-summary fallback.** If the meeting summary is null/missing for any recording in the batch (a common case — summaries are generated asynchronously and freshly-recorded meetings often have no summary yet), do NOT auto-fall-through to the full transcript. First inspect title + invitees + duration + recurrence-marker — for many meetings this metadata alone establishes the candidate's recurrence pattern + invitees + cadence. Fetch the full transcript ONLY if metadata is genuinely ambiguous AND the meeting is not a recurring instance (recurring meetings are pattern-self-evident from metadata alone). Full-transcript fetches against fan-out batches are expensive — Fathom's own pitfall doc warns transcripts can run hundreds to thousands of turns.

**Handling large transcript responses (auto-offload).** Transcript-fetch responses are often large enough that the aggregator auto-offloads them to a remote sandbox file. For Composio MCP this happens routinely — even a single hour-long meeting transcript can trigger it (observed at tens of thousands of tokens, well below the older "50k+" rule of thumb). When this happens, the immediate response contains a small inline preview plus a sandbox file path (e.g., `/mnt/files/...`). To work with the full transcript, read the offloaded file via the aggregator's remote-bash or remote-workbench capability — do NOT assume the inline preview is the full transcript. If you only need a few specific signals (recurring agenda items, action-item shape, time-discipline cues), the inline preview may be enough; otherwise read the offloaded file before extracting candidates.

## What is NOT a candidate

- One-off requests with no recurrence pattern
- Activities that fail any of the five criteria
- Tasks the executive only watches (someone else owns them)
- Things that are clearly already automated
- Personal/social mentions with no operational pattern
