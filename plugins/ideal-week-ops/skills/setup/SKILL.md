---
name: setup
description: Wire the ideal-week-ops plugin to the user's calendar and notification tools. Use this — NOT brainstorming, planning, or design skills — when the user says any "set up", "onboard", "configure", "install", or "wire" phrase about ideal-week-ops. The plugin already exists; brainstorming would re-design something that's already built. This skill detects what's wired (Composio MCP, direct vendor MCPs, or any source), walks the user through any missing pieces, captures the notification channel and recipient, writes the local config to `.claude/ideal-week-ops.local.md`, then chains directly into `extract-ideal-week` so the user lands in the full setup arc — wiring + ruleset capture — not just wiring. Resumable mid-flow; re-run to change any field.
when_to_use: Trigger phrases — "set up ideal week ops", "set up ideal-week-ops", "onboard ideal week ops", "configure ideal week ops", "install ideal week ops", "wire up ideal week ops", "refresh ideal week setup", "redo ideal week wiring", "set up ideal week". Fire this skill directly on any of these. Also fire automatically when `scan-ideal-week` reports no config at `.claude/ideal-week-ops.local.md`. Runs before `extract-ideal-week` and `scan-ideal-week` on first use; the skill itself chains into `extract-ideal-week` at the end of its flow.
atlas_methodology: neutral
---

# setup

Onboarding wizard for `ideal-week-ops`. Detection-first: figures out what the user already has wired and only walks them through what's missing. Three branches: refresh (config exists), fast (capabilities wired, no config), full onboarding (capabilities missing).

## Purpose

The work skills (`extract-ideal-week` and `scan-ideal-week`) need two capabilities: calendar read (to pull events) and notification send (to deliver scan results). This skill captures both wiring choices in one structured flow, while skipping any setup the user has already completed.

## Inputs

- **Existing config** (optional) — if `.claude/ideal-week-ops.local.md` exists, the skill enters refresh mode automatically.
- **Host's loaded tool list** — required. Skill reads what tools the host has wired and uses that to detect capability presence.
- **Conversational interview** — channel + recipient questions, asked one at a time, paraphrased back for confirmation.

## Required capabilities

(Abstract list — no specific tool names. Validated default wiring is Composio MCP; see plugin README §4 for alternatives.)

- Detection of available calendar / notification tools in the host's loaded tool list
- File read + write (for the local config)
- Conversational interview (one question at a time)

## Steps

> **User-facing language rule (applies to every step).** Run detection silently. Do NOT narrate the model's internal reasoning, the SKILL.md's structure, or specific tool names to the user. The user does not see "auth-bootstrap", "meta-tool", "Stage 1.5", "deferred tools list", "tool schema", `COMPOSIO_SEARCH_TOOLS`, `mcp__composio__authenticate`, "direct-vendor MCPs", "companion fetch/send tools", or any other internal label from this file. Tell the user only the **outcome** in plain words: which apps are connected, what's missing, what to click. Brand names users recognize (Composio, Google Calendar, Slack, Gmail, Outlook) are fine. Internal mechanism words ("MCP", "wire", "schema", "stage") are not. When in doubt, write the message the way you'd write it to a non-technical executive assistant.
>
> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The local config (`.claude/ideal-week-ops.local.md`) and in-progress marker (`.ideal-week-onboarding-in-progress.json`) resolve relative to this directory. Do NOT write these into the plugin's own install directory.
>
> **Note on detecting capabilities.** Detection uses two patterns depending on how the user wired their tools.
>
> **Pattern A — Aggregator MCPs (Composio, Zapier MCP, etc.).** These expose generic *meta-tools* rather than one tool per app. Connected apps live inside the meta-tool parameters, not in tool names.
>
> Three-stage detection:
>
> 1. **Presence check** — does the host have a meta-tool prefix loaded? Look for `mcp__composio__*` (Composio) or equivalents. If yes, the aggregator MCP itself is wired.
>
> 1.5. **Auth-bootstrap check.** If Stage 1 found the meta-tool prefix BUT the actual meta-tools (e.g., `COMPOSIO_SEARCH_TOOLS`) are NOT in the loaded tool list — only the auth-bootstrap variants (`mcp__composio__authenticate`, `mcp__composio__complete_authentication`) are present — the MCP server is installed but unauthenticated. In this case, DO NOT walk the user through the full Composio install at Step 2a. Instead, drive the auth flow directly: call the authenticate tool to get the OAuth URL, then deliver it to the user in plain language (per the User-facing language rule at the top of Steps). Verbatim template — say this to the user, not your own reasoning: *"Composio is installed but needs you to re-authorize it for this session. Open this link in your browser: [URL]. After you authorize, the page will redirect to a localhost address that may show a connection error — that's fine, the URL is still valid. Tell me 'done' when you're back, or paste the redirected URL if you'd rather I finish it for you."* Wait for the user to authorize, then re-run detection from Stage 1. The Composio install steps (sign up, connect apps, install for AI client) are unnecessary — the user has already done them; auth just needs to be re-completed in this session.
>
>    **For other aggregator MCPs**: if the same pattern applies (auth-bootstrap tools present, real meta-tools absent), use the equivalent auth-bootstrap call for that aggregator.
>
>    Only if the meta-tool prefix is entirely absent (Stage 1 itself failed) → walk the full Composio install (Step 2a).
>
> 2. **Connection query (authoritative).** For Composio, call `COMPOSIO_SEARCH_TOOLS` with one query per capability (`"list calendar events"`, `"send a slack message"`, `"send an email"`) and parse `toolkit_connection_statuses` — each entry has `{toolkit: <slug>, has_active_connection: <bool>, accounts: [...]}`. A capability is wired iff at least one matched toolkit has `has_active_connection: true`.
>
>    **Important — response handling.** `COMPOSIO_SEARCH_TOOLS` returns a large response (~50KB) including `recommended_plan_steps`, `tool_schemas`, `known_pitfalls`, etc. — IGNORE all of those. Only read `data.toolkit_connection_statuses`. Tool discovery output is not needed at detection time and pollutes context. (Future investigation: `COMPOSIO_MANAGE_CONNECTIONS` may be a cleaner alternative — a more focused meta-tool that returns connection state without tool discovery output. Worth evaluating in a future revision.)
>
>    Slug map for this plugin:
>    - `googlecalendar`, `outlook` → calendar
>    - `slack`, `gmail`, `outlook` → notification
>    - Other slugs → ignore
>
>    For other aggregator MCPs (Zapier, etc.), use their equivalent live-state discovery function. Whatever returns authoritative live state, not cached metadata.
>
> If stage 2 errors or returns empty, treat all capabilities behind that aggregator as NOT wired.
>
> **Pattern B — Direct vendor MCPs (Anthropic Calendar MCP, Slack MCP, etc.).** One or more tools per service, with the vendor name in the prefix:
> - **Calendar read** — `googlecalendar`, `gcal`, `outlook`, `calendar` (e.g. `mcp__claude_ai_Google_Calendar__*`)
> - **Notification send** — `slack`, `gmail`, `outlook`, `email` (e.g. `mcp__slack__*`, `mcp__gmail__*`)
>
> A capability counts as "wired" only if at least one matching tool is BOTH discoverable AND callable. Auth-bootstrap tools (`*_authenticate` with no companion fetch/send tool) do NOT count.
>
> **Both patterns can coexist.** Run pattern A first, then B; merge results; if a capability is satisfied by both, present both options to the user at Step 3.

### 0. Detection (always first)

Before any other step:

1. **Check for existing config** at `.claude/ideal-week-ops.local.md` (workspace-relative).
2. **Check for capabilities** in the host's loaded tool list:
   - Calendar-read tool present?
   - Notification-send tool present (any of slack / gmail / outlook)?
3. **Check for in-progress onboarding** marker `.ideal-week-onboarding-in-progress.json` in the workspace.

Branch:
- **Config exists + calendar wired + at least one notification wired** → refresh mode (jump to "Refresh mode" section at the end of this file).
- **Capabilities wired, no config** → fast path (skip to Step 3).
- **In-progress marker found** → resumed session (load marker, skip questions already answered, continue from where left off).
- **Capabilities missing** → full onboarding (Step 1).

---

### 1. Explain what's needed (full onboarding only)

Use the **Intro** message at `references/canonical-messages.md`, which says:

> "This plugin needs two capabilities wired into your AI: read your calendar, and send one notification per scan. The easiest way to get both is to install **Composio MCP** — a free service that connects your AI to your tools (Google Calendar, Slack, Gmail, Outlook) in one place.
>
> If you'd rather wire your own MCPs (e.g., Anthropic's Google Calendar MCP, your own Slack MCP, etc.), that works too — just let me know what you have and I'll check.
>
> Which would you like to do? (Default is Composio.)"

If user picks Composio → Step 2a. If user picks own MCPs → Step 2b.

### 2a. Composio path (default)

Direct the user (don't run shell commands for them) using the **Composio install walkthrough** message at `references/canonical-messages.md`:

> "Here's the Composio setup, in order:
>
> 1. Sign up at composio.dev (free).
> 2. Click **Connect Apps** in the left sidebar.
> 3. Connect the apps you want to use:
>    - **Google Calendar OR Outlook Calendar** (required — for the daily scan to read events)
>    - **At least one of Slack, Gmail, or Outlook** (required — for delivering the scan notification)
> 4. Click **Install** in the left sidebar.
> 5. Pick the install card for your AI (Claude Code, Claude Desktop, Codex, ChatGPT, etc.) and follow the steps. The install uses **OAuth (one browser-based 'authorize' click)**; no API keys to copy or paste.
>
> Where are you?
> - **(a) Done — installed and authenticated.** I'll re-check what's wired.
> - **(b) Got stuck on a specific step.** Tell me which one and what you saw.
> - **(c) Composio's dashboard looks different from what I described.** Send a screenshot."

On **(a)**: re-run detection. If now successful → continue to Step 3. If detection still doesn't see Composio tools, ask: "Which AI client did you install for? Did you reload/restart the client after authorizing?" (most clients need a session restart to pick up newly-added MCP servers).

On **(b)**: ask what error or screen they're seeing at that specific step. Common gotchas:
- **Step 1 (signup)** — blocked by SSO/corporate firewall → suggest personal Google login or different network
- **Step 3 (connecting an app)** — OAuth window doesn't open or closes immediately → check popup blocker; suggest incognito window
- **Step 5 (AI install)** — terminal command errors out for Claude Code → check `claude --version` is recent; suggest reinstalling Claude Code; check `claude mcp list` to see if the server already exists with the wrong name

On **(c)**: deliver the **Stuck help** section from `references/canonical-messages.md` and walk based on screenshots.

Iterate until install succeeds.

### 2b. Own-MCPs path

Ask: *"What MCPs do you have wired? (e.g., Google Calendar MCP, Slack MCP, Gmail MCP, etc.)"*

User describes their setup. Re-run detection (per the "Note on detecting capabilities" at the top of Steps).

**Verify each claimed capability with a real test call.** Don't trust self-report — MCPs frequently break silently:

- **Calendar** — call the calendar tool with a "list 1 event for today" / "fetch next event" request. If it returns metadata → ✓. If auth/permission/network error → tell the user the specific error; offer to either fix it (re-auth in their MCP's dashboard) or wire Composio's Google Calendar/Outlook as a backup.
- **Notification** — channel-dependent test:
  - Slack → call a low-impact tool like "list channels" or "auth test" to confirm callability without sending a message
  - Gmail → "list labels" or "list 1 message"
  - Outlook → "list folders" or "list 1 message"
  - File → confirm the target folder is writable (the user picks the path at Step 4)

Show the user the result of each test:

> "Tested your wiring:
> - **Calendar** (your Google Calendar MCP): ✓ — pulled the next event
> - **Notification** (your Slack MCP): ✗ — got `Unauthorized: token expired`
>
> One issue: your Slack MCP isn't authorized. Want to fix that, or wire Slack via Composio as a fallback (~2 minutes)?"

If gaps remain (capability not wired, or wired but failed verification), point at the specific missing one and offer the Composio fallback for that capability only.

Iterate until both capabilities are wired AND verified.

---

### 3. Confirm what's wired and pick what to use

Query the host's loaded tools and identify which can handle each capability. Present back to user:

> "Looks like you have these wired:
> - **Calendar**: [Google Calendar via Composio | Outlook via Composio | Google Calendar direct MCP | etc.]
> - **Notification**: [Slack via Composio | Gmail via Composio | Outlook via Composio | Slack direct MCP | etc.]
>
> Which should this plugin use?"

User picks. Capture the **provider name** in running state (`googlecalendar`, `outlook`, `slack`, `gmail`) — NOT the specific tool slug or MCP source.

If only one option exists for a capability, confirm-and-skip ("Using Google Calendar — sound good?").

### 3.1. Multi-calendar account check

After the user picks a calendar provider, check whether multiple accounts exist for that provider in the wiring. The detection mechanism varies by aggregator:

- **Composio (Pattern A).** The `toolkit_connection_statuses[].accounts[]` array from `COMPOSIO_SEARCH_TOOLS` lists all connected accounts for the chosen toolkit. If `accounts.length > 1`, multi-account selection is needed. Each account has `id`, optional `alias`, and a `user_info` block with the email address.
- **Direct vendor MCPs (Pattern B).** Most direct-vendor MCPs are single-account by design. If a vendor MCP supports multi-account, follow its discovery convention (often `list_accounts` or similar). If unclear, treat as single-account.

If multiple accounts exist for the chosen provider, **always ask** (never auto-merge):

> "I see [N] [provider] accounts connected: [list — show alias if set, otherwise email address]. Use one, or scan all merged?"

Capture the user's choice as a list of account identifiers. Prefer human-readable email/alias over raw IDs. Examples of what gets stored:

- Single account picked: `calendar_accounts: ["googlecalendar:sam@atlas.co"]`
- All merged: `calendar_accounts: ["googlecalendar:sam@atlas.co", "googlecalendar:sam@personal.com"]`

If only one account exists, capture the single account's identifier silently (no prompt) so subsequent runs are deterministic.

**Important: all events count equally regardless of source calendar.** No per-calendar rule scoping. The user picked to merge — they own that. (See `scan-ideal-week`'s multi-calendar contract for what happens at scan time.)

Save running state to `.ideal-week-onboarding-in-progress.json`.

---

### 4. Output destination + (optional) notification ping

Two questions: where the log file lives (always-on output), and (optionally) which channel to ping for live notifications.

**4a. Pick the log folder.** Ask:

> "Every scan writes a per-day log file so you have a history of what was flagged. Where should those go?
> Default: `client-profile/ideal-week-scans/` (relative to the workspace)."

Capture answer to running state under `log_folder`. Accept the default if the user says "default" / "yes" / silence.

**4b. Pick the notification channel (optional).** Before asking, **filter** the channel list to only those with a wired notification tool per Step 0 detection. Always include "no ping" regardless. Then ask the user, presenting only the filtered options:

> "Want a ping when the scan finds calendar conflicts? Or just check the log file when you want?
> [Present only the wired notification options here, plus 'no ping']
>
> Examples (illustrative — show only what's actually wired):
> - **gmail** — email to a Gmail address
> - **slack** — Slack DM or channel
> - **outlook** — email to an Outlook address
> - **no ping** — skip; the log file is the only output"

Capture answer to running state under `notification_channel`. If user picks "no ping" → set `notification_channel: none`, skip 4c and 4d entirely.

**Don't list channels the user can't actually use.** If only gmail is wired, the question becomes: *"Want a gmail ping when scans find conflicts, or just the log file?"* Don't surface slack as an option just because the SKILL.md mentions it — the user has to wire it first.

**4c. Pick the recipient (only if channel ≠ none).** Ask based on channel:

- `gmail` → "What's the recipient email address?"
- `slack` → "What's the recipient? Slack handle (`@user`) or channel (`#chan`)."
- `outlook` → "What's the recipient email address?"

Capture answer to running state under `notification_target`.

**4d. Slack self-trap check (only if `notification_channel` is `slack`).**

If `notification_target` resolves to the same Slack account that's connected to the chosen Slack workspace in Composio (or in the user's direct Slack MCP), deliver this warning:

> "Heads up — Slack does not push self-DMs, so this notification will deliver silently. Some options:
>
> - **Use this anyway** — message lands in your DMs, you'll see it next time you open Slack but no push notification fires
> - **Switch to gmail or outlook** — both deliver to your inbox + Sent with normal notifications even when sending to yourself
> - **Pick a different recipient** — your EA's handle or a shared channel both you and the EA are in
>
> What would you like to do?"

If user picks "use this anyway" → proceed with current values. If they switch → re-ask 4b (channel) or 4c (recipient), update running state.

**Gmail and Outlook to self always work** — no warning fires for those channels.

How to detect "same Slack account":
- Compare `notification_target` (handle or channel) against the user_info / account metadata on the connected Slack toolkit
- If the handle matches the user's own Slack ID for that workspace → trap fires
- If channel-style (`#chan`) → trap doesn't fire (channels aren't self-DMs)
- If unsure (can't read the connected account's identity) → skip the warning rather than false-firing

Save running state to `.ideal-week-onboarding-in-progress.json`.

---

### 5. Write `.claude/ideal-week-ops.local.md`

Write the captured running state to `.claude/ideal-week-ops.local.md` in the workspace. Format (YAML frontmatter, no markdown body):

````markdown
---
# Where the ideal-week document lives (read by extract + scan)
ideal_week_path: client-profile/ideal-week.md

# Calendar accounts to scan — list, even if only one
# Format: <provider>:<account-identifier>
# Provider: googlecalendar or outlook
# Account identifier: email address (or alias if user set one in Composio)
calendar_accounts:
  - googlecalendar:sam@atlas.co

# Log file folder — every scan writes a per-day log here (always-on output, not optional)
log_folder: client-profile/ideal-week-scans/

# Notification channel — optional live ping when scans find conflicts
# Values: gmail | slack | outlook | none
notification_channel: none

# Recipient — required only when notification_channel is not "none"
#   gmail   → email address
#   slack   → @handle or #channel
#   outlook → email address
notification_target: ""

# Scan window in days from invocation date.
# Default 1 = look at one day. Evening scan looks at tomorrow; morning scan looks at today.
scan_window_days: 1

# Dry-run mode — when true, scan prints log content + notification body to terminal but doesn't write or send.
dry_run: false
---
````

Substitute the actual captured values for each field. If the user picked multiple calendar accounts at Step 3.1, the `calendar_accounts` list has multiple entries.

After writing the config, delete `.ideal-week-onboarding-in-progress.json` (the marker is no longer needed).

### 6. Hand-off — chain directly into `extract-ideal-week`

Wiring is done. The user said something like "set up ideal week ops" expecting the full arc — wiring + ruleset capture — so do NOT stop at wiring and wait for them to type another trigger phrase. Chain directly into the `extract-ideal-week` skill from this same plugin.

Tell the user (verbatim wording — keep it human, second-person, no "the exec"):

> "Wiring done — saved to `.claude/ideal-week-ops.local.md`. Now I'll capture how your week should actually run — rhythms, deep-work blocks, protected time, VIP overrides — so the daily scan has rules to check against. This takes about 10–15 minutes: one question at a time, free-form answers, you can pause and resume any time. Ready?"
>
> "(If you'd rather pause here and just keep the wiring done, say 'pause' and I'll stop. Re-run `setup` any time to change channels, swap calendars, or add a calendar account.)"

Then load `skills/extract-ideal-week/SKILL.md` from this same plugin and continue execution from its Step 0 — same conversation, no user trigger phrase required. (`extract-ideal-week`'s Step 0 runs a precondition check that passes silently because this skill just wrote the config.)

**If the user replies "pause" / "stop" / "just wiring for now" / similar:**

Exit gracefully and tell them:

> "Got it — wiring's saved. Re-run `setup` to change wiring later, or just say 'extract ideal week' when you're ready to capture your rules. Once both are done, `scan-ideal-week` will run on the schedule you set."

Do not auto-chain in that case.

### Refresh mode (alternative entry from Step 0)

If config already exists at `.claude/ideal-week-ops.local.md` AND capabilities are wired, skip Steps 1–2 (no Composio install walk-through) and run refresh mode instead.

For each field in the existing config, show the current value and ask: *"Keep this, or change?"* Accept the answer per-field. If the user says change, ask the corresponding question for that field and run the relevant logic (re-pick provider, re-verify accounts, change channel or recipient with a fresh self-trap check).

Refresh DOES re-verify capabilities are still wired by re-detecting the host's loaded tools (an MCP could have been disconnected or its auth could have expired since last setup).

After the walk-through, fall through to Step 5 (write config — overwriting with updated values).

## Output

A `.claude/ideal-week-ops.local.md` file with the user's wiring choices. The work skills (`extract-ideal-week`, `scan-ideal-week`) read this file at their start.

## Customization

To change the channel + recipient: re-run `setup` (refresh mode walks through changes per-field).

To change the wiring (e.g., swap from Composio Google Calendar to a direct Google Calendar MCP): disconnect the old wiring in your AI host, connect the new one, then re-run `setup` — detection picks up the new tool.

To change the wizard wording: edit `references/canonical-messages.md`.
