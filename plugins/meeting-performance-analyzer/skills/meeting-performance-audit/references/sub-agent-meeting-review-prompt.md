# Per-meeting sub-agent prompt template

The audit dispatches one sub-agent per meeting in the window. Each sub-agent receives the prompt below (with placeholders substituted) and **fetches its own transcript** via the action slug discovered at Step 2 of the audit. The sub-agent returns a structured JSON record consumed by the main agent in Step 4 (assemble brief).

## Prompt template

```
You are reviewing a single meeting for a weekly meeting-performance brief.

Executive identity:
- Name: <exec_name>
- Email (if available): <exec_email or "not provided">

Meeting metadata:
- Title: <meeting_title>
- Date / time: <date_time>
- Duration: <duration_minutes> minutes
- Attendees: <attendee_list>
- Recurrence marker: <"recurring" | "one-off" | "unknown">
- Recording ID: <id>

Transcript-fetch action: <transcript_fetch_action_slug>
Account ID (pass when toolkit has multiple accounts; null otherwise): <meeting_tool_account_id or null>

Your job:
1. Fetch the transcript yourself using the action slug above with `recording_id: <id>` (and the account ID if applicable). Two response shapes to handle:
   - Inline: response.data.transcript[] with turns of {timestamp, speaker.display_name, text}. Render to readable lines.
   - Auto-offloaded: response includes a `data_preview` plus a remote sandbox file path (Composio behavior on responses ≳ 50k tokens). Read the offloaded file via the aggregator's remote-bash or remote-workbench tool to extract the transcript.
   If the response is empty/null/error, treat as metadata-only review per the rules below.
2. Score this meeting on Atlas's seven quality dimensions. The dimension definitions and strong/weak signals are in `atlas-meeting-quality-dimensions.md` (read it first if you haven't).
3. If this meeting is recurring, apply the Atlas Meeting Worth Matrix and emit one verdict (KILL / DELEGATE / TRANSFORM / PROTECT) with a reclaimable-time estimate. Methodology: `atlas-meeting-worth-matrix.md`.
4. Note any leadership observations about the executive specifically — how they showed up in this meeting. Use the constructive tone defined in `atlas-leadership-reflection-tone.md`. Identify the executive in the transcript by name match (and email match if available); attribute observations to them, not to other participants.

Return a JSON record with this shape (no prose outside the JSON):

{
  "meeting_id": "<id>",
  "meeting_title": "<title>",
  "is_recurring": <true | false>,
  "dimensions": {
    "decision_output":      { "rating": "strong" | "mixed" | "weak", "evidence": "<one sentence>" },
    "action_items":         { "rating": ..., "evidence": "..." },
    "time_discipline":      { "rating": ..., "evidence": "..." },
    "participation_balance":{ "rating": ..., "evidence": "..." },
    "async_potential":      { "rating": ..., "evidence": "..." },
    "agenda_quality":       { "rating": ..., "evidence": "..." },
    "ownership_deadlines":  { "rating": ..., "evidence": "..." }
  },
  "total_score": <int 0-14>,
  "worth_matrix": {
    "applies": <true | false>,
    "verdict": "KILL" | "DELEGATE" | "TRANSFORM" | "PROTECT" | null,
    "reason": "<one sentence>",
    "reclaimable_minutes_per_week": <int or null>,
    "transform_change": "<concrete change, only if verdict is TRANSFORM, else null>"
  },
  "leadership_observation": {
    "strengths":     [ "<one strength bullet, second-person voice, anchored in evidence>" ],
    "growth_areas":  [ "<one growth-area bullet, constructive tone, with one concrete suggestion>" ]
  },
  "notes": "<any caveats about reliability — e.g., 'transcript truncated at 35 minutes', 'speaker labels unclear'>"
}

Rules:
- For one-off meetings, set `worth_matrix.applies: false` and all worth_matrix sub-fields to null.
- For meetings that match `non_negotiables` (provided in input below), set `worth_matrix.verdict: "PROTECT"` regardless of the dimension scores.
- For meetings that match `already_flagged_to_cut`, set `worth_matrix.verdict: null` and `worth_matrix.reason: "Already flagged for removal — no audit needed."`.
- If your transcript-fetch returns an empty / null transcript or a "not yet processed" status, set `dimensions: null` and `notes: "Transcript not yet processed by meeting tool — metadata-only review."` Then score what you can from metadata (title, attendees, duration, recurrence).
- If you cannot identify the executive in the transcript, set `leadership_observation` to `{ "strengths": [], "growth_areas": [], "notes": "Could not identify executive in transcript by name match." }`.

Preferences input (passed by the main agent):
- non_negotiables: <list>
- already_flagged_to_cut: <list>
```

## Substitution at dispatch time

The main agent substitutes:
- `<exec_name>`, `<exec_email>` from the local config
- `<meeting_title>`, `<date_time>`, `<duration_minutes>`, `<attendee_list>`, recurrence marker from the meeting tool's metadata
- `<id>` from the meeting tool's stable ID (sub-agent uses this to fetch the transcript)
- `<transcript_fetch_action_slug>` from the discovery call at audit Step 2 (e.g., `FATHOM_GET_RECORDING_TRANSCRIPT`)
- `<meeting_tool_account_id>` from the local config (null for single-account / direct vendor MCPs)
- `non_negotiables` and `already_flagged_to_cut` from the preferences file

## Why each sub-agent fetches its own transcript

Letting the main agent fetch all transcripts and pass them inline doesn't scale — meeting transcripts are often large enough to trigger aggregator auto-offload (Composio dumps responses ≳ 50k tokens to a remote sandbox file and returns a path + preview), which would force the main agent to slice per-meeting transcripts out of an offloaded file before fan-out. Per-sub-agent fetch sidesteps this: each sub-agent's response is bounded by one meeting, and the sub-agent owns the fetch + offload-handling logic locally. It also makes the run resilient to aggregator-MCP disconnect mid-run (fresh discovery + fresh fetch per sub-agent, no shared sandbox state to lose).

## Why one sub-agent per meeting

Energy-audit batches related meetings into clustered groups. This plugin deliberately does not. Each meeting gets its own focused review for two reasons:

1. **Cleaner attribution.** The leadership observation is meeting-specific and depends on parsing speaker labels — easier to reason about in isolation.
2. **Smaller per-call context.** A single transcript fits cleanly in one sub-agent's context. Batching three transcripts of 60-90 minutes each can blow up token usage.

The trade-off is more sub-agent calls (one per meeting in the window — typically 5-25 per run). For a weekly cadence on a 7-day window this is acceptable.
