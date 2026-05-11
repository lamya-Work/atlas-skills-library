---
name: setup
description: One-time onboarding wizard for the meeting-performance-analyzer plugin. Detects whether the host's AI has the capabilities the audit needs (meeting-tool read, email send) connected. Walks the user through Composio setup if they're missing — including the full first-time install flow (sign up at composio.dev → connect apps → install for AI host). Captures which meeting tool to use, which email account to send the notification from, the executive's name and optional email (used to attribute leadership observations in the audit), and the output folder for briefs. Resumable mid-flow. Re-runnable to refresh any field.
when_to_use: Run before using `extract-meeting-preferences` or `meeting-performance-audit` for the first time. Trigger phrases — "set up meeting performance analyzer", "configure meeting analyzer", "connect meeting analyzer", "refresh meeting analyzer setup", "redo meeting analyzer setup". Auto-trigger when `meeting-performance-audit` reports no config exists at `client-profile/meeting-performance-analyzer.local.md`.
atlas_methodology: neutral
---

# setup

Onboarding wizard for `meeting-performance-analyzer`. Detection-first: figures out what the user already has connected and only walks them through what's missing. Five branches: refresh (config exists), fast (capabilities connected, no config), resumed (in-progress marker found), auth-bootstrap (Composio installed but unauthenticated), full onboarding (Composio not installed at all), own-MCPs (user prefers direct vendor MCPs).

## Purpose

The work skills (`extract-meeting-preferences` and `meeting-performance-audit`) need two capabilities connected: meeting-tool read (to pull transcripts and metadata) and email send (to deliver the audit-complete heads-up). This skill captures the wiring choice, account selection, executive identity, and output folder. Saves to `client-profile/meeting-performance-analyzer.local.md` in the workspace. Resumable. Re-runnable.

The audit's weekly cadence is encoded in its own design (7-day cap window). This skill does NOT capture a schedule anchor — recurring runs are wired in whichever scheduler the user's AI host provides (Cowork's scheduler, Claude Code's `/schedule`, GitHub Actions, OS cron, etc.). The Hand-off message surfaces this as a recommendation.

## Inputs

- **Existing config** (optional) — if `client-profile/meeting-performance-analyzer.local.md` exists, the skill enters refresh mode automatically.
- **Host's loaded tool list** — required. Skill reads what tools the host has connected and uses that to detect capability presence.
- **In-progress marker** (optional) — `.meeting-performance-analyzer-onboarding-in-progress.json` in the workspace, if a previous run was interrupted.
- **Conversational interview** — questions asked one at a time, paraphrased back for confirmation.

## Required capabilities

(Abstract list — no specific tool names. Validated default wiring is Composio MCP; see plugin README §4 for alternatives.)

- Detection of available meeting-tool / email tools in the host's loaded tool list
- File read + write (for the local config and in-progress marker)
- Conversational interview (one question at a time)

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The local config (`client-profile/meeting-performance-analyzer.local.md`) and in-progress marker (`.meeting-performance-analyzer-onboarding-in-progress.json`) resolve relative to this directory. Do NOT write these into the plugin's own install directory. Plugin-internal references (`references/canonical-messages.md`) use the skill folder's own `references/` subdirectory.
>
> **Note on detecting capabilities.** Detection uses two patterns depending on how the user connected their tools.
>
> **Pattern A — Aggregator MCPs (Composio, Zapier MCP, etc.).** These expose generic *meta-tools* rather than one tool per app. Connected apps live inside the meta-tool parameters, not in tool names.
>
> Three-stage detection:
>
> 1. **Presence check** — does the host have a meta-tool prefix loaded? Look for `mcp__composio__*` (Composio) or equivalents. If yes, the aggregator MCP itself is connected.
>
> 1.5. **Auth-bootstrap check.** If Stage 1 found the meta-tool prefix BUT the actual meta-tools (e.g., `COMPOSIO_SEARCH_TOOLS`) are NOT in the loaded tool list — only the auth-bootstrap variants (`mcp__composio__authenticate`, `mcp__composio__complete_authentication`) are present — the MCP server is installed but unauthenticated. In this case, DO NOT walk the user through the full Composio install at Step 2a. Instead, drive the auth flow directly: load and show the **Composio re-auth (Stage 1.5)** section from `references/canonical-messages.md` (the section opens with a status line so the user can tell why the full install isn't running), call the authenticate tool, deliver the URL, wait for the user to authorize, then re-run detection from Stage 1.
>
>    Only if the meta-tool prefix is entirely absent (Stage 1 itself failed) → walk the full Composio install (Step 2a).
>
> 2. **Connection enumeration (authoritative).** Composio's MCP server annotates the loaded `COMPOSIO_SEARCH_TOOLS` tool description with the user's currently-connected toolkit slugs. The annotation reads roughly: *"User has manually connected the apps: `<comma-separated slug list>`. Prefer these apps when intent is unclear."* Detection reads this annotation directly — no `COMPOSIO_SEARCH_TOOLS` call required (and the prior search-driven approach could miss connected toolkits when the search query didn't relevance-match the slug).
>
>    Three substeps:
>
>    **2a. Read the annotation.** Inspect the loaded `COMPOSIO_SEARCH_TOOLS` tool description as the host exposes it. Extract the slug list after the `User has manually connected the apps:` prefix (case-insensitive, comma-separated, trim whitespace). The annotation reflects the MCP server's view at the time the host loaded the tool registry — if the user connected a toolkit AFTER the session started, the host typically needs a session restart to pick it up. If detection looks stale (e.g., the user just connected something), prompt for a session restart before going further.
>
>    **2b. Map slugs to capabilities** using the slug map below (matches done locally on the slug list — no further tool calls). A capability is *connected* iff at least one of its slug-mapped toolkits appears in the annotation.
>
>    **2c. Per-toolkit account detail.** When a per-source step (Step 4 meeting-tool selection, Step 5 email account picker) needs the toolkit's connected accounts (IDs, aliases, statuses), call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]`. The response returns `accounts[]` for that toolkit. The call accepts an array, so multiple toolkits can be batched into one round-trip when more than one is needed at the same step.
>
>    **Fallback if the annotation is missing.** If the host stripped the description annotation during tool ingestion (some hosts only forward the schema, not the human-readable description), fall back to a single `COMPOSIO_MANAGE_CONNECTIONS` call with the full slug map as the `toolkits` array. For each slug, a non-empty `accounts[]` means connected. Slower than the annotation read but doesn't depend on description forwarding.
>
>    **Response sizes** are small for both the annotation read and `COMPOSIO_MANAGE_CONNECTIONS` action `list`. No out-of-band file overflow handling needed.
>
>    Slug map for this plugin:
>    - `fathom`, `granola`, `fireflies` → meeting tool
>    - `gmail` → notification email
>    - Other slugs → ignore
>
>    For other aggregator MCPs (Zapier, etc.), use their equivalent enumeration / list-connections function.
>
> If stage 2 returns no toolkits (annotation absent AND fallback `MANAGE_CONNECTIONS` call returns empty `accounts[]` for every slug in the map), treat all capabilities behind that aggregator as NOT connected.
>
> **Pattern B — Direct vendor MCPs.** One or more tools per service, with the vendor name in the prefix:
> - **Meeting tool read** — `fathom`, `granola`, `fireflies` (e.g., `mcp__fathom__*`)
> - **Email send** — `gmail` (e.g., `mcp__gmail__*`)
>
> A capability counts as connected only if at least one matching tool is BOTH discoverable AND callable. Auth-bootstrap tools (`*_authenticate` with no companion fetch/send tool) do NOT count.
>
> **Both patterns can coexist.** Run Pattern A first, then B; merge results.

### 0. Detection (always first)

**Resolve `client-profile/meeting-performance-analyzer.local.md` against the workspace (user's CWD), NOT the plugin install directory. If the path doesn't exist in the workspace, do NOT fall back to checking the plugin's own folder.**

Three checks:

1. **Existing config** at `<workspace>/client-profile/meeting-performance-analyzer.local.md`.
2. **Capabilities** in the host's loaded tool list, per the Pattern A + B detection above.
3. **In-progress marker** `<workspace>/.meeting-performance-analyzer-onboarding-in-progress.json`.

Branch:

| Condition | Branch |
|---|---|
| Config exists + meeting tool connected + email connected | **Refresh mode** (jump to "Refresh mode" section below) |
| Capabilities connected, no config | **Fast path** (load and show the **Fast-path acknowledgement** section from `references/canonical-messages.md` — substituting readable toolkit names for `[list of connected toolkits]` — then skip to Step 3) |
| In-progress marker found | **Resumed session** (load marker, skip questions already answered, continue from where left off) |
| Composio meta-tool prefix loaded but only auth-bootstrap variants (Stage 1.5 fired) | **Auth-bootstrap** (load and show the **Composio re-auth (Stage 1.5)** section, drive auth flow, re-detect) |
| Capabilities missing entirely | **Full onboarding** (continue to Step 1) |

**Resumability rule** — for Steps 1–7, save running state to `<workspace>/.meeting-performance-analyzer-onboarding-in-progress.json` after every captured answer.

---

### 1. Explain what's needed (full onboarding only)

Load and show the **Intro** section from `references/canonical-messages.md`.

If the user picks Composio → Step 2a. If the user picks own MCPs → Step 2b.

### 2a. Composio path (default)

Load and show the **Composio install walkthrough** section from `references/canonical-messages.md`.

On **(a) Done**: re-run detection. If now successful → continue to Step 3. If detection still doesn't see Composio tools, ask: "Which AI client did you install for? Did you reload/restart the client after authorizing?" (most clients need a session restart to pick up newly-added MCP servers).

On **(b) Stuck**: ask what error or screen they're seeing at that specific step. Common gotchas:
- **Step 1 (signup)** — blocked by SSO/corporate firewall → suggest personal Google login or different network
- **Step 3 (connecting an app)** — OAuth window doesn't open or closes immediately → check popup blocker; suggest incognito window
- **Step 5 (AI install)** — terminal command errors out → check the AI client version is recent; suggest reinstalling

On **(c) Dashboard differs**: deliver the **Stuck help** section from `references/canonical-messages.md` and walk based on screenshots.

Iterate until install succeeds.

### 2b. Own-MCPs path

Ask the user what tools they have connected. Re-run detection per the "Note on detecting capabilities" callout.

**Verify each claimed capability with a real test call.** Don't trust self-report — MCPs frequently break silently:

- **Meeting tool** — call the tool with a "list 1 transcript from the past week" or equivalent. If it returns metadata → ✓. If auth/permission/network error → tell the user the specific error; offer to either fix it (re-auth in their MCP's dashboard) or wire Composio's Fathom / Granola / Fireflies as a backup.
- **Notification email** — call the email tool with "list labels" (Gmail) or equivalent low-impact discovery call.

Show the user the result of each test:

> "Tested the connections you described:
> - **Meeting tool** (your Fathom MCP): ✓ — pulled the latest transcript metadata
> - **Email** (your Gmail MCP): ✗ — got `Unauthorized: token expired`
>
> One issue: your Gmail MCP isn't authorized. Want to fix that, or connect Gmail via Composio as a fallback (~2 minutes)?"

If a capability is connected but failed verification, point at the specific issue and offer the Composio fallback for that capability only.

Iterate until both capabilities are connected AND verified.

---

### 3. Confirm what's connected and pick the meeting tool

Query the host's loaded tools and identify which can handle each capability. If both Composio and a direct vendor MCP are connected for the same capability, both options are surfaced; the user picks.

Load and show the **Connection choice** section from `references/canonical-messages.md`, substituting the actual list of detected sources.

If only one option exists for a capability, confirm-and-skip ("Using Fathom — sound good?") rather than asking.

Capture the picked **provider name** (`fathom`, `granola`, `fireflies`, `gmail`) and the **wiring source** (`composio` or `direct`) into running state under `meeting_tool` and `email_provider`.

#### 3.1 Multi-meeting-tool case

If multiple of {Fathom, Granola, Fireflies} are connected (e.g., the executive uses two), ask which to use for this audit. Save the answer; the audit pulls transcripts from the picked tool only.

> "I see [N] meeting tools connected: [Fathom, Granola]. Which one should the audit pull from?"

(Single picked tool only — the audit does not merge transcripts across tools in v1.)

#### 3.2 Multi-account case for the picked meeting tool

The picked meeting toolkit (Fathom / Granola / Fireflies) may itself have multiple accounts connected via the same aggregator — e.g., the executive's personal Fathom and a shared workspace Fathom both authorized through Composio. Detection mirrors Step 4 (email):

- **Composio (Pattern A).** Call the per-toolkit account detail step (Stage 2c) for the picked meeting toolkit: `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: "<picked slug>", action: "list"}]`. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1`, multi-account selection is needed.
- **Direct vendor MCPs (Pattern B).** Most direct-vendor meeting MCPs are single-account by design. If a vendor MCP supports multi-account, follow its discovery convention. If unclear, treat as single-account and leave `meeting_tool_account_id` null.

If multiple accounts exist, **always ask** (never auto-pick):

> "I see [N] [Fathom/Granola/Fireflies] accounts connected: [list — alias if set, otherwise account identifier]. Which one should the audit pull transcripts from?"

Capture the picked account ID under `meeting_tool_account_id` in running state.

If only one account exists (or the source is direct-vendor and treated as single-account), capture silently — leave `meeting_tool_account_id` null.

#### 3.3 Verify the picked meeting toolkit exposes the actions the audit needs

A toolkit can be connected without exposing the actions the audit calls — rare, but possible (toolkit slug authorized for a different scope, vendor renamed an action, etc.). Probe once before writing the config so the audit doesn't fail later:

- **Composio (Pattern A).** Call the toolkit-discovery capability with two queries scoped to the picked meeting toolkit: *"list meetings in the past 7 days"* and *"fetch meeting transcript by recording ID"*. Confirm the response returns at least one action slug per query, scoped to the picked toolkit (e.g., `FATHOM_LIST_MEETINGS` and `FATHOM_GET_RECORDING_TRANSCRIPT` if `fathom` was picked). Don't hardcode the names — read whatever slugs the discovery layer returns.
- **Direct vendor MCPs (Pattern B).** Confirm the host's loaded tool list contains at least one tool whose name or description matches "list meetings" intent and one matching "fetch transcript" intent for the picked toolkit's prefix.

If the probe returns no matching actions for either query, surface this to the user before continuing:

> "Looks like [toolkit] is connected, but it doesn't have the access the audit needs — specifically, it can't list your meetings or pull a transcript. Would you like to pick a different meeting tool, or sign in to [toolkit] again to check that read access is turned on?"

If the probe succeeds, capture nothing — the audit re-runs the same discovery at runtime. The probe is just a setup-time sanity check to fail fast.

Save running state to `.meeting-performance-analyzer-onboarding-in-progress.json`.

---

### 4. Email account picker

The notification toolkit (Gmail) may have multiple accounts connected — same-address aliases with different OAuth scopes are common. Detection mechanism varies by aggregator:

- **Composio (Pattern A).** Call the per-toolkit account detail step (Stage 2c) for the Gmail toolkit: `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: "gmail", action: "list"}]`. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1`, multi-account selection is needed.
- **Direct vendor MCPs (Pattern B).** Most direct-vendor MCPs are single-account by design. If a vendor MCP supports multi-account, follow its discovery convention. If unclear, treat as single-account.

If multiple accounts exist, **always ask** (never auto-pick):

> "I see [N] Gmail accounts connected: [list — alias if set, otherwise email address]. Which one should the audit-complete email send from?"

Capture the picked account ID under `email_account_id` in running state. Note the picked account's display name (alias or email) for the hand-off message.

If only one account exists, capture silently (no prompt).

#### 4a. Recipient

Ask:

> "What's the email address that should receive the audit-complete heads-up? (Most users send to themselves or to their EA.)"

Capture under `email_recipient` in running state.

---

### 5. Executive identity

The audit's leadership-reflection section needs to attribute observations to the executive specifically (rather than to other meeting participants). Capture the executive's name as it appears in transcripts, and optionally their email.

> "What's the executive's name as it appears in meeting transcripts? (e.g., 'Sam Smith', 'Sam', or 'S. Smith' — match what your meeting tool actually outputs.)"

Capture under `exec_name` in running state.

> "Optional: what's the executive's email? Some meeting tools tag participants by email, which makes attribution more reliable than name-only matching. (Skip if unsure.)"

Capture under `exec_email` in running state if provided; otherwise leave null.

---

### 6. Output folder

Ask:

> "Where should the weekly briefs be saved? Default: `meeting-briefs/` (relative to your workspace)."

Capture under `output_folder`. Accept the default if the user says "default" / "yes" / silence.

If the user provides an absolute path (e.g., for a cross-project library), accept it as-is. Note in the hand-off: chat-link previews on the audit-complete email may not be clickable if the path is outside the current workspace.

Save running state to `.meeting-performance-analyzer-onboarding-in-progress.json`.

---

### 7. Write `client-profile/meeting-performance-analyzer.local.md`

Write the captured running state to `<workspace>/client-profile/meeting-performance-analyzer.local.md`. Format (YAML frontmatter, no markdown body):

````markdown
---
# Meeting tool — which toolkit to pull transcripts from
meeting_tool: fathom            # fathom | granola | fireflies | <direct-vendor name>
meeting_tool_source: composio   # composio | direct
meeting_tool_account_id: ""     # optional — only set for Composio multi-account; null for single-account or direct MCPs

# Notification email — for the audit-complete heads-up (sent once per run)
email_provider: gmail
email_source: composio
email_account_id: ""            # required when email_source is composio with multiple accounts; null otherwise
email_recipient: placeholder@example.com

# Executive identity — used by the audit to attribute leadership observations
exec_name: "Sam Smith"
exec_email: ""                  # optional — improves attribution when transcripts include attendee emails

# Output folder for the weekly brief — relative to the workspace, or an absolute path
output_folder: meeting-briefs/

# Dry-run mode — when true, the audit prints the brief and notification body to terminal but doesn't write files or send email.
dry_run: false
---
````

Substitute the actual captured values for each field. After writing the config, delete `.meeting-performance-analyzer-onboarding-in-progress.json` (the marker is no longer needed).

---

### 8. Hand-off

Load and show the **Hand-off** section from `references/canonical-messages.md`.

---

### Refresh mode (alternative entry from Step 0)

If config already exists at `client-profile/meeting-performance-analyzer.local.md` AND capabilities are connected, skip Steps 1–2 (no Composio install walk-through) and run refresh mode instead.

For each field in the existing config, show the current value and ask: *"Keep this, or change?"* Accept the answer per-field. If the user says change, ask the corresponding question for that field.

Refresh DOES re-verify capabilities are still connected by re-detecting the host's loaded tools (an MCP could have been disconnected or its auth could have expired since last setup).

After the walk-through, fall through to Step 7 (write config — overwriting with updated values).

## Output

A `client-profile/meeting-performance-analyzer.local.md` file with the user's wiring choices. The work skills (`extract-meeting-preferences`, `meeting-performance-audit`) read this file at their start.

## Customization

To change any field: re-run `setup` (refresh mode walks through changes per-field).

To swap wiring (e.g., move from Composio Gmail to a direct Gmail MCP): disconnect the old wiring in your AI host, connect the new one, then re-run `setup` — detection picks up the new tool.

To change the wizard wording: edit `references/canonical-messages.md`.
