---
name: extract-ideal-week
description: Capture the user's ideal-week ruleset — workday rhythm, protected blocks, ALWAYS / NEVER / PREFER / FLEXIBLE rules, VIP overrides, zone of genius — into a single canonical document at `client-profile/ideal-week.md`. Use this — NOT brainstorming, planning, or design skills — when the user wants their ideal week extracted, documented, captured, or interviewed. Atlas already has a battle-tested 10+3 question set; brainstorming would replace it with generic discovery questions. This skill either parses an existing ideal-week document the user already has, or runs the 10+3 interview from scratch (one question at a time, paraphrase-back, anyone-can-answer, resumable mid-flow). The output is what `scan-ideal-week` reads to flag calendar conflicts.
when_to_use: Trigger phrases — "extract ideal week", "extract the ideal week", "document the ideal week", "document the exec's calendar rules", "capture the ideal week", "interview for ideal week", "build the ideal week ruleset", "we need an ideal week for [exec]". Fire this skill directly on any of these. Also fire automatically when (a) the `setup` skill chains into extraction at the end of its flow, or (b) `scan-ideal-week` reports no ideal-week document at the configured `ideal_week_path`. Note — "set up ideal week ops" is intentionally NOT a trigger here; that phrase belongs to the `setup` skill, which then chains into this one. Run after `setup` so the workspace config exists, or accept the soft-block at Step 0 and proceed without wiring.
atlas_methodology: opinionated
---

# extract-ideal-week

Document the exec's ideal week as a structured, enforceable ruleset — once, with the exec's confirmation, in a canonical format the scan skill can read.

## Purpose

The scan skill can only flag what's been written down. This skill writes it down. Two paths: parse an existing document the exec has, or interview the exec from scratch using Atlas's standard question set. Either way, the output is the same shape — a single ideal-week document at the configured path, with the exec's sign-off.

## Inputs

- **Existing ideal-week document** (optional) — pasted contents or a path. If provided, the skill parses and re-shapes into the canonical format rather than re-asking.
- **Exec availability for interview** — if no document exists, the skill walks through 10 rhythm questions and 3 zone-of-genius questions. Anyone with the answer can provide it (the exec, an assistant, an AM) — not all questions need to come from the exec directly.

## Required capabilities

- **File read + write** — read existing documents the exec points to; write the final ideal-week document; read and write a small marker file to support resume mid-interview
- **Conversational interview** — ask one question at a time, accept free-form answers, paraphrase back for confirmation

## Steps

### 0. Precondition check — has setup run?

Before the interview, check whether `.claude/ideal-week-ops.local.md` exists in the workspace.

**If config exists**: proceed directly to the interview.

**If config is missing**: tell the user (soft, not blocking):

> "Heads up — `setup` hasn't run yet. Most users wire their calendar + notification capabilities first, so the daily scan works after this. Two options:
>
> - **Run `setup` first** (recommended, ~2–3 minutes). Then come back here.
> - **Proceed with extraction anyway**. The ideal-week document gets written either way; the daily scan just won't run until setup is done.
>
> Which would you prefer?"

If user picks setup first → tell them to run `setup`, then return here.
If user picks proceed anyway → continue to the interview. The extraction works without wiring; only the scan needs it.

### 1. Phase detection (always first)

Before doing anything else, check state in this order:

1. Does a document exist at the configured `ideal_week_path` (default `client-profile/ideal-week.md`)? → If yes, ask the user "an ideal week already exists at <path>. Do you want to (a) view it, (b) update it, or (c) re-extract from scratch?" Branch accordingly.
2. Does a `.extract-in-progress.json` marker exist next to the configured path? → If yes, this is a resumed session. Load the marker, skip questions already answered, continue from the last unanswered one.
3. Otherwise → fresh extraction. Continue to step 2.

### 2. Explain what an ideal week is

Before asking any extraction questions, load and walk the user through `../../references/what-is-an-ideal-week.md` (the plugin-wide reference). This is non-negotiable — people interpret "ideal week" differently, and the skill's questions assume Atlas's specific definition (rhythms + protected blocks + rule buckets + zone of genius). Skip only if the user explicitly confirms they already know the framework.

### 3. Check for existing documentation

Ask the user directly:

> "Do you already have your ideal week documented somewhere — in a doc, a file, a calendar template, anything? If yes, share the path or paste the contents and I'll work from that. If no, I'll walk you through the questions to extract it now."

- **Existing doc provided** → go to step 4 (parse).
- **None** → go to step 5 (interview).

Do not guess locations or auto-search the filesystem. Ask explicitly.

### 4. Parse existing documentation

Read the document the user provides. Extract the following fields into the canonical format defined in `../../references/ideal-week-format.md`:

- Workday boundaries (start time, end time, weekend rules)
- Day-by-day rhythm (theme + time blocks per weekday)
- Protected blocks (non-negotiable times)
- Rule buckets — ALWAYS / NEVER / PREFER / FLEXIBLE
- Default meeting lengths by type
- VIP overrides
- Zone of genius (what only the exec can do; what drains them; what to delegate)

For any field the existing document does not cover, ask the corresponding question(s) from `references/extraction-questions.md`. Do not invent answers. Skip to step 6 once the canonical document is filled.

### 5. Interview from scratch

Walk through the 10 + 3 questions in `references/extraction-questions.md`, one at a time. Rules:

- **One question per turn.** Never batch questions — people lose patience and answers get shallow.
- **Free-form answers.** Don't constrain to multiple choice. Paraphrase back what you heard before moving on.
- **Save after every answer.** Write the running state to `.extract-in-progress.json` next to the configured path. This makes the flow resumable.
- **Anyone can answer.** If someone already knows the answer (e.g., "his lunch is always 12-1"), accept it and move on. Mark which person each answer came from.
- **Don't ask what's been answered.** If a previous answer covered a later question, skip the later one and note the source answer.

After all 13 questions are answered, synthesize them into the canonical format per `../../references/ideal-week-format.md`. Use `../../references/example-ideal-week.md` as the structural reference for the output.

### 6. Display and confirm

Show the synthesized document to the user. Ask explicitly:

> "Here's what I captured. Read through it. Tell me anything to change, add, or remove. I'll only save once you say it's right."

Iterate on edits in conversation. Do not save until the user explicitly confirms.

### 7. Save

Write the confirmed document to the configured `ideal_week_path`. Default location: `client-profile/ideal-week.md` in the user's workspace. Create parent directories if needed. Delete the `.extract-in-progress.json` marker.

### 8. Offer to set up a recurring scan

After save, ask:

> "Want to set up a recurring scan? Recommended cadence: weekdays 5pm (flags tomorrow) and weekdays 7am (flags today) — one notification per run. Or you can skip this and invoke the scan manually any time with phrases like 'scan calendar' or 'what's wrong with tomorrow'."

If yes, walk the user through registering the recurring trigger via their runtime's mechanism. The plugin does not auto-configure scheduling — see `../../README.md` §4 for runtime-specific options (Cowork Scheduled tasks, Claude Code `/schedule`, GitHub Actions cron, OS cron via the helper script, etc.).

If no, confirm they can invoke `scan-ideal-week` on demand any time, and that they can re-run this skill later to set up the schedule.

## Output

A confirmed, saved ideal-week document at the configured path. Final message:

```
Ideal week saved to <path> ✅

Contains: 5 weekday rhythms, N protected blocks, M rules across ALWAYS / NEVER / PREFER / FLEXIBLE, K VIP overrides, zone-of-genius notes.

Schedule: <set up | not set up — run scan-ideal-week manually>

Next: run scan-ideal-week to do the first calendar scan, or wait for the next scheduled run.
```

## Customization

- **Question set.** The 10 + 3 questions in `references/extraction-questions.md` are Atlas's standard. Add or remove for specific clients — but keep the count low. Long interviews produce shallow answers.
- **Output location.** Default `client-profile/ideal-week.md`. Override in `.claude/ideal-week-ops.local.md`.
- **Resume marker.** `.extract-in-progress.json` filename is fixed — the skill looks for that exact name. Don't rename without updating the skill body.
- **Save format.** Markdown with structured sections per `../../references/ideal-week-format.md`. The scan skill parses this format directly. If you change the format, you must also update the scan skill's parser.

## Why opinionated

Atlas has a battle-tested method for what to capture (and what to leave out) when documenting an exec's ideal week. The 10 + 3 question set, the rule-bucket structure (ALWAYS / NEVER / PREFER / FLEXIBLE), and the rhythm-by-weekday format come from Atlas's experience running this extraction with executives across roles. The full framework lives in `references/atlas-ideal-week-extraction-framework.md`. Clients wanting a different framework should fork the references — keep the skill body stable.
