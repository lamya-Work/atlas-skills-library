# Atlas meeting quality dimensions

Seven dimensions used to score every meeting (recurring or one-off) in the audit. Each per-meeting sub-agent evaluates all seven against the transcript and metadata, returning a strong / mixed / weak rating per dimension plus a one-sentence rationale.

The brief's top-5 / bottom-3 ranking aggregates across all seven dimensions, weighted equally.

---

## 1. Decision output

**The question:** Were strategic decisions made in this meeting? Did anything change as a result of the conversation?

**Strong signals:**
- Explicit decision language: "we're going with X," "decided," "approved," "let's commit to."
- A trade-off was named and resolved.
- A choice was made between options that the meeting itself surfaced.

**Weak signals:**
- The meeting ended without resolving any of its stated questions.
- Discussion looped without progress; multiple "let's pick this up next time" moments.
- Status updates only — no inflection points.

---

## 2. Action items

**The question:** Did the meeting produce concrete next steps with owners and deadlines?

**Strong signals:**
- Each action item has a named owner ("Sam will...") and a specific timeline ("by Friday," "next week").
- Action items recap at the end of the meeting.
- Items are concrete enough that someone outside the meeting could understand what's been agreed.

**Weak signals:**
- Vague verbs: "we should think about," "someone should look into."
- No owner specified; "we" instead of a name.
- No deadline; "soon" or "eventually."

---

## 3. Time discipline

**The question:** Did the meeting start on time, end on time, and stay on the agenda?

**Strong signals:**
- Started within 2 minutes of scheduled time.
- Ended at or before scheduled end.
- The host kept off-topic threads short ("let's take that offline").
- Transitions between topics happened cleanly.

**Weak signals:**
- Started 5+ minutes late.
- Ran over by 10+ minutes.
- Whole sections of the agenda skipped because of digression.
- One topic consumed >50% of the time when the agenda allocated less.

---

## 4. Participation balance

**The question:** Did the right voices speak? Was airtime distributed appropriately for the meeting's purpose?

**Strong signals:**
- All required attendees contributed at least once.
- The host actively prompted quieter participants.
- Subject-matter experts spoke when their topics came up.

**Weak signals:**
- One or two voices dominated >70% of speaking time.
- Required attendees said nothing (and weren't explicitly there to listen).
- Decisions were made without input from the people most affected.

Note: balance is purpose-relative. A board update where the executive does most of the talking is *fine*. A working session where one VP dominates is not.

---

## 5. Async potential

**The question:** Could this meeting have been a Slack post, an email update, or a shared doc?

**Strong signals (the meeting WAS necessary, low async potential):**
- Real-time debate or back-and-forth that wouldn't survive async.
- Decision-making that needed multiple voices in the room simultaneously.
- A demo or walkthrough where seeing the thing live mattered.
- Sensitive material that warranted real-time presence.

**Weak signals (the meeting could have been async, high async potential):**
- One person presenting; others listening passively.
- A status readout that everyone could have read in 5 minutes.
- "Updates" with no questions asked or answered.

---

## 6. Agenda quality

**The question:** Was there a stated purpose? Was it referenced during the meeting?

**Strong signals:**
- Agenda named at the top of the meeting.
- The host steered back to the agenda when discussion drifted.
- Agenda items had time allocations that were respected.

**Weak signals:**
- No stated purpose at the start.
- "Anything for the good of the order?" energy throughout.
- Topics arose on the fly with no context.

---

## 7. Ownership / deadlines

**The question:** Were follow-ups concrete, assigned to a specific person, and tied to a specific time?

**Strong signals:**
- Every action item has a name attached.
- Deadlines are specific (date, day of week, "by EOD," etc.).
- Recap explicitly walks through each owner-deadline pair.

**Weak signals:**
- Action items exist but ownership is vague ("the team will...").
- Deadlines are open-ended ("at some point," "eventually").
- Recap doesn't happen, so the action items live only in the transcript.

This dimension overlaps with #2 (action items) — but #2 asks "were there action items at all?" while #7 asks "are the action items operationally complete?"

---

## Summary aggregation

Each per-meeting sub-agent returns a JSON record with seven `dimension: { rating: "strong" | "mixed" | "weak", evidence: "..." }` entries.

Equal weighting:
- Strong = 2 points
- Mixed = 1 point
- Weak = 0 points

Top-5 / bottom-3 ranking is by total score across the seven dimensions (max 14, min 0). Ties broken by recurrence (recurring meetings rank higher in bottom-3 because they have more leverage for change).
