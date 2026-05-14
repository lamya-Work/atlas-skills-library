---
name: scan-ideal-week
description: Scans the executive's calendar for a target window against the documented ideal week, flags every event that violates a rule with severity (block / warning / nudge), generates a concrete reschedule suggestion for each flagged event, and sends a single summary notification. Designed to run on a schedule (default end-of-day for tomorrow, morning-of for today) but can also be invoked manually.
when_to_use: Run after extract-ideal-week. Trigger phrases — "scan calendar", "scan ideal week", "check tomorrow's calendar", "flag calendar conflicts", "calendar guardian scan", "what's wrong with tomorrow", "is today's calendar clean". Also triggered by the scheduler set up at the end of extract-ideal-week.
atlas_methodology: opinionated
---

# scan-ideal-week

Compare the calendar to the documented ideal week, flag every violation with a fix suggestion, send one notification.

## Purpose

The exec's ideal week is enforceable only if something is checking the calendar against it on a cadence. This skill is that check. Loads the canonical ideal-week document, pulls calendar events for the target window, runs every event through every rule, produces a structured flag list, and sends one summary notification through the configured channel.

## Inputs

- **Target window** — default: `tomorrow` if invoked at or after 4pm; `today` otherwise (covers all hours from midnight through 4pm). Override with explicit dates if needed.
- **Ideal week document** — read from `ideal_week_path` in the local config (default `client-profile/ideal-week.md`)
- **Local config** — `<workspace>/client-profile/ideal-week-ops.local.md` for paths, notification channel, scan window, severity overrides

## Required capabilities

- **Calendar read** — list events for the target date range across all relevant calendar accounts
- **File read** — read the ideal-week document and the local config
- **Notification send** — deliver one summary message to the configured channel
- **(Optional) Date / time math** — for buffer calculations between events. Use the runtime's date facilities; do not implement timezone logic in the skill body.

## Steps

### 0. Precondition check — config required

Before any other logic, check `<workspace>/client-profile/ideal-week-ops.local.md` exists AND contains the required fields. If absent, also check `<workspace>/.claude/ideal-week-ops.local.md` (legacy path — see hard-fail branches below).

**Required fields:**
- `calendar_accounts` (non-empty list)
- `log_folder` (non-empty string)
- `notification_channel` (one of `gmail`, `slack`, `outlook`, `none`)
- `notification_target` (required only if `notification_channel` is not `none`; can be empty/absent otherwise)
- `ideal_week_path` (non-empty string)

**Hard-fail branches:**

If the file exists ONLY at the legacy `.claude/` path:
> "Your ideal-week-ops wiring config is at the old `.claude/` path and hasn't been migrated. Run `/ideal-week-ops:setup` once — it will copy the file forward to `client-profile/` and pick up where you left off."

If neither path has the file, or if the file at the new path is missing required fields:
> "Cannot scan — `client-profile/ideal-week-ops.local.md` is [missing | missing required field <name>]. Run `setup` first to wire calendar + notification, then re-run scan."

Do NOT attempt to use defaults or fall back. The user needs to know setup is the precondition; silent fallback masks the real issue.

**If `notification_channel` is set to a value other than `none` AND that channel has no corresponding wired tool** (e.g., `slack` with no Slack tool loaded): hard-fail with a clear message pointing at re-running `setup`. (If `notification_channel` is `none`, no ping is needed and this check is skipped — the log file is still always written.)

After the precondition check passes:
1. Read the ideal-week document at the configured `ideal_week_path`. If absent or unparseable per `../../references/ideal-week-format.md`, hard-fail with: "Cannot scan — ideal-week document at <ideal_week_path> [is missing | could not be parsed]. Run `extract-ideal-week` first."
2. Resolve the target window from inputs or invocation time (per "Pull calendar events" Step 1's window rule).
3. Detect dry-run mode (config flag `dry_run: true` or skill argument). In dry-run, the notification is rendered and printed to terminal but not actually sent.

### 1. Pull calendar events

Use the available calendar tool to list events for the scan window.

**Calendar accounts come from `client-profile/ideal-week-ops.local.md`** (`calendar_accounts:` list). The format is `<provider>:<account-identifier>` — e.g., `googlecalendar:sam@atlas.co`.

**Window**:
- If invocation time is at or after 4pm local: `scan_window_start = tomorrow 00:00`, `scan_window_end = tomorrow 23:59`
- Otherwise: `scan_window_start = today 00:00`, `scan_window_end = today 23:59`
- If `scan_window_days > 1` in config, extend `scan_window_end` accordingly.

**For each account in `calendar_accounts`**, call the available calendar tool's "list events" function with:
- The account identifier as the connected-account / target-account parameter (Composio: `connected_account_id` or matching toolkit + email; direct MCP: whatever its account-selection convention is)
- `start = scan_window_start`, `end = scan_window_end`

Collect all events into a single list, **tagging each event with which account it came from** (used in the grouped output later: `event.source_calendar = "googlecalendar:sam@atlas.co"`).

**De-duplicate** events that appear on multiple accounts:
- Primary key: `iCalUID` if present
- Fallback: `(start, end, title)` tuple match

The merged, de-duplicated list is what gets passed to the rule check.

### Multi-calendar contract

If `calendar_accounts` has length > 1, the skill MUST query each entry separately and merge results. Single-calendar is a special case of the loop, not a different code path.

The fetch + dedupe logic in "Pull calendar events" already handles this (loop over the list, tag each event, dedupe by iCalUID). This section is a **contract** to prevent silent regressions — any future edit that introduces a "use the first calendar account" shortcut breaks the multi-calendar feature.

**All events count equally regardless of source calendar.** No per-calendar rule scoping. If a rule is "no meetings before 9am" and there's an 8am personal-calendar event, it gets flagged. The user picked to merge — they own that.

### Multi-calendar grouped output

When rendering the output (both the log file and any notification ping), the format depends on the number of calendar accounts in config:

**1 entry in `calendar_accounts`** → single flag list, no calendar labels:

> *Tomorrow's calendar:*
> - Tuesday 8am dentist violates no-meetings-before-9am. Suggest: move to Friday morning.
> - Tuesday 2pm board prep overlaps deep-work block. Suggest: move to Wednesday afternoon.

**2+ entries in `calendar_accounts`** → flags grouped per source calendar:

> *Tomorrow's calendar:*
>
> **Work calendar (`sam@atlas.co`):**
> - Tuesday 2pm board prep overlaps deep-work block. Suggest: move to Wednesday afternoon.
>
> **Personal calendar (`sam@personal.com`):**
> - Tuesday 8am dentist violates no-meetings-before-9am. Suggest: reschedule for after 9.

Cross-calendar conflicts (e.g., 8am dentist on personal AND 8am internal standup on work) are caught by the rule check (both events go through the same merged list). The flag appears under whichever calendar holds the violating event — typically both, with one flag in each calendar's section.

### 2. Run the rule engine

For each day in the window, evaluate the documented ideal week against the day's events. Categories of checks (full taxonomy in `references/atlas-calendar-enforcement-methodology.md`):

- **Protected-block violations.** Any event scheduled inside a protected block (e.g., thinking time 7-8am, deep-work day, lunch window) → severity `block` unless the event title or organizer matches a VIP override.
- **NEVER rule violations.** Any event matching a NEVER rule (e.g., meeting on Tuesday/Wednesday for a no-meeting-day exec, back-to-back without buffer, before-workday-start) → severity `block` unless VIP override.
- **Cap violations.** Sum of meeting hours per day exceeds the configured daily cap (e.g., 4-hour ceiling). → severity `block` if hard cap, `warning` if target cap.
- **PREFER rule violations.** Soft-pattern violations (e.g., 1on1 not on a Thursday for a Thursday-batched exec, external call not on Mon/Thu/Fri) → severity `warning` or `nudge` per rule definition.
- **Overlap (double-booking).** Two or more events whose time ranges overlap → severity `block`. The exec literally cannot attend both. Distinct from a buffer violation; this is a true conflict, not just tightness.
- **Buffer violations.** Two meetings less than the configured buffer apart (default 15 min) but not overlapping → severity `warning`.
- **Missing prep blocks.** External calls without a 15-min prep block in front → severity `warning`.

For every flagged event, generate a **concrete suggestion** — name a specific alternative slot or action ("move to Thursday afternoon block", "shorten to 30 min", "add 15-min prep at 9:45"). A flag without a suggestion is invalid output and must be regenerated.

### 3. VIP override pass

For every flag, check the event's organizer/attendees against the VIP override list in the ideal-week document. If matched, downgrade severity by one level (block → warning, warning → nudge, nudge → drop). Annotate the flag with "VIP override applied: <name>".

### 4. Build the summary

**READ `../../references/scan-output-format.md` before composing the summary.** That reference file is the source of truth for the output template — header line shape, severity emoji (🚫 / ⚠️ / 💡), two-line flag entry format (identifier line + `→` indented suggestion line), `## Day` headers, footer format. Do NOT improvise the format from this skill body alone. The structure summary below is a quick orientation, not a substitute for reading the reference:

- Header: target window + scan timestamp + total flag count by severity
- One section per day, ordered chronologically
- Within each day: blocks first, then warnings, then nudges
- Each flag: event title + time + rule violated + suggestion
- Footer: clean-day note if any day in the window has zero flags

If the entire window is clean, the summary is a single line: "Calendar clean for <window>. No flags."

### 5. Output

The scan has two outputs: an always-on log file, and an optional notification ping.

**5a. Write the log file (always).**

Write a per-day log file to `log_folder` from config. Use read-then-append safety — see "Log file safety" section below. This is the canonical output: the user has history of every scan even if they never receive or look at notifications.

**5b. Send the notification ping (only if `notification_channel` is not `none`).**

Use the available notification tool. **Channel and recipient come from `client-profile/ideal-week-ops.local.md`** (`notification_channel`, `notification_target`).

**Render format per channel:**

- **`gmail`** — HTML email. Subject: `Ideal-week scan — <date> — <N> flag(s)`. Body: HTML rendering of the flags grouped by source calendar (if multi-calendar). Send via the available Gmail tool's "send message" function with `to = notification_target`.
- **`slack`** — markdown blocks. Group flags by source calendar if `calendar_accounts` has 2+ entries (see "Multi-calendar grouped output" above). Use Slack mrkdwn (`*bold*`, `_italic_`, ``code``). Send via the available Slack tool's "send message" function with `target = notification_target`.
- **`outlook`** — HTML email, same shape as Gmail. Send via the available Outlook tool's "send message" function with `to = notification_target`.

If `dry_run: true` in config: render the notification body but DON'T call the send tool. Print the body to terminal so the user can verify wiring. (The log file is also skipped in dry-run; see "Log file safety" below.)

**Channel mismatch handling:** If `notification_channel` (other than `none`) doesn't match any wired tool (e.g., config says `slack` but no Slack tool is loaded), abort the ping with a clear message: "Config says channel=slack but no Slack tool is wired. The log file was still written to <log_folder>. Re-run `setup` to fix the ping."

### Log file safety

The scan ALWAYS writes a per-day markdown log file to `log_folder` from config — regardless of whether `notification_channel` is set. **NEVER overwrite — always read-then-append.**

Logic:

1. Compute file path: `<log_folder>/<YYYY-MM-DD>.md`. Use the local date the scan was invoked. Example: `client-profile/ideal-week-scans/2026-04-30.md`.

2. **If the file does not exist**: create it with a top-level header (`# Ideal-week scans — 2026-04-30`) and write the new section below.

3. **If the file already exists**: READ its full contents first. Then write back the existing contents PLUS a new appended section. This is the critical safety contract — overwriting would clobber the morning scan when the evening scan runs (or vice versa).

Each section is timestamped with HH:MM so morning + evening scans are distinguishable:

````markdown
## Evening scan — 2026-04-30 17:00 (looking at tomorrow)

[flags rendered here]

---

## Morning scan — 2026-04-30 07:02 (looking at today)

[flags rendered here]

---
````

The order of sections in the file follows write order — the log file accumulates over the day.

**If the configured `log_folder` doesn't exist**: create it (recursively) before the first write.

**If `dry_run: true`**: don't write to disk; print the rendered section to terminal so the user can preview.

### 6. Persist scan log

Write a one-line log entry to `<workspace>/client-profile/ideal-week-ops.scan-log.jsonl` (create the file and the `client-profile/` directory if missing) with: timestamp, window, flag counts, channel, send-success. The log is for debugging "did the scan run?" — not for analytics. Do NOT consult any legacy `<workspace>/.claude/ideal-week-ops.scan-log.jsonl` — old entries are not migrated; the scan log effectively starts fresh at the new path.

## Output

The notification body itself (the summary built in step 4). In dry-run mode, also printed to terminal. In live mode, the skill returns a one-line confirmation: `Scan complete — N flags sent to <channel>`.

## Customization

- **Severity mapping.** Per-rule severities live in the ideal-week document. The scan skill reads them — it does not embed defaults. To change the meaning of "block" vs "warning" globally, edit `references/atlas-calendar-enforcement-methodology.md`.
- **Suggestion engine.** The "where to move it" suggestion logic is per the methodology reference. The default looks for the next-available slot inside the same day's allowed blocks. Override per-rule in the ideal-week document if a specific rule needs a different suggestion strategy (e.g., "always suggest declining, never moving").
- **Notification format.** The channel-specific renderer (Slack vs Gmail vs Outlook) is part of the skill body itself — see Step 5 below. Edit the format strings in Step 5 (or this skill's body more broadly) to change how notifications render per channel.
- **Window detection.** The "tomorrow if afternoon, today if morning" rule can be overridden by passing an explicit window argument when invoking the skill, or by setting `default_window:` in the local config.

## Why opinionated

The rule taxonomy (protected-block / NEVER / cap / PREFER / buffer / missing-prep) and the severity model (block / warning / nudge with VIP downgrade) are Atlas's specific approach to calendar enforcement. The full methodology, including why VIP overrides downgrade rather than skip, lives in `references/atlas-calendar-enforcement-methodology.md`. Clients wanting different rule categories or severity mechanics should fork the references — keep the skill body stable.
