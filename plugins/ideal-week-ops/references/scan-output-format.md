# Scan Output Format

The shape of the summary the `scan-ideal-week` skill produces. The same structure renders in every channel — Slack, email, file. The implementation layer adapts spacing and emphasis per channel; the content shape is fixed.

## Top-level structure

```
[Header]
[Day section per day in window]
[Footer]
```

## Header

One line, always present:

```
Ideal-week scan — <window-label> — <flag-count-summary> — <timestamp>
```

- `<window-label>` — e.g., "tomorrow (Mon Apr 27)" or "today (Sun Apr 26)" or "next 3 days (Apr 27 – Apr 29)"
- `<flag-count-summary>` — e.g., "3 blocks, 2 warnings, 1 nudge" or "all clean ✅"
- `<timestamp>` — local time, e.g., "scanned 2026-04-26 17:02"

## Day section

One per day in the window. Order by date ascending. If a day is clean, include it as a single line: `<day-label>: clean ✅`. Otherwise:

```
## <day-label>

🚫 BLOCKS
- <event title> · <time> · <rule violated>
  → <suggestion>
- ...

⚠️ WARNINGS
- <event title> · <time> · <rule violated>
  → <suggestion>
- ...

💡 NUDGES
- <event title> · <time> · <rule violated>
  → <suggestion>
- ...
```

Within each severity bucket, order events by start time ascending. If a bucket is empty, omit the bucket entirely (no empty headers).

## Flag entry format

Each flag is two lines:

1. **Identifier line** — `<event title>` · `<time range>` · `<rule violated>`
2. **Suggestion line** — `→ <concrete action>`

If a VIP override was applied, append a third line:
3. `(VIP override applied: <name> — severity downgraded from <original>)`

If the flag carries zone-of-genius context (event consumes drain time):
4. `(zone-of-genius: this falls in your drain category — consider delegating to <person>)`

## Footer

Two-line block, always present:

```
[Scan log: <log-file-path>]
[<dry-run notice if applicable>]
```

- The scan log path is the location of the JSONL log entry for this run.
- The dry-run notice appears only in dry-run mode: `DRY RUN — no notification was sent.`

## Worked example — output for one flagged day

```
Ideal-week scan — tomorrow (Mon Apr 27) — 2 blocks, 1 warning — scanned 2026-04-26 17:02

## Mon Apr 27

🚫 BLOCKS
- Vendor pitch · 7:30-8:00am · NEVER rule: no meetings before 9am
  → Decline; ask vendor to schedule Mon/Thu/Fri after 9am
- Surprise board prep call · 12:00-12:45pm · Protected block: lunch window
  → Move to 2:45-3:30pm (currently free in strategic block)

⚠️ WARNINGS
- Marketing weekly · 10:00-10:30am · No prep block in front of 30-min meeting
  → Add 15-min prep at 9:45am (currently free)

[Scan log: client-profile/ideal-week-ops.scan-log.jsonl]
```

## Worked example — clean window

```
Ideal-week scan — tomorrow (Mon Apr 27) — all clean ✅ — scanned 2026-04-26 17:02

## Mon Apr 27: clean ✅

[Scan log: client-profile/ideal-week-ops.scan-log.jsonl]
```

## Channel-specific rendering notes

The MCP notification tool adapts the above for each channel:

- **Slack** — emoji severity markers render natively; use Slack mrkdwn for `*bold*` only if you want emphasis; the canonical content shape (emoji headers, `## Day`, two-line flag entries) already works in plain text. Collapse very long flag lists into a thread reply with header in the main message.
- **Email (Gmail / Outlook)** — send as **plain text**. The canonical format above already has visual structure (emoji severity headers, `## ` day headers, `· ` separators, `→` indented suggestions, blank lines between sections) that renders identically in every email client. Do NOT convert to HTML — most send-mail tools (e.g., `GMAIL_SEND_EMAIL` via Composio) default to plain text, and HTML tags would render as literal text in the recipient's inbox.
- **File** — write the markdown-style canonical format as-is. Read-then-append safety per `scan-ideal-week`'s "Log file safety" section.

The content (headers, severity ordering, suggestion line, VIP annotation) is identical across channels. Only the rendering changes.

## Length guidance

A clean day is one line. A typical flagged day is 4-8 lines. A pathological day (10+ flags) should still be sent as one notification — never split. Long notifications are better than missed notifications. The reader scans visually for severity emoji and reads details only on the flags they care about.

## What the format does NOT include

These are intentional omissions — keep them out:

- **Statistics across runs.** "You had 12 blocks this week" belongs in a separate weekly review, not in the daily scan.
- **Past-day flags.** The scan looks forward only. Yesterday's flagged events don't reappear.
- **Aspirational suggestions.** "Maybe you should consider canceling Thursday 1on1s" is out of scope. The scan suggests fixes for specific events; it doesn't recommend ideal-week changes. Those go through `extract-ideal-week` re-runs.
- **Quiet hours.** The scan does not delay or silence notifications based on time-of-day. The schedule (when the scan runs) controls that.
