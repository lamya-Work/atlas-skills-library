# meeting-preferences.md template

The canonical shape of `client-profile/meeting-preferences.md`. Used as a reference when the SKILL writes the file at Step 3 and as documentation when a user wants to hand-edit the file.

```yaml
---
# Recurring meetings the executive considers permanent — audit always categorizes these as PROTECT.
# These are meetings the executive has decided are worth the time, full prep, no question.
# Names should match how the meetings appear in your meeting tool's transcript titles.
non_negotiables:
  - "Weekly leadership team meeting"
  - "Monthly board sync"

# Recurring meetings the executive has already decided to drop — audit skips the verdict on these.
# Use when the executive has independently flagged a meeting for removal but hasn't yet acted.
# The audit will not waste time re-justifying the cut; it just confirms the meeting still happened in the window.
already_flagged_to_cut:
  - "Friday status standup"

# Meetings that should not appear in the brief at all — audit excludes from analysis entirely.
# Use for sensitive meetings: legal review, sensitive 1:1s, confidential board sessions, etc.
# Patterns are matched case-insensitively against meeting titles. Specific names are exact-match.
restricted_types:
  - "Legal review"
  - "1:1 with [HR contact]"
---
```

## Field semantics

- **`non_negotiables` overrides Worth Matrix verdicts.** If a recurring meeting matches an entry here, the audit emits a PROTECT verdict regardless of what the per-meeting sub-agent's quality scores would otherwise suggest.
- **`already_flagged_to_cut` short-circuits the Worth Matrix.** Recurring meetings matching here get a stub verdict in the brief: *"Already flagged for removal — no audit needed."*
- **`restricted_types` excludes from analysis.** Matches are dropped from the transcript list at Step 3 of the audit before any sub-agent dispatch. They do not appear in the brief at all (not even in the totals).

## Matching rules

- Names match against the meeting tool's reported title.
- Matching is case-insensitive.
- Substring match is used: `"1:1 with"` matches both "1:1 with John" and "Weekly 1:1 with the EA."
- An empty list (e.g., `non_negotiables: []`) means no rules — the audit applies its default behavior.

## Editing manually

The file can be edited directly with any text editor. Re-running `extract-meeting-preferences` is the recommended path because it validates the YAML structure on save, but a hand-edit will work as long as the YAML is valid.
