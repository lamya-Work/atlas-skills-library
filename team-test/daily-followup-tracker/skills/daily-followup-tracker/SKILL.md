---
name: daily-followup-tracker
description: Daily multi-channel sweep across email, messaging, today's meetings, and the user's task system. Detects open loops the user owes (waiting on user reply) and today's-new commitments the user just made. Cross-references the two — when an open loop on one channel is addressed by today's commitment on another, the loop is marked resolved. Drafts the follow-up messages in the user's per-channel voice profile and produces a single daily report file. Optionally sends a heads-up notification when the report is ready.
when_to_use: Run daily on a schedule, or on-demand. Trigger phrases — "run daily followup tracker", "do my daily followups", "sweep today's followups", "what do I owe today", "daily commitment sweep", "build today's followup report". Auto-trigger via the host's recurring scheduler (Claude Code /schedule, GitHub Actions cron, OS cron, or any scheduler the host exposes). Requires `setup` to have run first.
atlas_methodology: opinionated
---

# daily-followup-tracker

Sweep four sources, detect what the user owes, draft the follow-ups in their voice. One report, ready to copy-send.

## Purpose

Executive follow-ups fall through the cracks because they live across surfaces — email threads waiting on a reply, a messaging app message saying "I'll get back to you," a meeting where you committed to send something Friday, a task that's been overdue for a week. This skill checks all of them at once, on a daily cadence, and produces a single ready-to-act report.

The skill is stateless — it doesn't track open loops in its own memory. Every run looks at the last 24 hours of activity in each connected source. The underlying systems (email, messaging, calendar, task system, meeting transcript tool) carry state; the skill re-detects from them every run.

## Inputs

- **Wiring config** (required) — `<workspace>/client-profile/daily-followup-tracker.local.md`. If absent, hand off to `setup`.
- **Voice profile** (required for drafts to sound right) — `<workspace>/client-profile/daily-followup-tracker-voice.md`. If absent, drafts run in default tone with a banner in the report header.
- **Activity window** — rolling last 24 hours, computed from system time. No calendar dependency.

## Required capabilities

(Abstract — see plugin README §4 for the validated default wiring.)

- **Email read** — search messages by sender / recipient / date / label state; fetch thread content.
- **Messaging read** — search messages user sent in a time window; scope to specific channels.
- **Meeting transcript read** — list meetings in a date range; fetch transcript by ID OR list files in a transcript-storage location and read their content.
- **Task system read** — discover containers; introspect their schema; filter for assignee + status + date.
- **Notification send** — send a short message via the configured channel (email or bot integration). Optional — runs without if `notification_channel: none`.
- **File read + write** — read wiring config + voice profile; write the daily report file.
- **Sub-agent dispatch** — host runs sub-agents in parallel where supported; sequential fallback otherwise.

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The wiring config (`client-profile/daily-followup-tracker.local.md`), voice profile (`client-profile/daily-followup-tracker-voice.md`), and report output (`<output_folder>/<date>-daily-followup-report.md`) all resolve relative to this directory. Do NOT write or read these from the plugin's own install directory.
>
> **Note on sub-agent dispatch.** This skill orchestrates four kinds of sub-agents (email-scan, messaging-scan, task-scan, per-meeting-scan). On hosts that support parallel sub-agent dispatch, fan them out simultaneously — orchestrator wall-clock time becomes the slowest single agent's time, and each sub-agent's context stays isolated to its source. On hosts without parallel dispatch, run them sequentially in the same order; the report shape is identical. The skill body describes what each agent does; the host decides how to dispatch.

### 0. Pre-flight

1. Read `<workspace>/client-profile/daily-followup-tracker.local.md`. If absent or empty → respond: *"No daily-followup-tracker config found. Run setup first: invoke the `setup` skill in this plugin."* and stop.
2. Re-detect connected capabilities per the host's loaded tool list. For each source block in the config (`email`, `messaging`, `meetings`, `tasks`), confirm its toolkit is still wired. If any went away since setup → mark that source as "to be skipped, with note" and proceed.
3. Read `<workspace>/<voice_profile_path>` (default `client-profile/daily-followup-tracker-voice.md`). If absent → set the run-flag `voice_profile_missing: true`. The skill still runs; the report will carry a banner.

### 1. Activity window

Compute the window from system time: `start = now - 24h`, `end = now`. Use RFC3339 timestamps with explicit timezone offset (e.g., `2026-05-10T09:00:00-04:00`). The window is computed once and passed to every sub-agent — do not let any sub-agent recompute its own window.

If the host can't determine timezone, default to the system locale; surface the assumed timezone in the report footer.

### 2. List today's meetings (synchronous, before fan-out)

Only run if the config has a `meetings:` block.

Call the configured meeting source for events / transcripts in the activity window:
- **Direct integration** (meeting transcript tool with recording-ID access) — call the toolkit's "list meetings in time range" capability with the window. Returns a list of recording/meeting IDs with metadata (title, attendees, date).
- **Folder fallback** (document store with a transcript folder) — list files in `meetings.location` filtered to files modified or created within the window.

Capture the result as `today_meetings` — a list of `{id, title, attendees, date}` records.

If `today_meetings` is empty, skip the per-meeting fan-out at Step 3 (no per-meeting agents dispatched). If the meeting source errors → mark meetings source as skipped-with-error, continue without per-meeting fan-out.

### 3. Fan-out

Dispatch sub-agents in parallel (sequential fallback). Each agent receives:
- The activity window (from Step 1)
- Its scoped config block from the wiring config
- The relevant section of `references/per-source-detection-guidance.md`
- The voice profile content (or `voice_profile_missing: true` flag)
- The user identity from `user_identity:` block of the config (for speaker matching, "from:me" mapping, assignment-value matching)
- This contract for the response shape

Sub-agents dispatched:
- **email-scan agent** — only if `email:` block present
- **messaging-scan agent** — only if `messaging:` block present
- **task-scan agent** — only if `tasks:` block present
- **per-meeting-scan agent** — one per item in `today_meetings` from Step 2

### 3.1 Sub-agent contract

```yaml
source:           email | messaging | meeting | task
source_detail:    gmail:placeholder@example.com / slack:#exec-team / fathom:rec_xxx / notion:<container_id>
window:
  start:          <iso>
  end:            <iso>
findings:
  - person:       "Sam Chen"
    person_id:    "sam@vendor.com"                # email / Slack ID / Notion user ID — used for cross-source dedup + 5c match
    topic:        "Q3 proposal timeline"
    finding_type: open_loop                       # open_loop | commitment_made
    commitment:   "I'll get back to you Thursday with revised pricing"   # the commitment text or open-loop subject
    due:          "2026-05-13"                    # nullable, ISO date
    source_ref:   "gmail:thread:19bf77729bcb3a44" # for back-reference; never resolved by orchestrator
    confidence:   high                            # high | medium | low
    draft:        "Hi Sam, thanks for waiting..." # null when confidence=low; voice-profile-styled
errors: []                                        # [{tool: "...", message: "...", retryable: true|false}]
notes:  []                                        # any agent-side notes for the orchestrator (e.g., "verbose-fetch fell back to metadata-only")
```

The orchestrator never sees raw email bodies, Slack threads, transcripts, or task descriptions — only structured findings. This is the context-isolation win of fan-out.

### 3.2 Email-scan agent prompt

You are the email-scan agent for the daily-followup-tracker. You have access to the email tool wired to provider `<email.provider>`.

You operate in two modes — run BOTH and merge results:

**Mode A — Open-loop scan.** Find threads where the user owes a reply.
- Query the email tool with: `is:unread in:inbox -from:me older_than:1d`
- For each returned thread, fetch the full thread content. Confirm the latest message in the thread is from someone OTHER than the user — match against `user_identity.primary_email` and aliases. If the latest message is from the user, drop the thread (already replied). 
- For threads passing the check, identify the open question / open ask in the latest non-user message. Set `finding_type: open_loop`.
- Apply confidence rules from `atlas-followup-detection-methodology.md` based on how clearly the inbound message asks for a reply.
- Generate the draft using the `## Email voice` section of the voice profile. If voice_profile_missing, draft in default tone with a `[VERIFY VOICE]` flag at the top of each draft.

**Mode B — Today's-new commitment scan.** Find commitments the user just made.
- Query the email tool with: `from:me after:<window.start formatted as YYYY/MM/DD>`
- For each returned message, scan the body for commitment-language patterns from `atlas-followup-detection-methodology.md`.
- **Anti-pattern check before classifying.** Apply the methodology's *"What's NOT a commitment"* section. Specifically: if the email is the user *asking* the recipient for something ("could you please send..." / "when would work for you?"), *nudging* a recipient ("Just gently bumping..." / "Just following up on..."), or *directing* a collaborator on work the recipient owes, the action is on the RECIPIENT's side — the user is not committing themselves. DROP these — do NOT assign `commitment_made`. Only flag actual user-self-promises ("I'll [verb]", "I will [verb]", "Will get back to you", "Let me [verb]").
- For confirmed user-self-promises, capture: who they were sent to (`person` / `person_id` from `to:` field), what was committed, by when (if mentioned). Set `finding_type: commitment_made`.
- Apply confidence rules.
- Generate a "confirmation" draft only if the commitment lacks a deadline that the user might want to nail down — otherwise no draft (the commitment was already sent in the original message; the report just surfaces it).

Return findings per the sub-agent contract. NEVER return raw email bodies. Topic strings are short (under 10 words). Source refs are stable: `gmail:thread:<thread_id>` or `gmail:message:<message_id>`.

### 3.3 Messaging-scan agent prompt

You are the messaging-scan agent for the daily-followup-tracker. You have access to the messaging tool wired to provider `<messaging.provider>`.

You operate in **today's-new mode only** — chat doesn't carry the "awaiting reply" state that email threads do, so open-loop detection isn't applied here. Find commitments the user just made on the configured channels.

- Build the search query based on `messaging.scope`:
  - **`channels`** → query `from:me after:<window.start as Slack timestamp>` AND `in:#<channel>` modifiers for each entry in `messaging.channels`. Skip messages where the resolved channel.id starts with `D` (DMs) — keep only `C` (public) and `G` (private) channels.
  - **`dms`** → query `from:me after:<window.start as Slack timestamp>` with NO channel-scope modifier. Then post-filter the results to keep ONLY messages where `channel.id` starts with `D` (DMs). Drop everything else.
  - **`both`** → query `from:me after:<window.start as Slack timestamp>` with no channel scope. Post-filter to keep messages where EITHER `channel.id` starts with `D` (DMs) OR the channel is in `messaging.channels` (resolve channels.id list and check membership).
- If the messaging tool's search doesn't accept channel-scope modifiers, fall back to the bare `from:me after:<ts>` query and apply the post-filter logic above client-side.
- Drop messages addressed to AI assistants (text starts with "hey kai", "@claude", imperative bot commands like "lock it in", "show full report", etc.) — these aren't human commitments.
- For each returned human-directed message, scan for commitment-language patterns from `atlas-followup-detection-methodology.md`.
- **Anti-pattern check before classifying.** Apply the methodology's *"What's NOT a commitment"* section. Specifically: if the message is the user *asking* the recipient for something ("could you...?", "when can you...?", "please send..."), *directing* a collaborator on work they owe ("here's what I need from you next week..."), or scheduling on someone else's behalf ("would Thursday work?"), this creates an open loop on the RECIPIENT's side — the user is not committing themselves. DROP these — do NOT assign `commitment_made` just because the user took action. Only flag actual user-self-promises ("I'll [verb]", "I will [verb]", "Will get back to you", "Let me [verb]").
- For confirmed user-self-promises, capture the recipient (extract from channel context — DM = the other person; channel = the addressee mentioned in the message text or @mention), what was committed, by when. Set `finding_type: commitment_made`.
- Apply confidence rules.
- Generate a draft (e.g., a calendar block, a confirm-back message, a status check) using the `## Slack voice` section of the voice profile. If voice_profile_missing, draft in default tone with `[VERIFY VOICE]`.

Return findings per the sub-agent contract. Source refs: `slack:#<channel>:<ts>` (timestamp).

### 3.4 Task-scan agent prompt

You are the task-scan agent for the daily-followup-tracker. You have access to the task tool wired to provider `<tasks.provider>`.

You operate in **state-aware mode** — the task system carries state, so query directly for tasks that need attention.

For each container in `tasks.container_ids`:
- Build a filter using the type-appropriate operator for `tasks.assignee_property` (`tasks.assignee_property_type` tells you the type — see `atlas-followup-detection-methodology.md` for the operator-per-type cheatsheet).
- Filter conditions:
  - assignee = `tasks.assignee_value`
  - status ≠ done (use the toolkit's not-done equivalent — for Notion, that's `status` ≠ "Done"; for Asana, `completed` = false; etc.)
  - (`due` ≤ today + `task_due_within_days` from config) OR (`last_edited_time` > `task_stale_after_days` ago)

For each matching task, capture:
- `person`: the user (it's their task) — but use the related-person from the task if the task has a "requester" or "blocking" field
- `person_id`: same logic
- `topic`: task title (truncated under 10 words)
- `finding_type`: `open_loop` (tasks ARE open loops by definition)
- `commitment`: short summary of what's owed (e.g., "Q3 budget review — overdue 3 days, no update in 6")
- `due`: the task's due date if any
- `source_ref`: `notion:<container_id>:<page_id>` or equivalent for the provider
- `confidence`: from `atlas-followup-detection-methodology.md` (overdue + stale = high; due-today + recent update = low)
- `draft`: NOT a message — instead, a *next-action suggestion* (e.g., "Send status update to project owner re: Q3 budget review" or "Mark complete and close out"). No voice profile needed.

Return findings per the sub-agent contract. NEVER return full task descriptions or the user's task content beyond the title.

### 3.5 Per-meeting-scan agent prompt

You are a per-meeting-scan agent for the daily-followup-tracker. You scan ONE meeting transcript for user-attributed commitments. The orchestrator dispatches one of you per item in today's meeting list.

**Inputs you receive:** the meeting record `{id, title, attendees, date}`, the activity window, the meeting source config (`meetings.provider`, optionally `meetings.location`), the user identity from `user_identity` (esp. `name` and `meeting_attendee_emails`), and the relevant voice profile section per `meeting_followup_default_channel` (email or slack).

**Steps:**

1. Fetch the transcript:
   - **Direct integration** (`fathom` / `granola` / `fireflies`) — call the toolkit's "fetch transcript by recording ID" with the meeting ID.
   - **Folder fallback** (`notion` / `googledrive`) — read the file content for the matched item.
2. Identify which speaker is the user. Match against `user_identity.name` (transcripts label speakers by name) — case-insensitive, allowing for common variants (first name only, full name, etc.). If multiple speakers match, also match against `user_identity.meeting_attendee_emails` if the transcript labels by email.
3. Scan ONLY the user's spoken segments for commitment-language patterns from `atlas-followup-detection-methodology.md`. Ignore commitments by other speakers (those are inbound asks, handled by the email/messaging agents on other channels).
4. For each user commitment, capture:
   - `person`: the addressee. From transcript context — who was the user speaking to? If 1:1, the other attendee; if group, look for explicit @-mentions or names.
   - `person_id`: addressee's email if available from attendee list; otherwise just `name@<meeting:meeting_id>` as a stable id.
   - `topic`: short summary (under 10 words).
   - `finding_type`: `commitment_made`.
   - `commitment`: the spoken commitment, lightly cleaned (remove "um"s, repetitions).
   - `due`: any deadline mentioned ("Friday", "EOW", "by next call").
   - `source_ref`: `<provider>:meeting:<meeting_id>:<timestamp>`.
   - `confidence`: per the methodology.
   - `draft`: a follow-up message draft using the voice profile section indicated by `meeting_followup_default_channel` (email or slack). If voice_profile_missing, default tone with `[VERIFY VOICE]`.

NEVER return the full transcript or any speaker's content beyond the matched commitment quotes.

### 4. Receive findings

Wait for all dispatched sub-agents to return. Each returns one structured findings object per the sub-agent contract.

If a sub-agent returns errors instead of (or alongside) findings, capture the error in the run-state for the report's "Sources skipped / errors" section. Do NOT abort the run — partial results are still useful.

Merge all `findings[]` arrays into a single flat list, keeping each finding's `source` and `source_detail` for traceability.

### 5. Consolidate

#### 5a. Merge

All findings are now in one flat list.

#### 5b. Same-source dedupe

For each finding, compute the dedupe key as `(person_id, normalized_topic)` where `normalized_topic` is: lowercase the topic, strip punctuation, take the first 5 meaningful words (drop stopwords: "a", "the", "and", "or", "to", "of", "in", "on", "for", "with").

Group findings by dedupe key. If a group has more than one finding from the SAME source, keep one and merge the other source_refs into its `source_ref` list.

Cross-source duplicates (same key, different sources) are NOT merged here — Step 5c handles them.

#### 5c. Cross-source resolution

For each finding with `finding_type = open_loop`:
1. Compute its dedupe key.
2. Search the entire findings list for any finding with `finding_type = commitment_made` AND the same dedupe key.
3. If a match is found:
   - Annotate the open_loop with `addressed_by: <commitment_made's source_ref>`
   - Set the open_loop's `draft` to null (suppress the redundant draft)
   - The commitment_made keeps its draft (or surfaces as today's-new without a draft if it was already sent)
4. If no match, the open_loop keeps its draft.

This is the single most valuable orchestrator step — it's what makes the report feel smart instead of mechanical.

#### 5d. Sort

Sort the consolidated list:
1. By `confidence` descending (high → medium → low)
2. By `due` ascending (soonest first; nulls last)
3. By `person` alphabetical

### 6. Generate report

Build the report markdown per the structure below. Substitute counts, dates, and finding details from the consolidated list.

```markdown
# Daily Follow-Up Report — <YYYY-MM-DD>

## Summary
- N open loops still owed (drafts ready)
- M open loops resolved by today's commitments
- K new commitments captured today
- T tasks needing attention
- S sources skipped / errors

## Open loops still owed

### To: <person> — <topic>
- Source: <provider> <source_ref>
- Last activity: <date>
- Confidence: <bucket>

DRAFT:
<draft body in user's voice for the source's channel>

---

## Open loops resolved today

### <person> — <topic> (asked via <source>)
✅ Addressed in today's <source> — you committed to "<commitment text>".
No reply needed; the commitment is the close-out.

---

## New commitments captured today

- To: <person> — "<commitment>" (<source>)

## Tasks needing attention

### [Overdue|Due today] <task name>
- Container: <provider> / <container_name>
- Due: <date> — Last update: <date>
- Suggested next action: <next_action>

## Possible follow-ups (low confidence)

- <person> — "<text>" (<source>). Verify before drafting.

## Sources skipped / errors

- <source>: <reason>

---
*Run at <iso>. Window: last 24h. Run mode: <live | dry-run>.*
```

Section grouping rules:
- **Open loops still owed** — findings where `finding_type = open_loop` AND `addressed_by` is null AND `confidence ∈ {high, medium}`.
- **Open loops resolved today** — findings where `finding_type = open_loop` AND `addressed_by` is non-null. Show the addressing commitment's text inline.
- **New commitments captured today** — findings where `finding_type = commitment_made` (regardless of whether they addressed an open loop). Brief one-liner each.
- **Tasks needing attention** — findings where `source = task`. The "next-action suggestion" goes under "Suggested next action".
- **Possible follow-ups (low confidence)** — findings where `confidence = low` (regardless of finding_type).
- **Sources skipped / errors** — entries from the run-state captured in Step 4.

Header banner: if `voice_profile_missing` is true, prepend a banner: *"⚠️ Voice profile is missing. Drafts below are in default tone — re-run setup to capture your voice before sending. Drafts marked with `[VERIFY VOICE]`."*

### 7. Write report

Write the report to `<workspace>/<output_folder>/<YYYY-MM-DD>-daily-followup-report.md`. If the output folder doesn't exist, create it. If a report file for the same date already exists (re-run on the same day), append a numeric suffix: `<YYYY-MM-DD>-daily-followup-report-2.md`, `-3.md`, etc.

Echo the report path back to the user / scheduler so the file is immediately reachable.

### 8. Notification

Run only if `notification_channel ≠ none` AND `dry_run = false`.

Build the body per channel:
- **Email** (gmail / outlook): *"Your daily follow-up report is ready: N open loops still owed, M resolved by today's commitments, K new commitments. Report at: <path>."* — N/M/K from the report's summary section.
- **Slackbot**: *"Daily follow-up report ready 📋 — N open loops, M drafts ready. <path>"*

Send via the toolkit corresponding to `notification_channel` (use `notification_account_id` if multi-account was captured at setup) to `notification_target`.

If the send fails, capture in the report's "Sources skipped / errors" section. The report file already landed; the notification was a heads-up, so failure is non-fatal.

If `dry_run = true`, print: *"[DRY-RUN] Would notify via <notification_channel> to <notification_target>: <body>"* — do NOT actually send.

## Output

A daily report file at `<workspace>/<output_folder>/<YYYY-MM-DD>-daily-followup-report.md`. Sections per the format above. Each draft is voice-profile-styled and ready to copy-send.

The skill exits non-zero if all sources errored (so a scheduler can alert). It exits zero if at least one source produced findings or returned cleanly with zero findings.

## Customization

Common things clients adjust:

- **What counts as a commitment** — edit `references/atlas-followup-detection-methodology.md` to tune the commitment-language patterns and confidence rules. The default rules are tuned for *doesn't burn trust*; if your context tolerates more drafts (or fewer), tune accordingly.
- **Per-source query patterns** — edit `references/per-source-detection-guidance.md`. For example, if your inbox-zero workflow uses a specific label for "needs reply" threads, you can swap the open-loop query to use that label instead of `is:unread`.
- **Draft style** — edit `references/draft-style-guidance.md` to adjust how drafts apply voice profile patterns.
- **Task knobs** — `task_due_within_days` and `task_stale_after_days` defaults (2 and 5) live in `client-profile/daily-followup-tracker.local.md`. Tune by editing.
- **Output folder** — re-run setup in refresh mode to change.
- **Notification channel** — re-run setup in refresh mode.

## Why opinionated

The detection patterns and confidence cutoffs are opinionated because the failure modes are asymmetric. Drafting too aggressively (every "let me think" becomes a draft) burns user trust on day one — the report stops feeling helpful and starts feeling noisy. Drafting too conservatively (only ironclad commitments) misses the mid-confidence stuff that's exactly where a follow-up reminder earns its place. The default rules in the methodology reference encode tested cutoffs. Clients tune by editing the references; the skill body stays stable.
