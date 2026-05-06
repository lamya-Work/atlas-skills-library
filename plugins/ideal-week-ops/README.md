# ideal-week-ops

> Capture an executive's ideal week, document enforceable scheduling rules, and run a daily calendar scan that flags conflicts and suggests fixes.
> v0.1.0

## What it does

Turns "how this exec wants their week to run" from tribal knowledge into a documented, enforced ruleset:

- **Extracts the ideal week once.** A guided onboarding flow either parses an existing ideal-week document the exec already has, or interviews them with Atlas's standard extraction questions (rhythms, deep-work blocks, protected time, meeting limits, VIP overrides, zone of genius). Synthesizes the answers into a structured, machine-readable ideal-week document and confirms it with the user before saving.
- **Documents rules in a canonical format.** Output is a single source-of-truth file: rhythm by day-of-week, ALWAYS / NEVER / PREFER / FLEXIBLE rule buckets, protected blocks, VIP override list, default meeting lengths, and zone-of-genius notes.
- **Enforces the rules on a schedule.** Runs twice a day by default — end-of-day for tomorrow, morning-of for today. Pulls calendar events, checks them against every rule, flags violations with severity, suggests concrete fixes ("move this to Thursday afternoon", "add 15-min buffer before X"), writes a per-day log file (always), and optionally pings a chosen channel (Gmail / Slack / Outlook).

The result: no one has to manually re-check the calendar against a mental model, and the exec gets one clean daily ping when something breaks the rules.

## Who it's for

Executives and the people who support them — chiefs of staff, executive assistants, operators, account managers, or executives running their own calendar with an agent. Atlas built this for execs who already know how their week should run but lose hours every week to meetings that violate that pattern, because no one is enforcing it.

If the exec doesn't yet have an ideal week, the extraction skill creates one from scratch via interview. If they do, the plugin parses it.

## Required capabilities

The plugin's skills depend on these capabilities. Each is named abstractly — wire it up to whatever tools the host agent has access to.

- **Calendar read** — list events for a given date or date range across all relevant calendar accounts (work + personal if both are scheduled into; the plugin supports merging multiple accounts)
- **File read + write** — read and write the ideal-week document, the plugin's local config file, and the per-day scan log files (always-on output)
- **Notification send** *(optional)* — send a single message to a chosen channel (Slack DM/channel, Gmail, or Outlook). If skipped, scans still produce a per-day log file the user can check.

## Suggested tool wiring

| Capability | Validated default | Alternatives |
|---|---|---|
| Calendar read | **Composio MCP** → Google Calendar or Outlook Calendar (OAuth, no API keys) | Anthropic's Google Calendar MCP, Outlook MCP, or any other MCP that exposes a "list events" tool |
| Notification send (optional) | **Composio MCP** → Slack / Gmail / Outlook (OAuth, no API keys) | Slack MCP, Gmail MCP, Outlook MCP — or skip entirely (the log file is still written) |
| File read + write | Native filesystem tools | (no alternative needed) |
| Recurring trigger (optional) | Whatever your runtime provides — Cowork Scheduled tasks, Claude Code `/schedule`, GitHub Actions cron, OS cron | The plugin is agnostic. Without a recurring trigger, the user invokes `scan-ideal-week` manually. |

**Composio MCP is the recommended wiring for both calendar and notification** because it routes through one connection layer the user configures once at https://app.composio.dev. The `setup` skill walks the user through Composio install + tool selection. If you'd rather wire your own MCPs, the `setup` skill detects them and offers to fill any gaps with Composio.

**Notification send is optional.** If the user doesn't want a live ping, scans still write a per-day log file at `log_folder` (default `client-profile/ideal-week-scans/`).

**Recurring trigger is NOT shipped with this plugin.** Wire it through whatever recurring-task feature your runtime provides — see the `setup` skill's hand-off message for runtime-specific suggestions.

## Installation

```
/plugin install ideal-week-ops@atlas
```

**After installing, run `setup` first.** The `setup` skill detects what's already wired in your AI host, walks you through Composio install if needed, captures your notification channel + recipient, writes a local config file, and chains directly into `extract-ideal-week` to capture how your week should actually run. Once both finish, `scan-ideal-week` is ready.

## First-run setup

1. **Run `setup`.** Detection-first onboarding. Walks you through Composio install (or detects your existing MCPs), picks calendar + notification tools, captures channel + recipient, surfaces the self-notification trap warning if you pick Slack to your own handle, writes `.claude/ideal-week-ops.local.md`. Resumable mid-flow.
2. **`setup` chains into `extract-ideal-week` automatically.** After wiring captures, the flow continues into the extraction interview without a separate trigger phrase. Captures how your week should run (rhythms, deep-work blocks, protected time, VIP overrides, zone of genius), synthesizes a structured ideal-week document, and confirms with you before saving. Say "pause" during the hand-off if you'd rather skip extraction for now — you can run it later with "extract ideal week".
3. **(Optional) Wire a recurring trigger.** If you want twice-daily automatic scans, register a recurring task in your runtime (Cowork Scheduled tasks, Claude Code `/schedule`, GitHub Actions cron, etc.) that invokes `scan-ideal-week`. Recommended cadence: weekdays 5pm (flags tomorrow) + weekdays 7am (flags today). Without this, invoke `scan-ideal-week` manually any time.
4. **First scan.** Run `scan-ideal-week` once manually to confirm the wiring works end-to-end and the output looks right before relying on the schedule.

## Skills included

- **`setup`** — *neutral.* One-time onboarding wizard. Detects whether the host's AI has calendar-read and notification-send capabilities wired (via Composio MCP, direct vendor MCPs, or any other source). Walks the user through wiring if missing. Captures notification channel + recipient. Surfaces the self-notification trap on Slack (warning, not blocker). Writes `.claude/ideal-week-ops.local.md`. Resumable. Re-runnable to refresh any field.
- **`extract-ideal-week`** — *opinionated.* Captures the ideal week. Explains the concept, checks for existing documentation, parses it or runs the standard 10 + 3 question interview (week rhythm + zone of genius), synthesizes a structured ideal-week document, confirms with the user, saves to the configured path. Resumable — if interrupted mid-interview, picks up where it left off. Soft precondition check: warns if `setup` hasn't run, but allows proceeding.
- **`scan-ideal-week`** — *opinionated.* Loads the documented ideal week, pulls calendar events for the target window (default: tomorrow if invoked at or after 4pm, today otherwise) from all configured calendar accounts (merged + de-duplicated), checks every event against every rule, flags violations with severity (block / warning / nudge), generates concrete reschedule suggestions, writes a per-day log file at `log_folder` (always-on output), and optionally sends a notification ping through the configured channel (Gmail / Slack / Outlook). Designed to be invoked by a recurring trigger twice a day, but works on demand any time. Hard fail if `setup` hasn't run.

## Customization notes

Common things clients change:

- **Ideal-week document location.** Default is `client-profile/ideal-week.md`. Override by editing `.claude/ideal-week-ops.local.md` (`ideal_week_path: <your-path>`). Both skills read the same path.
- **Log folder + notification channel.** Both live in `.claude/ideal-week-ops.local.md`. `log_folder` is always-on output (default `client-profile/ideal-week-scans/`). `notification_channel: gmail | slack | outlook | none` chooses whether to also send a live ping; `notification_target` is the recipient (required only when channel ≠ none). The `setup` skill captures all of these interactively. **Heads up:** Slack does not notify when the recipient is the same account that's connected to your wiring — for self-pinging use `gmail` or `outlook`, or set `notification_target` to a different recipient (EA, shared channel). The `setup` skill surfaces this warning automatically.
- **Scan window.** Default: end-of-day scan looks at tomorrow only; morning scan looks at today only. Override via `scan_window_days` in the local config.
- **Rule severities.** Each of the seven rule categories (protected-block, NEVER, cap, PREFER, overlap, buffer, missing-prep) has a default severity (block / warning / nudge) — see [`skills/scan-ideal-week/references/atlas-calendar-enforcement-methodology.md`](skills/scan-ideal-week/references/atlas-calendar-enforcement-methodology.md) for the full taxonomy. Severities can be overridden per-rule in the ideal-week document.
- **Schedule cadence.** Default twice daily (weekdays 5pm + 7am). Adjust in `.claude/ideal-week-ops.local.md` (`scan_schedule:`). Wire the actual recurrence in your runtime's scheduling mechanism — see Suggested tool wiring.
- **Extraction questions.** The 10 + 3 questions in `references/extraction-questions.md` are Atlas's standard set. Edit if a particular client needs additional questions or fewer.

When customizing, edit the `SKILL.md`, reference files, and config in your installed copy or fork.

## Atlas methodology

This plugin encodes Atlas's view that an exec's calendar should be defended by an explicit, documented ruleset — not by anyone's memory. Key principles:

- **Document the week before you defend it.** You can't enforce a rule you haven't written down. The extract skill exists because verbal "deep work on Tuesdays" doesn't survive a real meeting request from a VIP under pressure. Written rules with severities + override lists do.
- **Flag every violation, even small ones.** Nudge-severity flags (preferred-pattern violations) are still surfaced. The exec can decide what to ignore — the plugin's job is to surface, not to filter silently.
- **Suggest concrete fixes, not just complaints.** A flag without a suggestion is noise. Every flag in the scan output names a specific alternative slot.
- **Twice-daily cadence, not real-time.** Two scheduled scans per day beats real-time alerts. Real-time interrupts the exec for things that haven't happened yet; twice-daily gives the user time to fix or escalate before the day starts.

The full extraction framework lives at [`skills/extract-ideal-week/references/atlas-ideal-week-extraction-framework.md`](skills/extract-ideal-week/references/atlas-ideal-week-extraction-framework.md). The enforcement methodology lives at [`skills/scan-ideal-week/references/atlas-calendar-enforcement-methodology.md`](skills/scan-ideal-week/references/atlas-calendar-enforcement-methodology.md).

## Troubleshooting

**`extract-ideal-week` can't find an existing document.** The skill asks the user explicitly whether one exists and where. If the answer is yes, paste the path or contents. If no, the skill runs the interview from scratch — that's the expected fallback, not an error.

**`scan-ideal-week` reports "no ideal week documented".** The scan skill reads from the path configured in `.claude/ideal-week-ops.local.md` (default `client-profile/ideal-week.md`). Run `extract-ideal-week` first, or update the config path if the document lives elsewhere.

**Calendar events aren't being detected.** If using Composio, verify at https://app.composio.dev that Google Calendar (or Outlook) is connected. If using direct MCP, confirm the MCP is authenticated and can list events for the target date range. Also check that the calendar account being read is the one the exec actually schedules into (work vs personal).

**Notifications aren't arriving.** First — check the log file at `log_folder` (default `client-profile/ideal-week-scans/<YYYY-MM-DD>.md`); it's always written regardless of notification status. If the log shows scans ran successfully but the ping isn't arriving: confirm `notification_channel` in `.claude/ideal-week-ops.local.md` matches an app you've connected (in Composio or via direct MCP), and that `notification_target` is the right recipient. Run `scan-ideal-week --dry-run` (or the equivalent flag in your wiring) to print the notification body to terminal — if that looks right, the issue is in the send step.

**The send returns success but the recipient never sees the notification.** This is the self-notification trap. If `notification_channel` is `slack` AND `notification_target` is the same account connected to your Slack wiring, the send succeeds silently — Slack does not push self-DMs. Two fixes: (a) change `notification_target` in `.claude/ideal-week-ops.local.md` to a different recipient (the EA's handle, a shared channel both you and the EA are in); or (b) switch `notification_channel` to `gmail` or `outlook` — both deliver to inbox + Sent with normal notifications even when the recipient and the connected account are the same.

**Scheduled scan didn't run.** Check that the recurring trigger was actually registered in your runtime — Cowork's Scheduled tasks list, Claude Code's scheduled-agents list, `crontab -l`, or wherever applicable. If scans are silently missing, check your runtime's scheduling logs or switch to a server-side scheduler (Cowork, scheduled agents, GitHub Actions).

**The flagged violations don't match what the exec actually cares about.** The rule severities and the rules themselves live in the ideal-week document — edit it directly. The scan skill is deterministic against whatever the document says, so adjust the source of truth, not the scan logic.
