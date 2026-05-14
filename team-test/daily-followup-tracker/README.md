# Daily Follow-Up Tracker

## 1. What it does

Sweeps the last 24 hours across email, messaging, today's meetings, and your task system. Detects open loops you owe (waiting on your reply) and today's-new commitments you just made. When an open loop on one channel is addressed by a commitment on another (a Slack question answered in a meeting), it marks the loop resolved — no redundant draft. Drafts every follow-up in your actual per-channel voice (email voice and Slack voice are different) and produces a single daily report file with drafts ready to copy-send. Optionally pings you when the report is ready.

## 2. Who it's for

Executives, founders, and operators whose follow-ups live across surfaces — email, Slack, calls, task boards. Specifically valuable for people who:
- Make commitments throughout the day in different channels and lose track
- Get pulled into back-to-back meetings and don't have time to walk the inbox at end of day
- Have an EA / Chief of Staff who runs the daily sweep on their behalf
- Already use Atlas's `proactive-actions` for post-meeting batches and want a daily multi-channel rollup to complement it

## 3. Required capabilities

(Capabilities are abstract — the skill works with any tool that satisfies them. See §4 for the validated default wiring.)

- **Email read** — search messages by sender / recipient / date / label state; fetch thread content. (Either Gmail or Outlook satisfies this.)
- **Messaging read** — search messages user sent in a time window; scope to specific channels. (Slack via OAuth satisfies this.)
- **Meeting transcript read** — list meetings in a date range; fetch transcript by ID OR list files in a transcript-storage location and read their content. (Fathom / Granola / Fireflies for direct integration; Notion DB or Drive folder for fallback.)
- **Task system read** — discover containers; introspect their schema; filter for assignee + status + date. (Notion, Asana, Linear, ClickUp, Todoist, Trello, or Monday all satisfy this — the skill discovers each tool's shape at runtime and asks the user about specifics where the schema varies.)
- **Notification send** — send a short message via the configured channel. (Gmail / Outlook for email; Slackbot for Slack — the user's Slack OAuth does NOT work as a notification target because messages-to-self don't ping.)
- **Toolkit introspection** — discover wired capabilities and ask the user about specifics where the schema varies.
- **File read + write** — for the wiring config, voice profile, in-progress marker, and report output.
- **Conversational interview** — for setup, one question at a time.
- **Sub-agent dispatch** — host runs sub-agents in parallel where supported; sequential fallback otherwise.

At least one of {email, messaging, meetings, tasks} must be wired. Notification can be `none`. Voice profile capture requires at least one of {email, messaging}.

## 4. Suggested tool wiring

Validated default wiring is Composio MCP — works in Claude Code, Codex, ChatGPT, and any host that loads MCP tools.

| Capability | Composio MCP toolkit | Notes |
|---|---|---|
| Email read | `gmail` (or `outlook`) | Both validated; Gmail probed end-to-end |
| Messaging read | `slack` (user OAuth — read source only) | NOT a notification target |
| Meeting transcript read (direct) | `fathom` / `granola` / `fireflies` | Pick whichever you use |
| Meeting transcript read (folder fallback) | `notion` / `googledrive` | When transcripts land in a DB or Drive folder |
| Task system read | `notion` / `asana` / `linear` / `clickup` / `todoist` / `trello` / `monday` | Whichever you use; the setup wizard discovers your tool's schema and walks you through the specifics |
| Notification send (email) | `gmail` / `outlook` | |
| Notification send (Slack) | `slackbot` (Slack Bot integration) | DISTINCT from `slack` toolkit; user OAuth posts as the user → no notification ping |

Alternatives that satisfy the capabilities (not validated this round): direct vendor MCPs (Pattern B), Microsoft Teams' equivalent for messaging.

## 5. Installation

Install via the Atlas marketplace:

```
/plugin marketplace add colin-atlas/atlas-skills-library
/plugin install daily-followup-tracker@atlas
```

## 6. First-run setup

Run the bundled `setup` skill before the first sweep:

> "set up daily followup tracker"

The wizard walks through:
1. **Capability detection** — which connected tools satisfy the required capabilities. If any are missing, walks through Composio install for that one tool.
2. **Source picker** — multi-select among email, messaging, meeting transcripts, task system. At least one required.
3. **Per-source detail** — accounts, channels, transcript location, task containers + assignment field (the wizard discovers each tool's schema and asks you about specifics where it varies by provider).
4. **User identity** — silently captured from connected accounts; confirmed once.
5. **Voice profile capture** — reads your last ~20 sent emails + recent Slack messages. Writes per-channel voice patterns to `<workspace>/client-profile/daily-followup-tracker-voice.md`.
6. **Meeting follow-up default channel** — one-time choice: email or Slack.
7. **Notification channel** — gmail / outlook / slackbot / none.
8. **Output folder** — default `followups/` in the workspace.

Saves to `<workspace>/client-profile/daily-followup-tracker.local.md`. Resumable mid-flow.

To wire the daily recurring trigger: the sweep's cadence is described in `daily-followup-tracker`'s skill body; recurrence registration lives in whichever scheduler your AI host provides (Claude Code `/schedule`, GitHub Actions cron, OS cron, or any other scheduler the host exposes). The setup hand-off message walks through the available options.

## 7. Skills included

- **`setup`** — onboarding wizard. Three branches (refresh / fast / full onboarding). Detection-first: figures out what the user already has connected and only walks them through what's missing. Resumable mid-flow. Re-runnable. Atlas methodology: neutral.
- **`daily-followup-tracker`** — the daily sweep. Orchestrates parallel sub-agent fan-out across configured sources. Cross-source loop closure. Voice-profile-styled drafts. Atlas methodology: opinionated.

## 8. Customization notes

- **What counts as a commitment** — edit `skills/daily-followup-tracker/references/atlas-followup-detection-methodology.md`. Tune phrase patterns and confidence cutoffs.
- **Per-source query patterns** — edit `skills/daily-followup-tracker/references/per-source-detection-guidance.md`. Useful if your inbox-zero workflow already labels "needs reply" threads.
- **Draft style** — edit `skills/daily-followup-tracker/references/draft-style-guidance.md`. Adjust how voice profile patterns apply.
- **Wizard wording** — edit `skills/setup/references/canonical-messages.md`.
- **Task knobs** — `task_due_within_days` (default 2) and `task_stale_after_days` (default 5) in `<workspace>/client-profile/daily-followup-tracker.local.md`. Tune by editing.

Voice profile patterns are captured into `<workspace>/client-profile/daily-followup-tracker-voice.md`. Edit by hand if the auto-extraction misses something specific.

## 9. Atlas methodology

Atlas has an opinion about:
- *What counts as a commitment* — verb + recipient + (optional) deadline; hedge words downgrade confidence.
- *What confidence threshold flips a finding from "draft" to "low-confidence flag"* — tuned for *doesn't burn trust*, not *maximize recall*.
- *What the daily report should feel like* — five sections in a fixed order, voice-profile-styled drafts, no padding.
- *How cross-source loops close* — the orchestrator does a pre-report pass that matches open loops on one channel against today's commitments on another. This is what makes the report feel smart.

The opinion lives in the references. Clients customize by editing the references; the skill body stays stable across forks.

## 10. Troubleshooting

**No findings detected, but I know I made commitments today.**
- Check that the sources you expected to be swept are all listed in the report's *Sources skipped / errors* section. If they're listed, the connector is unreachable — re-run setup in refresh mode to re-detect.
- Check that `user_identity.primary_email` (in `<workspace>/client-profile/daily-followup-tracker.local.md`) matches the sender address on your recent emails. If you sent from a different account, the today's-new email scan filtered them out.

**Drafts sound generic / robotic.**
- Check the report header for a *"Voice profile is missing"* banner. If present, re-run setup and pick "yes, re-capture voice" at Step 6.
- If the voice profile exists but drafts still feel off, hand-edit `<workspace>/client-profile/daily-followup-tracker-voice.md` to add specifics the auto-extraction missed (e.g., signature template, pet phrases).

**Slack notification didn't arrive.**
- Confirm `notification_channel: slackbot` (NOT `slack`) — the user's Slack OAuth posts as the user, which doesn't ping.
- Confirm the Slackbot integration has post permissions in the target channel/DM.
- Check the report's *Sources skipped / errors* section — notification failures are logged there.
