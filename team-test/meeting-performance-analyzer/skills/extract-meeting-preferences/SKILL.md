---
name: extract-meeting-preferences
description: Captures the executive's known opinions about recurring meetings into a single canonical preferences file. Three fields — non-negotiables (always PROTECT), already-flagged-to-cut (skip the verdict), and restricted types (excluded from the brief entirely). Doc-upload-first — if the user provides a meeting-prefs doc, read it for the three fields and only ask for what's missing. Resumable. Re-runnable to refresh any field.
when_to_use: Run once after `setup`, and any time the executive's opinions about their recurring meetings change. Trigger phrases — "extract meeting preferences", "capture meeting preferences", "refresh meeting preferences", "update meeting prefs". Auto-trigger when `meeting-performance-audit` reports no preferences file exists at `client-profile/meeting-performance-analyzer-preferences.md`.
atlas_methodology: opinionated
---

# extract-meeting-preferences

Captures the executive's known opinions about recurring meetings into one canonical preferences file. The audit reads this file at run-time to enforce the executive's pre-stated rules:

- **Non-negotiables** — recurring meetings the executive considers permanent. The audit always categorizes these as PROTECT (never recommends cutting them).
- **Already-flagged-to-cut** — recurring meetings the executive has already decided to drop. The audit skips the verdict charade for these.
- **Restricted types** — meetings that should never appear in the brief at all (e.g., sensitive 1:1s, board sessions, legal review). Excluded entirely from analysis and from the brief output.

## Purpose

Without this file, the audit would have to ask the executive's opinion on every recurring meeting it analyzes — undermining the whole point of automation. This skill captures those opinions once, in a structured file, so the audit can run quietly.

## Inputs

- **Existing preferences file** (optional) — if `<workspace>/client-profile/meeting-performance-analyzer-preferences.md` exists, the skill enters refresh mode automatically.
- **Optional uploaded doc** — the user can paste or attach a doc that already enumerates their non-negotiables, planned cuts, and restricted types. The skill parses it for the three fields and only asks the user for what's missing.
- **Conversational interview** — questions asked one at a time when fields are not covered by an uploaded doc.

## Required capabilities

- File read + write (for the preferences file)
- Conversational interview (one question at a time)
- (Optional) document parsing for the doc-upload-first branch — typically just reading the supplied text and pattern-matching for the three fields

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The preferences file (`client-profile/meeting-performance-analyzer-preferences.md`) resolves relative to this directory. Do NOT write into the plugin's own install directory.

### 0. Detection (always first)

**Resolve `client-profile/meeting-performance-analyzer-preferences.md` against the workspace, NOT the plugin install directory.**

Two checks:

1. **Existing preferences file** at `<workspace>/client-profile/meeting-performance-analyzer-preferences.md`.
2. **Optional uploaded doc** — if the user pasted/attached a meeting-prefs doc when invoking this skill, capture it for parsing in Step 1.

Branch:

| Condition | Branch |
|---|---|
| Preferences file exists | **Refresh mode** (jump to "Refresh mode" section below) |
| Uploaded doc provided, no existing file | **Doc-upload-first path** (Step 1: parse, then Step 2: ask only for missing fields) |
| Nothing exists, no doc provided | **Fresh capture** (Step 2: ask all three fields) |

---

### 1. Doc-upload-first parsing (only if a doc was provided)

Read the uploaded doc. For each of the three fields, look for a labeled section or list:

- **Non-negotiables** — sections titled "non-negotiables," "must-keep," "always protect," "permanent meetings," "never cut," etc.
- **Already-flagged-to-cut** — sections titled "to cut," "planned cuts," "to drop," "killing," "removing," etc.
- **Restricted types** — sections titled "restricted," "private," "do not analyze," "exclude," "confidential meetings," etc.

For each field, extract a list of meeting names (or types). If a section is absent or empty, mark that field as "needs ask" for Step 2.

Show the user what was extracted:

> "I read your prefs doc and found:
> - **Non-negotiables**: [list, or 'nothing — I'll ask in a moment']
> - **Already-flagged-to-cut**: [list, or 'nothing']
> - **Restricted types**: [list, or 'nothing']
>
> Does this look right? I'll ask about the missing fields next."

If the user corrects an extraction (e.g., "actually, the board sync should be in restricted, not non-negotiable"), update accordingly before continuing.

---

### 2. Capture missing fields (one question at a time)

For each field NOT covered by an uploaded doc, ask:

**Non-negotiables.**
> "Which recurring meetings are non-negotiable for the executive? These are meetings the audit should always categorize as PROTECT — full prep, never recommend cutting. List them by name (one per line, or comma-separated). Skip if none."

**Already-flagged-to-cut.**
> "Which recurring meetings has the executive already decided to drop? These are meetings the audit should skip the verdict on (no need to re-justify the cut). Skip if none."

**Restricted types.**
> "Which meetings or meeting types should NOT appear in the brief at all? These are sensitive meetings (e.g., 1:1s with specific people, legal review, board sessions) that the audit should exclude from analysis entirely. List by name or by recognizable pattern. Skip if none."

Capture each answer as a list (or empty list if skipped). Paraphrase back for confirmation:

> "Got it. So:
> - Non-negotiables: [list]
> - Already-flagged-to-cut: [list]
> - Restricted types: [list]
>
> All set?"

---

### 3. Write `client-profile/meeting-performance-analyzer-preferences.md`

Write the captured values to `<workspace>/client-profile/meeting-performance-analyzer-preferences.md`. Format (YAML frontmatter, no markdown body):

```yaml
---
# Recurring meetings the executive considers permanent — audit always categorizes these as PROTECT
non_negotiables:
  - "<meeting name>"

# Recurring meetings the executive has already decided to drop — audit skips the verdict
already_flagged_to_cut:
  - "<meeting name>"

# Meetings that should not appear in the brief at all — audit excludes from analysis
restricted_types:
  - "<meeting name or pattern>"
---
```

If a field is empty, write the key with an empty list (`non_negotiables: []`) — the audit's null-check is a list-empty check.

---

### Refresh mode (alternative entry from Step 0)

If `client-profile/meeting-performance-analyzer-preferences.md` already exists, load the current values and walk through them per-field:

> "Current non-negotiables:
> - [list]
>
> Keep, add, remove, or replace?"

Repeat for `already_flagged_to_cut` and `restricted_types`. Save the updated file (overwrite) at the end.

## Output

A `client-profile/meeting-performance-analyzer-preferences.md` file with the three fields. The audit reads this file at its start and enforces the rules.

## Customization

To change preferences: re-run this skill — refresh mode walks through changes per-field.

To pre-populate from an existing doc: provide the doc when invoking this skill — the doc-upload-first parser reads it and only asks for missing fields.
