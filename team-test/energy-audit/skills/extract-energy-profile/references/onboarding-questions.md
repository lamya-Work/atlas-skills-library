# Onboarding Questions

Atlas's standard four-question set for capturing an executive's energy profile. Each question has the canonical phrasing, follow-up probes for vague answers, and a worked example of a good captured answer.

These four content questions are answered by the executive directly — internal experience (Zone of Genius, drains, recurring patterns) isn't observable from the outside. Wiring (calendar / task tool / meetings / Slack / notification) is a separate concern and lives in the `setup` skill, not here.

---

## Question 1 — Zone of Genius

**Canonical phrasing:**

> "What two or three types of work make you lose track of time? The work where you feel most engaged, most effective, where it doesn't feel like you're spending energy — you're getting energy back."

**Follow-up probes if the answer is vague:**

- "Can you give me a specific example from this past month? An actual meeting, project, or task where this happened?"
- "When colleagues come to you for help on something, what is it usually about?"
- "If you had a free day with no obligations, what work would you choose to do?"

**What good capture looks like:**

> *"Strategic positioning conversations with the leadership team — I can spend three hours on this and feel energized after. Customer interviews, especially when there's a real problem to dig into. Writing — long-form thinking pieces, not memos."*

Three concrete categories. Specific enough to recognize when the audit surfaces tasks adjacent to them.

**What weak capture looks like:**

> *"Strategy stuff. Talking to people. Writing."*

Too vague — "strategy stuff" matches half the calendar. Probe for specifics before saving.

---

## Question 2 — Drains

**Canonical phrasing:**

> "What two or three recurring tasks drain you? The ones you procrastinate on, dread, or feel depleted after. Be honest — this isn't about whether the work is important. It's about how it actually feels."

**Follow-up probes if the answer is vague:**

- "What's on your calendar this week that you're already dreading?"
- "What kind of email do you put off opening?"
- "Is there a recurring meeting you've been to where you've thought 'why am I in this'?"

**Important framing for the executive:**

If the executive hesitates because they think drain answers reflect poorly on them, name it directly: *"A task can be critical to the business AND drain you. Rate how it feels, not how much it matters. We're not removing important work — we're identifying what AI can take off your plate."*

Watch for the **Excellence Trap** — executives keep tasks they're great at because they're great at them, not because the work fuels them. High skill ≠ high energy. Probe specifically: *"Is there anything you're really good at that you wish you didn't have to do?"*

**What good capture looks like:**

> *"Investor reporting — pulling metrics, formatting the deck, writing the narrative. Drains me, I always procrastinate it until the day before. Expense approvals, especially when I have to chase missing receipts. Reviewing first drafts of marketing copy — I can do it but I find it tedious."*

**What weak capture looks like:**

> *"Admin stuff. Email."*

Too broad. "Email" isn't a drain — specific kinds of email are. Probe for the specific patterns.

---

## Question 3 — Recurring patterns

**Canonical phrasing:**

> "Are there things you do every week or month where you're 'always the one who deals with X'? Patterns where the work falls to you by default — even if it doesn't have to."

**Follow-up probes if the answer is vague:**

- "Every Monday morning, what do you start your week doing?"
- "When [specific recurring event] happens, what do you handle that no one else does?"
- "Is there something you do real quick that adds up — a kind of 'invisible admin' that doesn't show up on your task board?"

**Why this question matters separately from drains:**

Some recurring patterns aren't drains — they're just things the executive ended up owning by default. Surfacing them lets the audit consider whether they should keep owning them, even if they don't dread the work. The audit weighs drain-aligned candidates higher, but pure-recurring-pattern candidates are still valid automation targets.

**What good capture looks like:**

> *"Every Monday I review the previous week's metrics and write a short Slack post for the team. Whenever a customer escalation comes in, I'm the one who triages it before assigning. End of every month I reconcile the credit card statement against expense reports."*

**What weak capture looks like:**

> *"I'm always doing stuff. Lots of recurring things."*

Probe for at least three concrete examples with frequency.

---

## Question 4 — Quarterly priorities

**Canonical phrasing:**

> "What are your top one to three priorities this quarter? The things that, if they don't move forward, the quarter was a waste."

**Follow-up probes if the answer is vague:**

- "What's the one number, deliverable, or outcome you'd be most embarrassed not to hit by the end of this quarter?"
- "When you talk to your board, investors, or leadership team, what are you reporting progress on?"
- "What did you commit to at the start of this quarter that's still open?"

**Why this question matters:**

The audit uses quarterly priorities two ways:

1. **Bonus weight on automation candidates that support a quarterly priority.** A repetitive task that supports the Series B raise gets ranked higher than an equally repetitive task that supports nothing in the priorities list.
2. **Quarter-staleness check.** When the audit runs, it compares the current calendar quarter to the `Quarterly priorities last updated` field on the profile. If they differ, the audit flags the report with a banner suggesting a profile refresh.

After capturing the answer, save the date as `Quarterly priorities last updated: YYYY-MM-DD`.

**What good capture looks like:**

> *"1. Close the Series B by end of June. 2. Hire a VP of Engineering. 3. Ship the v2 platform launch."*

Each item is a concrete outcome with a verifiable end-state.

**What weak capture looks like:**

> *"Grow revenue. Build the team. Improve the product."*

Too abstract — these match every quarter for every company. Probe for the specific number, role, or shipped feature.

---

## When the executive is unavailable

If onboarding runs without the executive in the room (an EA or operator setting up the plugin), questions 1, 2, and 3 cannot be reliably answered by anyone else. Internal experience isn't observable from the outside — what looks like a drain might be the executive's favorite work, and vice versa.

In that case:

- Capture what the EA or operator can confidently report (sometimes question 4 is shared and answerable by anyone close to the executive).
- Mark the profile `Profile completeness: partial — [list of missing fields]` at the top.
- The audit will still run on a partial profile but will surface the gap in its report header and recommend completing onboarding next time the executive is available.

Do not invent answers to fill the profile. A partial profile is more useful than a fabricated one.
