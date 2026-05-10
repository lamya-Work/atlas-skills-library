---
name: setup
description: One-time onboarding wizard for the daily-followup-tracker plugin. Detects whether the host's AI has the capabilities the sweep needs (email read / messaging read / meeting transcript read / task system read / notification send) connected. Walks the user through Composio setup if any are missing. Captures which sources to use, per-source detail (channels, task containers, meeting transcript location), the user's identity for speaker matching, the per-channel voice profile, the meeting follow-up default channel, the notification target, and the output folder. Resumable mid-flow. Re-runnable to refresh any field.
when_to_use: Run before using `daily-followup-tracker` for the first time. Trigger phrases — "set up daily followup tracker", "configure daily followup tracker", "wire daily followup tracker setup", "refresh daily followup tracker setup", "redo daily followup tracker wiring". Auto-trigger when `daily-followup-tracker` reports no config exists at `client-profile/daily-followup-tracker.local.md`.
atlas_methodology: neutral
---

# setup

Onboarding wizard for `daily-followup-tracker`. Detection-first: figures out what the user already has connected and only walks them through what's missing. Three branches: refresh (config exists), fast (capabilities connected, no config), full onboarding (capabilities missing).

## Purpose

The work skill (`daily-followup-tracker`) needs at least one of four data-source capabilities connected — email read, messaging read, meeting transcript read, or task system read — plus a notification target (gmail, outlook, or slackbot) for the report-ready heads-up. This skill captures which sources to use, the per-source details (email accounts, Slack channels, meeting transcript location, task containers + assignment field), the user's identity for transcript speaker matching, per-channel voice profiles, the meeting follow-up default channel, the notification target, and the output folder for reports. Saves to `client-profile/daily-followup-tracker.local.md`. Resumable. Re-runnable.

The sweep's daily cadence is encoded in its own design (rolling 24h window). This skill does NOT capture a schedule anchor — recurring runs are wired in whichever scheduler the user's AI host provides (Claude Code's `/schedule`, GitHub Actions cron, OS cron, or any other scheduler the host exposes). The Hand-off message surfaces this as a recommendation.

## Inputs

- **Existing config** (optional) — if `client-profile/daily-followup-tracker.local.md` exists, the skill enters refresh mode automatically.
- **Host's loaded tool list** — required. Skill reads what tools the host has connected and uses that to detect capability presence.
- **In-progress marker** (optional) — `.daily-followup-tracker-onboarding-in-progress.json` in the workspace, if a previous run was interrupted.
- **Conversational interview** — questions asked one at a time, paraphrased back for confirmation.

## Required capabilities

(Abstract list — no specific tool names. Validated default wiring is Composio MCP; see plugin README §4 for alternatives.)

- Detection of available email / messaging / meeting transcript / task system / notification-send tools in the host's loaded tool list
- File read + write (for the local config, voice profile, and in-progress marker)
- Conversational interview (one question at a time)

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The local config (`client-profile/daily-followup-tracker.local.md`) and in-progress marker (`.daily-followup-tracker-onboarding-in-progress.json`) resolve relative to this directory. Do NOT write these into the plugin's own install directory. Plugin-internal references (templates, methodology files, `references/canonical-messages.md`) use `../../` from this skill folder to mean plugin root.
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
> 2. **Connection enumeration (authoritative).** Composio's MCP server annotates the loaded `COMPOSIO_SEARCH_TOOLS` tool description with the user's currently-connected toolkit slugs. The annotation reads roughly: *"User has manually connected the apps: `<comma-separated slug list>`. Prefer these apps when intent is unclear."* Detection reads this annotation directly — no `COMPOSIO_SEARCH_TOOLS` call required (and the prior search-driven approach missed connected toolkits when the search query didn't relevance-match the slug — Notion-as-tasks was the dogfood example).
>
>    Three substeps:
>
>    **2a. Read the annotation.** Inspect the loaded `COMPOSIO_SEARCH_TOOLS` tool description as the host exposes it. Extract the slug list after the `User has manually connected the apps:` prefix (case-insensitive, comma-separated, trim whitespace). The annotation reflects the MCP server's view at the time the host loaded the tool registry — if the user connected a toolkit AFTER the session started, the host typically needs a session restart to pick it up. If detection looks stale (e.g., the user just connected something), prompt for a session restart before going further.
>
>    **2b. Map slugs to capabilities** using the slug map below (matches done locally on the slug list — no further tool calls). A capability is *connected* iff at least one of its slug-mapped toolkits appears in the annotation.
>
>    **2c. Per-toolkit account detail.** When a per-source step (Step 4a email, Step 4d task tool's container/schema fetches, Step 8 notification) needs the toolkit's connected accounts (IDs, aliases, statuses), call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]`. The response returns `accounts[]` for that toolkit. The call accepts an array, so multiple toolkits can be batched into one round-trip when more than one is needed at the same step.
>
>    **Fallback if the annotation is missing.** If the host stripped the description annotation during tool ingestion (some hosts only forward the schema, not the human-readable description), fall back to a single `COMPOSIO_MANAGE_CONNECTIONS` call with the full slug map as the `toolkits` array (each entry `{name: <slug>, action: "list"}`). For each slug, a non-empty `accounts[]` means connected. Slower than the annotation read but doesn't depend on description forwarding.
>
>    **Response sizes** are small for both the annotation read (a one-line text snippet in the tool description) and `COMPOSIO_MANAGE_CONNECTIONS` action `list` (account metadata only — IDs, aliases, statuses). No out-of-band file overflow handling needed — that was a `COMPOSIO_SEARCH_TOOLS` problem, and detection no longer calls `SEARCH_TOOLS`.
>
>    Slug map for this plugin:
>    - `gmail`, `outlook` → email
>    - `slack` → messaging
>    - `fathom`, `granola`, `fireflies` → meeting transcripts (direct)
>    - `notion`, `googledrive` → meeting transcripts (folder fallback)
>    - `notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday` → tasks
>    - `slackbot` → notification target *(distinct from `slack`; do not conflate)*
>    - `gmail`, `outlook` → notification target (also viable)
>    - Other slugs → ignore
>
>    For other aggregator MCPs (Zapier, etc.), use their equivalent enumeration / list-connections function.
>
> If stage 2 returns no toolkits (annotation absent AND fallback `MANAGE_CONNECTIONS` call returns empty `accounts[]` for every slug in the map), treat all capabilities behind that aggregator as NOT connected.
>
> **Pattern B — Direct vendor MCPs.** One or more tools per service, with the vendor name in the prefix:
> - **Email read** — `gmail`, `outlook`
> - **Messaging read** — `slack`
> - **Meeting transcripts** — `fathom`, `granola`, `fireflies`, `notion`, `googledrive`
> - **Task system** — `notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday`
> - **Notification send** — `gmail`, `outlook`, `slackbot`
>
> A capability counts as connected only if at least one matching tool is BOTH discoverable AND callable. Auth-bootstrap tools (`*_authenticate` with no companion fetch/send tool) do NOT count.
>
> **Both patterns can coexist.** Run Pattern A first, then B; merge results.

### 0. Detection (always first)

**Resolve `client-profile/daily-followup-tracker.local.md` against the workspace (user's CWD), NOT the plugin install directory. If the path doesn't exist in the workspace, do NOT fall back to checking the plugin's own folder.**

Three checks:

1. **Existing config** at `<workspace>/client-profile/daily-followup-tracker.local.md`.
2. **Capabilities** in the host's loaded tool list, per the Pattern A + B detection above.
3. **In-progress marker** `<workspace>/.daily-followup-tracker-onboarding-in-progress.json`.

Branch:

| Condition | Branch |
|---|---|
| Config exists + at least one source connected + notification connected | **Refresh mode** (jump to "Refresh mode" section below) |
| Capabilities connected, no config | **Fast path** (load and show the **Fast-path acknowledgement** section from `references/canonical-messages.md` — substituting the readable toolkit names from Stage 2's slug map for `[list of connected toolkits]` — then skip to Step 3) |
| In-progress marker found | **Resumed session** (load marker, skip questions already answered, continue from where left off) |
| Capabilities missing | **Full onboarding** (continue to Step 1) |

**Resumability rule** — for Steps 1–9, save running state to `<workspace>/.daily-followup-tracker-onboarding-in-progress.json` after every captured answer.

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

- **Email** — call the email tool with a "list 1 message in the last 24h" / "fetch latest message" request.
- **Messaging (Slack)** — call "list channels" or "auth test" (low-impact, no message sent).
- **Meeting transcripts (direct)** — call "list 1 transcript" or equivalent.
- **Meeting transcripts (folder fallback)** — Notion → "list pages in DB"; Drive → "list files in folder".
- **Task system** — call with "list 1 task assigned to the user" or "list tasks limit=1".
- **Notification send** —
  - Gmail / Outlook → "list labels" or "list folders" (low-impact)
  - Slackbot → "post message" intent — but DO NOT actually send during verification; confirm the tool slug is reachable and the bot has post permissions in the target channel

Show the user the result of each test. For any failed verification, deliver the **Capabilities not wired — pre-flight** section from `references/canonical-messages.md` scoped to the failing capability and offer to walk through Composio install for that one app.

Iterate until each capability the user wants is connected AND verified. If the user only wants a subset of the four sources, only those need to pass.

---

### 3. Source picker

Load and show the **Source picker** section from `references/canonical-messages.md`. Filter the displayed options to capabilities that have at least one connected tool — if the user wants a source they haven't connected, briefly offer to connect it (loop back to Step 2 scoped to that one source).

User picks a subset of {email, messaging, meeting transcripts, task system}. **At least one required.** If the user picks none, re-ask once. Save the selection list to running state.

---

### 4. Per-source detail capture

Sub-steps fire only for sources picked at Step 3.

#### 4a. Email (only if email picked)

Determine provider (Gmail / Outlook) — usually inferable from the connected toolkit name. If both providers have connected toolkits, ask which one to use.

**Multi-account check.** Call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]` for the chosen email toolkit. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1`, load and show the **Email account picker** section from `references/canonical-messages.md`.

Capture as a list under `email.accounts`, format `<provider>:<account-identifier>`. Examples:
- Single account: `["gmail:placeholder@example.com"]`
- All merged: `["gmail:placeholder@example.com", "gmail:other@example.com"]`

If only one account exists, capture silently (no prompt) so subsequent runs are deterministic.

Save under `email.provider` and `email.accounts`. Example:

```yaml
email:
  provider: gmail
  accounts:
    - gmail:placeholder@example.com
```

#### 4b. Messaging (only if messaging picked)

**First ask — scope.** Load and show the **Messaging scope** section from `references/canonical-messages.md`. The user picks one of:
- `channels` — only the channels they name (no DM scanning)
- `dms` — only DMs (no channel scanning)
- `both` — channels they name plus all DMs

Save under `messaging.scope`.

**Second ask — channel list (only if scope ∈ {channels, both}).** Load and show the **Slack channel name-and-verify** section from `references/canonical-messages.md`. User pastes a list of channel names (`#exec-team` style) and/or channel IDs (`C0123ABCD` style).

For each entry, attempt to resolve via the connected Slack tool's lookup function (handles both name and ID formats). If any miss, show the resolution-feedback message with the found / missed lists. Loop until all entries resolve.

Save the resolved list under `messaging.channels`. Each entry has `id` and `name`. If scope is `dms`, save `messaging.channels: []`.

Example (scope = `both`):

```yaml
messaging:
  provider: slack
  scope: both
  channels:
    - id: C0123ABCD
      name: "#exec-team"
```

Example (scope = `dms`):

```yaml
messaging:
  provider: slack
  scope: dms
  channels: []
```

#### 4c. Meeting transcripts (only if meetings picked)

Load and show the **Meetings source two-tier** section from `references/canonical-messages.md`.

**First ask** — direct integration check (Fathom / Granola / Fireflies). If yes:
- Confirm the toolkit is connected via re-detection.
- If not connected, loop back to Step 2 (Composio install walkthrough scoped to that one app).
- Save `meetings.provider: <fathom|granola|fireflies>`.

**Fallback ask** (only if first ask = no) — Notion DB or Drive folder. User names the location string (e.g., "Notion DB: Meeting Transcripts" or "Drive folder: /Meetings/2026"). Verify Notion or Google Drive is connected via detection; if not, offer Composio install scoped to that one app.
- Save `meetings.provider: <notion|googledrive>` and `meetings.location: "<the location string>"`.

**Neither path works** — record `meetings: skipped` in running state. The sweep will skip meetings and note it in the report header. Move on.

**4c.4 Verify the picked meetings toolkit exposes the actions the sweep needs.**

Skip this substep if 4c saved `meetings: skipped`. Otherwise probe once before continuing — a toolkit can be connected without exposing the actions the sweep calls (toolkit slug authorized for a different scope, vendor renamed an action, etc.):

- **Composio (Pattern A).** Call the toolkit-discovery capability with two queries scoped to the picked `meetings.provider`. Pick the queries by provider class:
  - For direct meeting providers (`fathom`, `granola`, `fireflies`): *"list meetings in the past 24 hours"* and *"fetch meeting transcript by recording ID"*.
  - For folder/DB providers (`notion`, `googledrive`): *"list files in folder"* (scoped to `meetings.location`) and *"read file content by ID"*.
  
  Confirm the response returns at least one action slug per query, scoped to the picked provider. Don't hardcode the names — read whatever slugs the discovery layer returns.
- **Direct vendor MCPs (Pattern B).** Confirm the host's loaded tool list contains at least one tool whose name or description matches the list-action intent and one matching the fetch-action intent for the picked toolkit's prefix.

If the probe returns no matching actions for either query, surface this to the user before continuing:

> "Looks like [toolkit] is connected, but it doesn't have the access the sweep needs — specifically, it can't [list your meetings / list files in [location]] or [pull a transcript / read file contents]. Want to pick a different meetings source, or sign in to [toolkit] again to check that read access is turned on?"

If the user picks "different source," loop back to the **First ask** at the top of 4c. If "re-auth," send them back to Step 2a (Composio install walkthrough scoped to that one app) or Step 2b (Own-MCPs verification scoped to meetings).

If the probe succeeds, capture nothing — the sweep re-runs the same discovery at run time. The probe is just a setup-time sanity check to fail fast.

#### 4d. Task system (only if tasks picked)

Determine provider from the connected toolkit (`notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday`, or anything else the host has wired). If multiple task toolkits are connected, ask which one to use.

**Provider-agnostic introspection flow.** Different task tools model tasks differently — Notion uses databases with arbitrary properties; Asana uses projects with a fixed `assignee` field; Linear uses teams with a fixed assignee; ClickUp / Monday have lists/boards with custom fields; Todoist is mostly flat. Instead of branching on provider, the skill introspects the chosen toolkit at runtime and walks the user through what it finds. The skill is responsible for handling the ambiguity — it learns the tool's shape from the tool itself and uses the user as the tiebreaker on anything ambiguous.

Load and show the **Tasks — container + assignee discovery** section from `references/canonical-messages.md`, then run the four substeps:

1. **Container discovery.** Most task tools group tasks into a container concept — projects, boards, lists, databases (name varies by provider). The sweep needs to know which container(s) to query.
   - Find the toolkit's "list containers" capability. For Composio, search the chosen toolkit's available meta-tools for a name matching the pattern `<TOOLKIT>_LIST_*S`, `<TOOLKIT>_SEARCH_*`, `<TOOLKIT>_GET_*S` (common shapes: Notion → `NOTION_SEARCH_NOTION_PAGE` with `filter_value="database"`; Asana → `ASANA_LIST_PROJECTS`; Linear → `LINEAR_LIST_TEAMS`; ClickUp → `CLICKUP_GET_LISTS`; Trello → `TRELLO_GET_BOARDS`; Monday → `MONDAY_LIST_BOARDS`; Todoist → `TODOIST_GET_PROJECTS`). For direct vendor MCPs, use whatever list-containers tool the vendor exposes. If you cannot identify a list-containers tool, ask the user what their tool's container concept is and how to enumerate them.
   - Ask the user where their tasks live, by name. Resolve each named container against the list-containers response, present the matches, and confirm before saving.
   - If the provider has no container concept (flat task list — Todoist's inbox, for example), skip this substep and save `tasks.container_ids: ["all"]` as a sentinel.

   Save as `tasks.container_ids: [<id>, ...]` and `tasks.container_names: [<name>, ...]` (parallel arrays — names are kept for human-readability in the sweep report header).

2. **Assignee-property discovery (introspect-then-ask).** For one saved container, fetch its schema/structure to learn what fields tasks in this tool can have.
   - Find the toolkit's "fetch container schema" capability. For Composio: typically `<TOOLKIT>_FETCH_*` or `<TOOLKIT>_GET_*_DETAILS` (Notion → `NOTION_FETCH_DATABASE`; Asana → fetch-project; Linear → fetch-team; etc.). For direct vendor MCPs: equivalent fetch-schema call. If the toolkit has no schema-fetch concept (some tools have a fixed task shape — e.g., Todoist tasks always have `responsible_uid`), skip the introspection and treat the fixed shape as a one-candidate result.
   - Walk the schema and build a candidate list of fields that could label assignment. Heuristics (apply all):
     - Type-based: typed user-reference fields (Notion `people`, Asana `assignee`, Linear `assignee`, ClickUp `assignees`); categorical types (`select`, `multi_select`, `status`); relation/link fields pointing to a teammates container; free-form text fields.
     - Name-based: field names matching `Assign`, `Assignee`, `Owner`, `Assigned to`, `Responsible` (case-insensitive) score higher.
   - Branch on candidate count:
     - **Zero candidates** → list ALL fields and ask the user which one labels assignment: *"I couldn't auto-detect an assignment field in `[container]`. Here's the full list of fields: [list with types]. Which one labels tasks assigned to you?"*
     - **Exactly one candidate** → take it but tell the user, so they can correct: *"In `[container]`, I'll filter on `[field]` (a `[type]` field). Looks like the assignment field — say if I should pick a different one."*
     - **Two or more candidates** → load the substep-2 prompt from canonical-messages and let the user pick.

   Save `tasks.assignee_property` (the field name as the toolkit returns it) and `tasks.assignee_property_type` (the field's type as the toolkit reports it — normalize to one of: `people`, `select`, `multi_select`, `status`, `relation`, `rich_text`, `text`, `enum`, `unknown`).

3. **Value capture.** Ask the type-appropriate question per the canonical-messages substep 3 lookup:
   - **User-reference types** (`people`, `assignee`-style) → *"Paste the email or user ID `[provider]` uses to label your tasks (likely your work email)."* If the toolkit needs a user ID rather than an email, resolve via the toolkit's user-search call after the user pastes their email.
   - **Categorical types with known options** (`select`, `multi_select`, `status`, custom dropdown) → list the option values from the schema response and ask the user to pick. Don't make them guess.
   - **Relation type** → ask the user for their row name in the linked container, then resolve to the linked-row UUID via the linked container's search call.
   - **Free-form text** → ask the user to paste the exact string used for their tasks.

   Save:
   - `tasks.assignee_value: <string>` for user-reference / select / status / text / relation
   - `tasks.assignee_value: [<string>, ...]` for `multi_select`

4. **Verification (silent on success, ask on zero-result).** Make one test query against the saved containers with the captured filter, bounded to the trailing 14 days. Use the toolkit's "query container" or "list tasks with filter" capability — Composio: `<TOOLKIT>_QUERY_*` / `<TOOLKIT>_LIST_*_WITH_FILTER`. Show no output on success — the count goes into the saved running state for the next setup refresh. On zero-result: *"I queried `[container(s)]` for tasks where `[field]` = `[value]` in the last 14 days and got 0 results. That might be right (slow period) or wrong (e.g., the value doesn't actually label your tasks). Want me to try a different value, or save as-is?"* Don't loop more than once — save with the user's call.

If the user has tasks in more than one container AND the schemas differ across containers (different field names or types), repeat substeps 2–4 per container and save a per-container block instead. The single-shared-property case is common; only branch into per-container shape when introspection shows the schemas actually differ.

---

### 5. User identity capture (silent where possible)

Used to: match speakers in meeting transcripts (find user-attributed commitments), map "from:me" / "I" pronouns when scanning text, filter task assignment values where the value uses the user's email.

Load and show the **User identity capture** section from `references/canonical-messages.md` for the user-facing prompt wording. Capture in this order, asking only when ambiguous:

1. **`primary_email`** — derive from the connected email account at Step 4a. If the user has multiple email accounts and picked all-merged, ask which one is primary.
2. **`slack_user_id`** — derive from the messaging connection. Composio Slack exposes the user's ID via `auth.test` equivalent (`SLACK_USER_INFO` or similar — check the toolkit). Capture silently. Skip if messaging not picked.
3. **`name`** — pull from the user's profile (Gmail profile name, Slack display name, whichever is available). Confirm once with the user (use the Name confirmation prompt from the canonical-messages section above).
4. **`meeting_attendee_emails`** — defaults to `[primary_email]`. Ask once (use the Email aliases prompt from the canonical-messages section).

Save under `user_identity`. Example:

```yaml
user_identity:
  name: "User Name"
  primary_email: placeholder@example.com
  slack_user_id: U0XXXXXXXX
  meeting_attendee_emails:
    - placeholder@example.com
```

If the messaging source isn't picked, omit `slack_user_id`. If meetings isn't picked, omit `meeting_attendee_emails`. Keep `name` and `primary_email` always.

---

### 6. Voice profile capture

Captures the user's actual writing voice per channel so drafts in the daily report sound like the user, not AI. Voice capture is bundled into setup rather than a separate skill — every user who runs this plugin needs it once.

Load and show the **Voice profile capture** section from `references/canonical-messages.md` for the user-facing prompt wording (pre-capture explanation, post-capture confirmation, redo flow).

**Pre-condition:** at least one of {email, messaging} must be wired AND picked at Step 3. If neither is picked, skip this step and write a `[VOICE PROFILE NOT CAPTURED]` banner to the profile file (the work skill will surface a runtime banner reminding the user to re-run setup).

**Email voice capture** (only if email picked):

1. Read the user's last ~20 sent emails: `from:me` filter, no other constraint, sorted by recency. Use the email toolkit's "list/fetch messages" capability with `query: from:me` and `max_results: 20`.
2. Extract patterns:
   - Salutations / greetings (e.g., "Hi <name>", "Hey", "<Name>,", no salutation)
   - Sign-offs (e.g., "Best,", "Thanks,", "—<Name>", no signoff)
   - Sentence rhythm (short/snappy vs. paragraph-form)
   - Common phrases / openers / closers
   - Formality bucket (formal / casual / mixed)
   - Signature template if any (auto-appended block at the end)
3. Write `## Email voice` section to `<workspace>/client-profile/daily-followup-tracker-voice.md` with the extracted patterns as descriptive prose, NOT raw email content.

**Slack voice capture** (only if messaging picked):

1. Use `SLACK_SEARCH_MESSAGES` with `from:me after:<7 days ago>` (or equivalent for the connected messaging tool) — capped at 50 messages.
2. Extract patterns specific to Slack:
   - Lowercase tendency
   - Emoji frequency + which emojis
   - Average message length
   - Thread-reply vs. new-message tendency
   - Common conversational openers ("hey", "hi", "yo", or none)
   - Punctuation level (full stops, single sentences)
3. Write `## Slack voice` section to the same profile file.

**No voice content captured verbatim** — the profile is a description of patterns, not a copy of the user's messages. The point is for the work skill's drafting agents to read patterns and apply them, not to mimic specific messages.

**Confirm with user once after capture:** show a 2–3 sentence summary of each captured voice section using the post-capture confirmation prompts from the canonical-messages section. User can say "looks good" or "redo".

**On redo:** read more messages (40 sent emails / 100 Slack messages) and try again. If still off, save what was captured with a `[NEEDS REFINEMENT]` flag and point the user at the file path so they can hand-edit (use the redo-fallback prompt from the canonical-messages section).

Save path under `voice_profile_path: client-profile/daily-followup-tracker-voice.md` (default; configurable to absolute path if user prefers).

---

### 7. Meeting follow-up default channel

Only ask if BOTH `meetings:` AND at least one of {email, messaging} are picked at Step 3.

Load and show the **Meeting follow-up default channel** section from `references/canonical-messages.md` for the prompt wording.

Save as `meeting_followup_default_channel: email` (or `slack`). Drives which voice profile section the meeting sub-agent uses when drafting follow-ups for meeting commitments.

If only one of email/messaging is wired at Step 3, skip the question and default `meeting_followup_default_channel` to that one silently.

If meetings isn't picked at Step 3, omit the field entirely.

---

### 8. Notification channel + recipient

Load and show the **Notification channel + recipient** section from `references/canonical-messages.md`. Two prompts in sequence:

1. **Channel choice** — `gmail | outlook | slackbot | none`. **Slack OAuth (`slack` toolkit) is NOT a notification option** — sending via the user's OAuth posts as the user, which delivers no notification ping. The Slackbot toolkit is the right tool for Slack notifications because it posts as a bot and the user receives a real inbound message. The work skill always writes the report file locally regardless of this setting; the notification is just the heads-up.
2. **Recipient** (only if channel ≠ `none`) — email address (gmail/outlook) OR Slack channel/DM ID (slackbot).

Save as `notification_channel` (`gmail` / `outlook` / `slackbot` / `none`) and `notification_target` (email address or Slack channel/DM ID; required only if channel ≠ `none`).

**Multi-account check.** After channel choice, call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]` for the chosen notification toolkit. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1`, load and show the **Notification account picker** section from `references/canonical-messages.md`. The user picks one account ID. Save as `notification_account_id`.

If `accounts.length == 1`, capture silently — no prompt — so subsequent runs are deterministic. If `notification_channel` is `none`, skip the picker entirely.

---

### 9. Output folder

Load and show the **Output folder** section from `references/canonical-messages.md`. Default `followups/` (workspace-relative — keeps the report path clickable straight from chat once the sweep finishes). Confirm the path. The folder gets created on first sweep run if it doesn't exist.

The path is configurable — users who want a single cross-project library can override to `~/Documents/Atlas/Followups/` or any absolute path. The canonical message surfaces the trade-off (chat-link previews work for workspace-relative paths only).

Save as `output_folder`.

---

### 10. Write `client-profile/daily-followup-tracker.local.md`

Write the captured running state to `<workspace>/client-profile/daily-followup-tracker.local.md`. Format (YAML frontmatter only, no markdown body):

```yaml
---
# Voice profile location — written by setup, read by sweep when generating drafts
voice_profile_path: client-profile/daily-followup-tracker-voice.md

# User identity — captured at setup. Used to:
#   - match speakers in meeting transcripts (find user-attributed commitments)
#   - map "from:me" / "I" pronouns when scanning text
#   - filter task assignment values where the value uses the user's email
user_identity:
  name: "User Name"                       # used for transcript speaker labels
  primary_email: placeholder@example.com  # primary; matches the email account if connected
  slack_user_id: U0XXXXXXXX               # only if messaging is wired
  # Per-tool aliases captured silently at detection time:
  meeting_attendee_emails:
    - placeholder@example.com             # all email addresses the user appears as in transcripts

# Default channel for meeting follow-up drafts
# Asked at setup (one-time choice). Drives which voice profile section the meeting sub-agent uses.
meeting_followup_default_channel: email   # or: slack

# Wired sources — at least one required. Each block present only if picked at step 1.

email:
  provider: gmail                         # or: outlook
  accounts:
    - gmail:placeholder@example.com

messaging:
  provider: slack
  scope: both                             # channels | dms | both — what the messaging-scan agent searches
  channels:                               # required if scope is channels or both; empty list if scope is dms
    - id: C0123ABCD
      name: "#exec-team"

meetings:
  provider: fathom                        # or: granola, fireflies, notion, googledrive
  # location: "Notion DB: Meeting Transcripts"   # only if provider is notion / googledrive

tasks:
  provider: notion                        # or: asana, linear, clickup, todoist, trello, monday
  container_ids:
    - 00000000-0000-0000-0000-000000000000
  container_names:
    - "My Tasks DB"
  assignee_property: Assignee
  assignee_property_type: people          # people | select | multi_select | status | relation | rich_text | text | enum | unknown
  assignee_value: placeholder@example.com

# Task source filter knobs (overrides; safe defaults if absent)
task_due_within_days: 2                   # default — surface tasks due today + N days
task_stale_after_days: 5                  # default — surface tasks with no update for X+ days

# Notification — `gmail | outlook | slackbot | none`. `slack` is a data source, not a notification target.
notification_channel: slackbot
notification_target: U0XXXX_OR_C0XXXX     # email address (gmail/outlook) OR Slack channel/DM ID (slackbot)
notification_account_id: slackbot_main    # only if multi-account picker fired

# Output
output_folder: followups/
dry_run: false
---
```

Substitute the actual captured values. **Each source block (`email`, `messaging`, `meetings`, `tasks`) is omitted entirely if not picked at Step 3.** The work skill reads "is this key present?" to decide whether to fetch from that source.

After writing, delete `<workspace>/.daily-followup-tracker-onboarding-in-progress.json` if it exists.

---

### 11. Hand-off

Load and show the **Hand-off** section from `references/canonical-messages.md`.

## Refresh mode (alternative entry from Step 0)

If config already exists at `client-profile/daily-followup-tracker.local.md` AND at least one source is connected AND notification is connected, skip Steps 1–2 (no Composio install walk-through) and run refresh mode instead.

For each field in the existing config, show the current value and ask: *"Keep this, or change?"* Accept the answer per-field. If the user says change, ask the corresponding question for that field and run the relevant logic (re-pick provider, re-verify accounts, re-capture voice, change channel or recipient, change output folder).

Refresh DOES re-verify capabilities are still connected by re-detecting the host's loaded tools (an MCP could have been disconnected since last setup).

After the walk-through, fall through to Step 10 (write config — overwriting with updated values).

## Output

A `<workspace>/client-profile/daily-followup-tracker.local.md` file with the user's source mix and wiring choices, plus a `<workspace>/client-profile/daily-followup-tracker-voice.md` file with per-channel voice patterns. The work skill (`daily-followup-tracker`) reads both at the start of each run.

In refresh mode, prefix the Hand-off message with "Updated."

## Customization

To change the channel + recipient or output folder: re-run `setup` (refresh mode walks through changes per-field).

To re-capture voice profiles (e.g., your communication style has shifted): re-run `setup` and pick "yes, re-capture voice" at Step 6.

To wire (or change) the recurring trigger: the sweep's daily cadence is encoded in its design; recurrence registration lives in your AI host's scheduler (Claude Code `/schedule`, GitHub Actions cron, OS cron, or any scheduler your host exposes). The Hand-off message at the end of `setup` covers the available options.

To change the wiring (e.g., swap from Composio Gmail to a direct Gmail MCP, or add a second email account): connect the new tool in your AI host, then re-run `setup` — detection picks up the new tool.

To change the wizard wording: edit `references/canonical-messages.md`.

To tune the task source's freshness knobs (`task_due_within_days`, `task_stale_after_days`) — defaults are 2 and 5 — edit them directly in `client-profile/daily-followup-tracker.local.md`. No setup question; the defaults work for most users until they don't.
