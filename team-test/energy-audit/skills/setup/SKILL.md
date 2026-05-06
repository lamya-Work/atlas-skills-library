---
name: setup
description: One-time onboarding wizard for the energy-audit plugin. Detects whether the host's AI has the capabilities the audit needs (calendar read / task read / meetings read / Slack read / notification email send) connected. Walks the user through Composio setup if any are missing. Captures which sources to use, which Slack channels to read, where meeting transcripts live, the notification email channel + recipient, and the output folder for reports. Resumable mid-flow. Re-runnable to refresh any field.
when_to_use: Run before using `extract-energy-profile` or `energy-audit` for the first time. Trigger phrases — "set up energy audit", "configure energy audit", "wire energy audit setup", "refresh energy audit setup", "redo energy audit wiring". Auto-trigger when `energy-audit` reports no config exists at `.claude/energy-audit.local.md`.
atlas_methodology: neutral
---

# setup

Onboarding wizard for `energy-audit`. Detection-first: figures out what the user already has connected and only walks them through what's missing. Three branches: refresh (config exists), fast (capabilities connected, no config), full onboarding (capabilities missing).

## Purpose

The work skills (`extract-energy-profile` and `energy-audit`) need at least one of four data-source capabilities connected — calendar read, task tool read, meetings read, or Slack read — plus a notification-email path (Gmail or Outlook) for the audit-complete heads-up. This skill captures which sources to use, the per-source details (Slack channel list, meetings location, calendar accounts), the notification channel + recipient, and the output folder for reports. Saves to `.claude/energy-audit.local.md`. Resumable. Re-runnable.

The audit's bi-weekly cadence is encoded in its own design (14-day window). This skill does NOT capture a schedule anchor — recurring runs are wired in whichever scheduler the user's AI host provides (Cowork's scheduler, Claude Code's `/schedule`, GitHub Actions, OS cron, etc.). The Hand-off message surfaces this as a recommendation.

## Inputs

- **Existing config** (optional) — if `.claude/energy-audit.local.md` exists, the skill enters refresh mode automatically.
- **Host's loaded tool list** — required. Skill reads what tools the host has connected and uses that to detect capability presence.
- **In-progress marker** (optional) — `.energy-audit-onboarding-in-progress.json` in the workspace, if a previous run was interrupted.
- **Conversational interview** — questions asked one at a time, paraphrased back for confirmation.

## Required capabilities

(Abstract list — no specific tool names. Validated default wiring is Composio MCP; see plugin README §4 for alternatives.)

- Detection of available calendar / task / meetings / Slack / notification-email tools in the host's loaded tool list
- File read + write (for the local config and in-progress marker)
- Conversational interview (one question at a time)

## Steps

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The local config (`.claude/energy-audit.local.md`) and in-progress marker (`.energy-audit-onboarding-in-progress.json`) resolve relative to this directory. Do NOT write these into the plugin's own install directory. Plugin-internal references (templates, methodology files, `references/canonical-messages.md`) use `../../` from this skill folder to mean plugin root.
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
>    **2c. Per-toolkit account detail.** When a per-source step (Step 4a calendar, Step 4b task tool's container/schema fetches, Step 5 notification) needs the toolkit's connected accounts (IDs, aliases, statuses), call `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]`. The response returns `accounts[]` for that toolkit. The call accepts an array, so multiple toolkits can be batched into one round-trip when more than one is needed at the same step.
>
>    **Fallback if the annotation is missing.** If the host stripped the description annotation during tool ingestion (some hosts only forward the schema, not the human-readable description), fall back to a single `COMPOSIO_MANAGE_CONNECTIONS` call with the full slug map as the `toolkits` array (each entry `{name: <slug>, action: "list"}`). For each slug, a non-empty `accounts[]` means connected. Slower than the annotation read but doesn't depend on description forwarding.
>
>    **Response sizes** are small for both the annotation read (a one-line text snippet in the tool description) and `COMPOSIO_MANAGE_CONNECTIONS` action `list` (account metadata only — IDs, aliases, statuses). No out-of-band file overflow handling needed — that was a `COMPOSIO_SEARCH_TOOLS` problem, and detection no longer calls `SEARCH_TOOLS`.
>
>    Slug map for this plugin:
>    - `googlecalendar`, `outlook` → calendar
>    - `notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday` → tasks
>    - `fathom`, `granola`, `fireflies` → meetings (direct)
>    - `notion`, `googledrive` → meetings (transcript folder fallback)
>    - `slack` → Slack
>    - `gmail`, `outlook` → notification email
>    - Other slugs → ignore
>
>    For other aggregator MCPs (Zapier, etc.), use their equivalent enumeration / list-connections function.
>
> If stage 2 returns no toolkits (annotation absent AND fallback `MANAGE_CONNECTIONS` call returns empty `accounts[]` for every slug in the map), treat all capabilities behind that aggregator as NOT connected.
>
> **Pattern B — Direct vendor MCPs.** One or more tools per service, with the vendor name in the prefix:
> - **Calendar read** — `googlecalendar`, `gcal`, `outlook`, `calendar`
> - **Task read** — `notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday`
> - **Meetings read** — `fathom`, `granola`, `fireflies`, `notion`, `googledrive`
> - **Slack read** — `slack`
> - **Notification email** — `gmail`, `outlook`
>
> A capability counts as connected only if at least one matching tool is BOTH discoverable AND callable. Auth-bootstrap tools (`*_authenticate` with no companion fetch/send tool) do NOT count.
>
> **Both patterns can coexist.** Run Pattern A first, then B; merge results.

### 0. Detection (always first)

**Resolve `.claude/energy-audit.local.md` against the workspace (user's CWD), NOT the plugin install directory. If the path doesn't exist in the workspace, do NOT fall back to checking the plugin's own folder.**

Three checks:

1. **Existing config** at `<workspace>/.claude/energy-audit.local.md`.
2. **Capabilities** in the host's loaded tool list, per the Pattern A + B detection above.
3. **In-progress marker** `<workspace>/.energy-audit-onboarding-in-progress.json`.

Branch:

| Condition | Branch |
|---|---|
| Config exists + at least one source connected + notification connected | **Refresh mode** (jump to "Refresh mode" section below) |
| Capabilities connected, no config | **Fast path** (load and show the **Fast-path acknowledgement** section from `references/canonical-messages.md` — substituting the readable toolkit names from Stage 2's slug map for `[list of connected toolkits]` — then skip to Step 3) |
| In-progress marker found | **Resumed session** (load marker, skip questions already answered, continue from where left off) |
| Capabilities missing | **Full onboarding** (continue to Step 1) |

**Resumability rule** — for Steps 1–6, save running state to `<workspace>/.energy-audit-onboarding-in-progress.json` after every captured answer.

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

- **Calendar** — call the calendar tool with a "list 1 event for today" / "fetch next event" request.
- **Task tool** — call with "list 1 task assigned to the user" or "list tasks limit=1".
- **Meetings (direct)** — call "list 1 transcript" or equivalent.
- **Slack** — call "list channels" or "auth test" (low-impact, no message sent).
- **Notification email** —
  - Gmail → "list labels" or "list 1 message"
  - Outlook → "list folders" or "list 1 message"

Show the user the result of each test. For any failed verification, deliver the **Capabilities not wired — pre-flight** section from `references/canonical-messages.md` scoped to the failing capability and offer to walk through Composio install for that one app.

Iterate until each capability the user wants is connected AND verified. If the user only wants a subset of the four sources, only those need to pass.

---

### 3. Source picker

Load and show the **Source picker** section from `references/canonical-messages.md`. Filter the displayed options to capabilities that have at least one connected tool — if the user wants a source they haven't connected, briefly offer to connect it (loop back to Step 2 scoped to that one source).

User picks a subset of {calendar, task tool, meetings, Slack}. **At least one required.** If the user picks none, re-ask once. Save the selection list to running state.

---

### 4. Per-source detail capture

Sub-steps fire only for sources picked at Step 3.

#### 4a. Calendar (only if calendar picked)

Determine provider (Google Calendar / Outlook) — usually inferable from the connected toolkit name (`googlecalendar` → `googlecalendar`; `outlook` → `outlook`). If both providers have connected toolkits, ask which one to use.

**Multi-account check.** The detection mechanism varies by aggregator:

- **Composio (Pattern A).** Call Stage 2c (per-toolkit account detail) for the chosen calendar toolkit: `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]`. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1`, multi-account selection is needed.
- **Direct vendor MCPs (Pattern B).** Most direct-vendor MCPs are single-account by design. If a vendor MCP supports multi-account, follow its discovery convention. If unclear, treat as single-account.

If multiple accounts exist, **always ask** (never auto-merge):

> "I see [N] calendar accounts connected: [list — show alias if set, otherwise email address]. Use one, or scan all merged?"

Capture as a list under `calendar.accounts`, format `<provider>:<account-identifier>`. Examples:
- Single account: `["googlecalendar:placeholder@example.com"]`
- All merged: `["googlecalendar:placeholder@example.com", "googlecalendar:other@example.com"]`

If only one account exists, capture silently (no prompt) so subsequent runs are deterministic.

**Per-account calendar list.** A single account can host several calendars (the user's primary, secondary calendars, calendars they own on someone else's behalf, holiday calendars, etc.). Scanning all of them mixes irrelevant signal into the audit; scanning only the primary may miss meetings the user puts on a secondary calendar. Ask explicitly per account.

For each chosen account:

1. Call the available list-calendars function for the provider:
   - Google Calendar (Composio): `GOOGLECALENDAR_LIST_CALENDARS` with the account ID.
   - Outlook (Composio): the equivalent list-calendars meta-tool.
2. Load and show the **Calendar list per account** section from `references/canonical-messages.md`, substituting the response into `[N]`, `[primary]`, and `[list]`.
3. Capture the answer:
   - "Just primary" → save `["primary"]` for this account (the special string `primary` means "the account's primary calendar at audit time" so the audit doesn't have to re-resolve a UUID that may rotate).
   - "All merged" → save the full list of calendar IDs for this account.
   - "Specific ones" → resolve each pasted name or ID against the list-calendars response and save the list of IDs.
4. If only one calendar is visible for the account (rare but possible), capture silently as `["primary"]` — no prompt.

Save under `calendar.calendar_ids` keyed by account identifier. Example:

```yaml
calendar:
  provider: googlecalendar
  accounts:
    - googlecalendar:placeholder@example.com
  calendar_ids:
    googlecalendar:placeholder@example.com:
      - primary
      - secondary@group.calendar.google.com
```

#### 4b. Task tool (only if tasks picked)

Determine provider from the connected toolkit (`notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday`, or anything else the host has wired). If multiple task toolkits are connected, ask which one to use.

**Provider-agnostic introspection flow.** Different task tools model tasks differently — Notion uses databases with arbitrary properties; Asana uses projects with a fixed `assignee` field; Linear uses teams with a fixed assignee; ClickUp / Monday have lists/boards with custom fields; Todoist is mostly flat. Instead of branching on provider, the skill introspects the chosen toolkit at runtime and walks the user through what it finds. The skill is responsible for handling the ambiguity — it learns the tool's shape from the tool itself and uses the user as the tiebreaker on anything ambiguous.

Load and show the **Tasks — container + assignee discovery** section from `references/canonical-messages.md`, then run the four substeps:

1. **Container discovery.** Most task tools group tasks into a container concept — projects, boards, lists, databases (name varies by provider). The audit needs to know which container(s) to query.
   - Find the toolkit's "list containers" capability. For Composio, search the chosen toolkit's available meta-tools for a name matching the pattern `<TOOLKIT>_LIST_*S`, `<TOOLKIT>_SEARCH_*`, `<TOOLKIT>_GET_*S` (common shapes: Notion → `NOTION_SEARCH_NOTION_PAGE` with `filter_value="database"`; Asana → `ASANA_LIST_PROJECTS`; Linear → `LINEAR_LIST_TEAMS`; ClickUp → `CLICKUP_GET_LISTS`; Trello → `TRELLO_GET_BOARDS`; Monday → `MONDAY_LIST_BOARDS`; Todoist → `TODOIST_GET_PROJECTS`). For direct vendor MCPs, use whatever list-containers tool the vendor exposes. If you cannot identify a list-containers tool, ask the user what their tool's container concept is and how to enumerate them.
   - Ask the user where their tasks live, by name. Resolve each named container against the list-containers response, present the matches, and confirm before saving.
   - If the provider has no container concept (flat task list — Todoist's inbox, for example), skip this substep and save `tasks.container_ids: ["all"]` as a sentinel.

   Save as `tasks.container_ids: [<id>, ...]` and `tasks.container_names: [<name>, ...]` (parallel arrays — names are kept for human-readability in the audit report header).

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

#### 4c. Meetings (only if meetings picked)

Load and show the **Meetings source two-tier** section from `references/canonical-messages.md`.

**First ask** — direct integration check (Fathom / Granola / Fireflies). If yes:
- Confirm the toolkit is connected via re-detection.
- If not connected, loop back to Step 2 (Composio install walkthrough scoped to that one app).
- Save `meetings.provider: <fathom|granola|fireflies>`.

**Fallback ask** (only if first ask = no) — Notion DB or Drive folder. User names the location string (e.g., "Notion DB: Meeting Transcripts" or "Drive folder: /Meetings/2026"). Verify Notion or Google Drive is connected via detection; if not, offer Composio install scoped to that one app.
- Save `meetings.provider: <notion|googledrive>` and `meetings.location: "<the location string>"`.

**Neither path works** — record `meetings: skipped` in running state. The audit will skip meetings and note it in the report header. Move on.

**4c.4 Verify the picked meetings toolkit exposes the actions the audit needs.**

Skip this substep if 4c saved `meetings: skipped`. Otherwise probe once before continuing — a toolkit can be connected without exposing the actions the audit calls (toolkit slug authorized for a different scope, vendor renamed an action, etc.):

- **Composio (Pattern A).** Call the toolkit-discovery capability with two queries scoped to the picked `meetings.provider`. Pick the queries by provider class:
  - For direct meeting providers (`fathom`, `granola`, `fireflies`): *"list meetings in the past 14 days"* and *"fetch meeting transcript by recording ID"*.
  - For folder/DB providers (`notion`, `googledrive`): *"list files in folder"* (scoped to `meetings.location`) and *"read file content by ID"*.
  
  Confirm the response returns at least one action slug per query, scoped to the picked provider. Don't hardcode the names — read whatever slugs the discovery layer returns.
- **Direct vendor MCPs (Pattern B).** Confirm the host's loaded tool list contains at least one tool whose name or description matches the list-action intent and one matching the fetch-action intent for the picked toolkit's prefix.

If the probe returns no matching actions for either query, surface this to the user before continuing:

> "Looks like [toolkit] is connected, but it doesn't have the access the audit needs — specifically, it can't [list your meetings / list files in [location]] or [pull a transcript / read file contents]. Want to pick a different meetings source, or sign in to [toolkit] again to check that read access is turned on?"

If the user picks "different source," loop back to the **First ask** at the top of 4c. If "re-auth," send them back to Step 2a (Composio install walkthrough scoped to that one app) or Step 2b (Own-MCPs verification scoped to meetings).

If the probe succeeds, capture nothing — the audit re-runs the same discovery at run time. The probe is just a setup-time sanity check to fail fast.

#### 4d. Slack (only if Slack picked)

Load and show the **Slack channel name-and-verify** section from `references/canonical-messages.md`. Initial ask. User pastes a list of channel names (`#exec-team` style) and/or channel IDs (`C0123ABCD` style).

For each entry, attempt to resolve via the connected Slack tool's lookup function (handles both name and ID formats). If any miss, show the resolution-feedback message with the found / missed lists. Loop until all entries resolve.

Save the resolved list under `slack.channels`. Each entry has `id` and `name`. Example:
```yaml
slack:
  channels:
    - id: C0123ABCD
      name: "#exec-team"
```

---

### 5. Notification channel + recipient

Load and show the **Notification channel + recipient** section from `references/canonical-messages.md`. Two prompts in sequence:

1. Channel choice — Gmail or Outlook (or `none`). **Slack is NOT an option** — Slack is a data source, not a notification target. The audit always writes the report file locally regardless of this setting; the email is just the heads-up.
2. Recipient address (only if channel ≠ `none`).

Save as `notification_channel` (`gmail` / `outlook` / `none`) and `notification_target` (email address; required only if channel ≠ `none`).

**Multi-account check.** After channel choice, call Stage 2c (per-toolkit account detail) for the chosen notification toolkit: `COMPOSIO_MANAGE_CONNECTIONS` with `toolkits: [{name: <slug>, action: "list"}]`. The response's `accounts[]` lists IDs, aliases, statuses. If `accounts.length > 1` (e.g., three Gmail aliases connected to the same address with different OAuth scopes), load and show the **Notification account picker** section from `references/canonical-messages.md`. The user picks one account ID. Save as `notification_account_id` (e.g., `gmail_work-account`).

If `accounts.length == 1`, capture silently — no prompt — so subsequent runs are deterministic. If `notification_channel` is `none`, skip the picker entirely.

---

### 6. Output folder

Load and show the **Output folder** section from `references/canonical-messages.md`. Default `audits/` (workspace-relative — keeps the report path clickable straight from chat once the audit finishes). Confirm the path. The folder gets created on first audit run if it doesn't exist.

The path is configurable — users who want a single cross-project library can override to `~/Documents/Atlas/Energy-Audits/` or any absolute path. The canonical message surfaces the trade-off (chat-link previews work for workspace-relative paths only).

Save as `output_folder`.

---

### 7. Write `.claude/energy-audit.local.md`

Write the captured running state to `<workspace>/.claude/energy-audit.local.md`. Format (YAML frontmatter only, no markdown body):

````markdown
---
# Where the energy profile lives (read by extract-energy-profile + energy-audit)
profile_path: client-profile/exec-energy-profile.md

# Wired sources — at least one required. Each block present only if the source is picked at Step 3.
calendar:
  provider: googlecalendar              # or: outlook
  accounts:
    - googlecalendar:placeholder@example.com
  # Per-account calendar selection. Special string `primary` = "the account's primary calendar at audit time".
  # If a user has multiple calendars and picks specific ones, list the IDs here.
  calendar_ids:
    googlecalendar:placeholder@example.com:
      - primary

tasks:
  provider: notion                       # or: asana, linear, clickup, todoist, trello, monday, or anything else

  # Container IDs — projects, boards, lists, databases (whatever the provider's container concept is).
  # Captured by setup Step 4b's container-discovery substep. The literal string "all" = "no container concept;
  # query the toolkit's flat task list" (Todoist-style). Names array is parallel to IDs for human readability.
  container_ids:
    - 00000000-0000-0000-0000-000000000000
  container_names:
    - "My Tasks DB"

  # Assignee filter — captured by setup Step 4b's introspect-then-ask substeps. The audit builds a
  # type-appropriate filter from these three fields and queries each container.
  assignee_property: Assign              # the field name as the toolkit returns it
  assignee_property_type: select         # normalized type: people, select, multi_select, status, relation, rich_text, text, enum, unknown
  assignee_value: EA                     # string for most types; array for multi_select

  # Optional per-container override — only present if introspection found different schemas across containers.
  # When present, the top-level assignee_* fields are ignored for the listed container_id and the per-container
  # block applies instead.
  # per_container:
  #   <container-id>:
  #     assignee_property: ...
  #     assignee_property_type: ...
  #     assignee_value: ...

meetings:
  provider: fathom                       # or: granola, fireflies, notion, googledrive
  # If provider is notion or googledrive:
  location: "Notion DB: Meeting Transcripts"

slack:
  channels:
    - id: C0123ABCD
      name: "#exec-team"

# Notification — Gmail or Outlook only. Audit always writes the report file locally regardless.
notification_channel: gmail              # or: outlook | none
notification_target: placeholder@example.com   # required only if channel ≠ none
notification_account_id: gmail_work-account      # optional — only present if multi-account picker fired at Step 5

# Output folder — where audit reports save. Workspace-relative default keeps chat-link previews clickable.
output_folder: audits/

# Dry-run — when true, audit prints report path + notification body but doesn't write or send
dry_run: false
---
````

Substitute the actual captured values. **Each source block (`calendar`, `tasks`, `meetings`, `slack`) is omitted entirely if not picked at Step 3.** The audit reads "is this key present?" to decide whether to fetch from that source.

After writing, delete `<workspace>/.energy-audit-onboarding-in-progress.json` if it exists.

---

### 8. Hand-off

Load and show the **Hand-off** section from `references/canonical-messages.md`.

---

## Refresh mode (alternative entry from Step 0)

If config already exists at `.claude/energy-audit.local.md` AND at least one source is connected AND notification is connected, skip Steps 1–2 (no Composio install walk-through) and run refresh mode instead.

For each field in the existing config, show the current value and ask: *"Keep this, or change?"* Accept the answer per-field. If the user says change, ask the corresponding question for that field and run the relevant logic (re-pick provider, re-verify accounts, change channel or recipient, change output folder).

Refresh DOES re-verify capabilities are still connected by re-detecting the host's loaded tools (an MCP could have been disconnected since last setup).

After the walk-through, fall through to Step 7 (write config — overwriting with updated values).

## Output

A `<workspace>/.claude/energy-audit.local.md` file with the user's source mix and wiring choices. The work skills (`extract-energy-profile`, `energy-audit`) read this file at their start.

In refresh mode, prefix the Hand-off message with "Updated."

## Customization

To change the channel + recipient or output folder: re-run `setup` (refresh mode walks through changes per-field).

To wire (or change) the recurring trigger: the audit's bi-weekly cadence is encoded in its design; recurrence registration lives in your AI host's scheduler (Cowork, Claude Code `/schedule`, GitHub Actions cron, OS cron). The Hand-off message at the end of `setup` covers the available options.

To change the wiring (e.g., swap from Composio Google Calendar to a direct Google Calendar MCP, or add a second calendar account): connect the new tool in your AI host, then re-run `setup` — detection picks up the new tool.

To change the wizard wording: edit `references/canonical-messages.md`.
