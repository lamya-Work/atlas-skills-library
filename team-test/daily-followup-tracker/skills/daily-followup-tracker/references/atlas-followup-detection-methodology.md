# Atlas Follow-Up Detection Methodology

The methodology behind `daily-followup-tracker`. Read this before tuning detection patterns, confidence cutoffs, or draft generation rules. Each rule is here because the failure mode of getting it wrong was costly enough to encode.

## What counts as a commitment

A commitment is a *promise made by the user, addressed to a specific person, with implied or explicit action the user owes back*. All three parts matter: it has to come from the user (not someone else committing on their behalf, and not the user acknowledging someone else's commitment), it has to land with a specific recipient (not a thrown-out hypothetical to a group), and it has to imply action — even if vague — that the user is on the hook to perform. If any of those three is missing, it is not a commitment for the purposes of this skill, and treating it as one will produce drafts that feel like noise. The cleanest mental model: would the recipient reasonably expect a follow-up from the user because of this message? If yes, it's a commitment. If they'd be surprised to hear from the user about it, it isn't.

## Commitment-language patterns

The patterns below are ranked by how reliably they predict an actual commitment. Match against the user's spoken or written text only — never against incoming text from others.

- **"I'll [verb]"** — high signal. The contraction-plus-verb pairing is the cleanest commitment marker in English.
- **"I will [verb]"** — high signal. Slightly more formal register; same reliability.
- **"Let me [verb]"** — high signal when followed by a concrete verb ("let me pull those numbers"); medium when the verb is soft ("let me think about it").
- **"Will get back to you"** — high signal. Almost always a real commitment, even without a deadline.
- **"Will follow up"** — high signal.
- **"Will [verb] by [date]"** — high signal with deadline; the deadline raises confidence further.
- **"Once I [verb]"** / **"After I [verb]"** — medium-to-high. The user is committing to a downstream action conditional on something else completing.
- **"Going to [verb]"** — medium. Slightly weaker than "I'll" because it sometimes describes a state of mind rather than a promise ("I'm going to be honest, that doesn't work for me" is not a commitment).
- **Hedged forms — "Might [verb]"**, **"Maybe I'll [verb]"**, **"If I can"**, **"Let me see"**, **"I'll try to"** — low signal. These are the trap. The user said *something*, but they explicitly preserved an out. Drafting a follow-up on a hedge tells the user "we ignored your hedge" — that's the trust burn this methodology exists to prevent. Capture them, surface them at low confidence, never auto-draft.

## Confidence rules

Three buckets. Every finding is assigned exactly one.

- **High** — explicit verb + clear recipient + (optional) deadline. The user's wording leaves no room for interpretation. *"I'll send the deck by EOD Thursday."*
- **Medium** — clear ownership but vague action OR no deadline. The user owns it; what exactly they'll do or when isn't pinned down. *"Let me look into that and circle back."*
- **Low** — hedge words present. The user softened the commitment with an explicit out. *"Might be able to swing that if I can find time."*

## What's NOT a commitment

The patterns below look like commitments at a glance but aren't. The detection logic must reject them.

- **Acknowledgements** — "yes, that makes sense", "got it", "noted", "understood". The user is confirming receipt or alignment, not promising action.
- **Hypotheticals** — "if we did X, then we'd Y", "we could probably Z if Q". Conditional reasoning, not a personal promise.
- **Other people's commitments** — "Sam said he'll pull the numbers Friday". This is a third-party hand-off — the user is reporting someone else's promise, not making one. The detector ignores these.
- **User asking a question** — "could you send that over by Thursday?". The user is creating an open loop on the *recipient's* side, not committing to anything themselves. The recipient now owes a reply; the user does not.
- **User nudging or chasing** — "Just gently bumping this..." / "Just following up on..." / "Any update on...?". The user is asking the recipient to act. The action is on the recipient's side, not the user's.
- **User directing a collaborator** — "here's what I need from you next week..." / "please go ahead and...". The user is delegating; the work item lives with the collaborator. The user owes nothing back unless they explicitly promised review or a follow-up action of their own.
- **User scheduling on someone else's behalf** — "My boss would like to meet — does Thursday at 2pm work?". The user is initiating a calendar request; the recipient owes the availability response. Until the recipient confirms, there is no user commitment to track. (Once confirmed, *sending the calendar invite* is a downstream user action — but that's a recipient-confirmation outcome, not a commitment captured at scan time.)

## Confidence-to-draft rules

Confidence drives drafting, not just sorting.

- **High** → draft inline in the report, ready to copy-send. The user can paste and go.
- **Medium** → draft inline with a `[DRAFT — VERIFY]` header. Signal to the user that the wording or the underlying commitment needs a quick eye before sending.
- **Low** → no draft. The finding is surfaced under *Possible follow-ups (low confidence)* with the source ref so the user can decide whether to act. Drafting on hedges is the single most common failure mode this skill needs to avoid.

## Topic normalization

The dedupe step at 5b and the cross-source resolution step at 5c both compare findings by their *normalized topic*. The same algorithm runs in both places.

1. **Lowercase** the topic string.
2. **Strip punctuation** (anything that isn't a letter, digit, or space).
3. **Drop stopwords**: `a`, `the`, `and`, `or`, `to`, `of`, `in`, `on`, `for`, `with`, `at`, `by`, `from`.
4. **Take the first 5 remaining words** in original order.
5. **Compare for exact match** against other findings' normalized topics.

The algorithm is deliberately simple. Fancier matching (stemming, semantic similarity) would catch more duplicates but also produces false positives — and a false positive at this step silently merges two distinct loops into one. Conservative beats clever.

## Task-source confidence cheatsheet

Tasks aren't conversations, so confidence is derived from the task's own state rather than language patterns. Use the row below that fits the task.

- **Overdue + no update in 5+ days** → **high**. Past due and silent — the user is on the hook and someone is probably waiting.
- **Overdue OR (no update in 5+ days, not yet due)** → **medium**. One signal of trouble, not both.
- **Due today + recent update** → **low**. Probably already in motion; the report should mention it but not push the user to act on it.
- **Due in 1 to `task_due_within_days` days** → **medium**. Approaching, not urgent.

## Filter operator cheatsheet (per task-property-type)

The task-scan agent introspects each container's schema, then picks the right operator for the assignee property's type. Provider-agnostic by design — the same logic works against Notion, Asana, Linear, Trello, ClickUp, Todoist, Monday, etc.

- **`people`** type — equality on user ID or email. The toolkit's "is" or "equals" operator.
- **`select` / `status`** type — equality on string value. The toolkit's "equals" operator on the option's name.
- **`multi_select`** type — contains-any on an array of strings. The toolkit's "contains" or "any of" operator.
- **`relation`** type — equality on the linked-row UUID. Resolve the user's row UUID once at the start of the run and cache it; relation filters need the row ID, not the user's email.
- **`text` / `rich_text`** type — equality on a string. Substring match if the toolkit supports it; exact match otherwise. Less reliable than typed properties — flag in the report if this is the only available match.

## Anti-patterns

What to avoid. Each one has caused a real bad output in past runs.

- **Detecting the same loop on three channels and drafting three follow-ups for it.** When the user mentioned a commitment in a meeting, then sent a Slack message about it, then wrote an email referencing it, naive detection produces three findings. Step 5c (cross-source resolution) is what prevents this. Never disable cross-source resolution to "see all the signals" — the report stops being useful the moment it nags the user three times for one promise.
- **Drafting on hedged "I'll see" / "let me think about it" / "might be able to."** These are explicit hedges. The methodology's confidence rules downgrade them to low; the draft logic must skip drafting at low confidence. Drafting on a hedge tells the user the system ignored the hedge — and a system that doesn't respect hedges gets turned off.
- **Including raw email bodies, transcript chunks, or full task descriptions in findings.** The orchestrator never sees raw content — only structured findings. The `commitment` field is a short quote or paraphrase, not the original. This is what makes the per-source fan-out a context-isolation win; leaking raw content back to the orchestrator throws away the win and risks spilling sensitive content into the report.
