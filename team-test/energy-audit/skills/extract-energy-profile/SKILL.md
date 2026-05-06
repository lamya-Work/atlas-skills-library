---
name: extract-energy-profile
description: Captures the executive's energy profile — Zone of Genius, drains, recurring patterns, quarterly priorities — into a single canonical document the audit reads. Up front, asks if the user has any existing prefs doc (OKRs, "how I work" page, drains list) — if yes, drafts the profile from that and only asks the questions that aren't already answered. Otherwise walks the four questions one at a time. Resumable mid-flow. Soft precondition check on `setup`.
when_to_use: Run after `setup`. Trigger phrases — "extract energy profile", "set up exec energy profile", "refresh my energy profile", "redo energy profile", "update my energy preferences". Also auto-trigger when `energy-audit` reports no profile exists at `client-profile/exec-energy-profile.md`.
atlas_methodology: opinionated
---

# extract-energy-profile

Captures the four content fields the audit reads — Zone of Genius, drains, recurring patterns, quarterly priorities — and saves them to `client-profile/exec-energy-profile.md`. Two paths: parse-existing-doc (if the user already has notes) or question-by-question. Resumable. Re-runnable.

## Purpose

The `energy-audit` skill needs the executive's energy profile to filter and rank automation candidates. This skill captures that profile in a single canonical document. It's deliberately content-only — wiring (calendar / task tool / meetings / Slack / notification) lives in `.claude/energy-audit.local.md`, written by `setup`.

The four content fields:

1. **Zone of Genius** — work that makes the executive lose track of time and feel most effective. Used by the audit as a *negative* filter: candidates that pull toward Zone-of-Genius work do NOT get automated.
2. **Drains** — recurring tasks that drain or get procrastinated. Used as the *primary* filter: candidates aligned to a drain rank highest.
3. **Recurring patterns** — "I'm always the one who deals with X" tasks, even when not draining. Used as a secondary filter for repetitive work.
4. **Quarterly priorities** — top 1–3 priorities for the current quarter. Used as a tiebreaker — candidates connected to a stated priority rank above otherwise-equivalent candidates.

The skill is opinionated about staying short. Four questions, free-form answers, no scoring. Atlas's experience: longer interviews go shallow; the four questions surface enough signal for the audit's hard cap of 2 plans.

## Inputs

- **Existing profile** (optional) — if `client-profile/exec-energy-profile.md` exists, the skill enters refresh mode automatically.
- **In-progress marker** (optional) — `.energy-profile-in-progress.json` in the workspace, if a previous run was interrupted.
- **Existing prefs doc** (optional) — if the user has an OKRs page, "how I work" doc, drains list, etc., the skill drafts from that and only asks the questions that aren't already covered.
- **Conversational interview** — questions asked one at a time, paraphrased back for confirmation.

## Required capabilities

(Abstract list — no specific tool names.)

- File read + write (for the profile, prefs doc, and in-progress marker)
- Conversational interview (one question at a time)

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The profile (`client-profile/exec-energy-profile.md`), in-progress marker (`.energy-profile-in-progress.json`), and any user-supplied prefs doc all resolve relative to this directory. Do NOT write or read these from the plugin's own install directory. Plugin-internal references (`references/onboarding-questions.md`, the profile template at `../../client-profile/templates/exec-energy-profile.template.md`) use `../../` from this skill folder to mean plugin root.
>
> **Resumability rule.** After every captured answer in Steps 2–4, save running state to `<workspace>/.energy-profile-in-progress.json` so the skill can resume mid-flow if the conversation is interrupted.

### 0. Soft precondition check

Look for `<workspace>/.claude/energy-audit.local.md`. If absent, surface this soft warning:

> "Heads up — `setup` hasn't run yet, so I don't know which tools you've connected. You can keep going with this and connect tools later, or run `setup` first. What do you want to do?"

Allow proceeding either way. The profile and the local config are independent — the audit reads both, but extracting the profile doesn't strictly require the wiring to be in place.

**Resolve the path against the workspace, NOT the plugin install directory. Don't fall back to checking the plugin folder if the file isn't in the workspace.**

### 1. Phase detection

Workspace-relative checks:

| Condition | Branch |
|---|---|
| `<workspace>/client-profile/exec-energy-profile.md` exists | **Refresh mode** (jump to Step 5) |
| `<workspace>/.energy-profile-in-progress.json` exists | **Resumed session** — load running state, jump to last unanswered question |
| Neither exists | **Fresh extraction** (continue to Step 2) |

### 2. Existing prefs doc check (fresh extraction only)

Ask:

> "Before we go through questions — do you have anything written already? An exec preferences doc, OKRs, a 'how I work' page, a drains list, anything like that. If yes, point me to where it is or paste it here and I'll use what's relevant. If not, no worries — we'll go through it together."

**If yes** — read whatever the user shares (file path or pasted content). Draft a candidate profile from whichever of the four fields the doc covers. Show the user what was drafted and what's still blank:

> "Here's what I drafted from your doc:
> - **Zone of Genius:** [drafted | not covered in your doc]
> - **Drains:** [drafted | not covered in your doc]
> - **Recurring patterns:** [drafted | not covered in your doc]
> - **Quarterly priorities:** [drafted | not covered in your doc]
>
> I'll go through the blank ones with you. The drafted ones — read them through and tell me anything to change."

Only ask the questions for blank fields in Step 3. The drafted-from-doc fields skip directly to Step 4 (Display + confirm).

**If no** — continue to Step 3 with all four questions queued.

### 3. The four content questions

Load the four questions from `references/onboarding-questions.md`. One per turn. Free-form answers. Paraphrase back for confirmation before moving to the next.

The four questions in order:

1. **Zone of Genius** — *"What work makes you lose track of time and feel most effective?"*
2. **Drains** — *"What recurring tasks drain you, or that you find yourself procrastinating on?"*
3. **Recurring patterns** — *"Things you do every week or month where you're 'always the one who deals with X'?"*
4. **Quarterly priorities** — *"Top one to three priorities this quarter? (The audit uses these to flag whether work connects to what matters most.)"*

After question 4, capture today's date as `Quarterly priorities last updated: YYYY-MM-DD`.

After every answer, save running state to `<workspace>/.energy-profile-in-progress.json` (resumability).

### 4. Display + confirm

Synthesize all four fields into a profile draft. Show:

> "Here's what I have. Read through it, tell me anything to change, and I'll save once you say it's right."

Iterate until the user confirms. If the user says "save it", proceed to Step 6.

### 5. Refresh mode (alternative entry from Step 1)

If `client-profile/exec-energy-profile.md` already exists:

For each of the four fields, show the current content and ask: *"Keep this, or change?"* If change, ask the corresponding Step 3 question and capture the new answer. Save running state after each.

After the field-by-field walk, fall through to Step 6 (write).

### 6. Save

Write the profile to `<workspace>/client-profile/exec-energy-profile.md` following the structure in `../../client-profile/templates/exec-energy-profile.template.md`:

- `# Exec Energy Profile` (H1)
- `## Zone of Genius` — answer prose
- `## Drains` — answer prose (typically a bulleted list)
- `## Recurring patterns` — answer prose (typically a bulleted list)
- `## Quarterly priorities` — answer prose (typically a numbered list of 1–3 priorities)
- `Quarterly priorities last updated: YYYY-MM-DD` (today's date in the user's local timezone)

If any of the four questions were skipped (e.g., the executive wasn't available to answer one), append a top-of-file marker: `Profile completeness: partial — [list of missing fields]`. The audit will still run with a partial profile and surface the gap in the report header.

If all four are present, no completeness marker is needed (or use `Profile completeness: complete`).

After writing, delete `<workspace>/.energy-profile-in-progress.json` if it exists.

### 7. Hand-off

> "Profile saved. Run `energy-audit` any time, or wait for the scheduled run if you set one up."

## Output

A `<workspace>/client-profile/exec-energy-profile.md` file with the four content fields and a quarterly-priorities-last-updated timestamp. The audit reads this file at its start.

In refresh mode, prefix the hand-off with "Updated."

## Customization

- The four onboarding questions live in `references/onboarding-questions.md`. Change the question wording there; the skill body's flow stays stable.
- The profile template at `../../client-profile/templates/exec-energy-profile.template.md` defines the section structure. Change the structure there.
- The resume marker filename is fixed (`.energy-profile-in-progress.json`).
- The profile path is fixed (`client-profile/exec-energy-profile.md`) so the audit can find it deterministically. To override, edit the `profile_path` field in `.claude/energy-audit.local.md` (written by `setup`).

## Why opinionated

Atlas's four-question content set is the deliberate minimum that surfaces enough signal for the audit's hard cap of 2 automation plans. Longer interviews go shallow; users get bored and answer in templates rather than specifics. The doc-upload-first path preserves what the executive already wrote — re-asking questions that have written answers is friction without value.

The profile structure also doesn't include scoring, weights, or quantitative fields. The audit applies its ranking logic at run time; the profile is qualitative input. Adding numbers here would create false precision.

This skill stays short on purpose.
