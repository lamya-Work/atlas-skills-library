# Canonical user-facing messages — energy-audit setup

These messages are scoped to the four data sources energy-audit can read (calendar, task tool, meetings, Slack) plus notification (Gmail or Outlook only). 

The setup skill loads this reference and uses these messages verbatim at the points specified in `SKILL.md`. They were drafted to preserve a careful, non-jargony voice. Substitute `[bracketed]` placeholders at runtime; otherwise preserve the wording.

`[bracketed]` = substitute at runtime (names, paths, errors).

---

### Intro

Used at: `SKILL.md` Step 1, full-onboarding branch only.

> "This plugin reviews your last 14 days of work, finds repetitive operational tasks, and produces 1–2 ready-to-build automation plans.
>
> It can read from any combination of: your **calendar**, your **task tool**, your **meeting transcripts**, and **selected Slack channels** — pick whichever ones you actually use, at least one.
>
> The easiest way to connect those tools is **Composio** — a free service that connects your AI to your apps in one place. Walk through the setup once and you're done; takes about 5 minutes.
>
> If you'd rather connect your own tools (e.g., a direct Google Calendar tool, your own Slack tool, etc.), that works too — just let me know what you have and I'll check.
>
> Which would you like to do? (Default is Composio.)"

### Composio install walkthrough

Used at: `SKILL.md` Step 2a, when user picks the Composio path.

Direct the user (don't run shell commands for them):

> "Here's the Composio setup, in order:
>
> 1. Sign up at composio.dev (free).
> 2. Click **Connect Apps** in the left sidebar.
> 3. Connect the apps you want to use:
>    - **Calendar** (optional, for reading your meetings) — Google Calendar OR Outlook Calendar
>    - **Task tool** (optional, for reading your assigned tasks) — Notion, Asana, Linear, ClickUp, Todoist, Trello, or Monday
>    - **Meetings** (optional, for reading meeting transcripts) — Fathom, Granola, or Fireflies
>    - **Slack** (optional, for reading selected channels)
>    - **Email** (required, for the audit-complete ping) — Gmail OR Outlook
> 4. Click **Install** in the left sidebar.
> 5. Pick the install card for your AI (Claude Code, Claude Desktop, Codex, ChatGPT, etc.) and follow the steps on that page — they're tailored to your AI and OS. The install uses **OAuth (one browser-based 'authorize' click)**; no API keys to copy or paste.
>
> Where are you?
> - **(a) Done — installed and authenticated.** I'll re-check what's connected.
> - **(b) Got stuck on a specific step.** Tell me which one and what you saw.
> - **(c) Composio's dashboard looks different from what I described.** Send a screenshot."

### Composio re-auth (Stage 1.5)

Used at: `SKILL.md` Step 0 / detection callout, Stage 1.5 — when the Composio MCP server is installed in the AI but only the auth-bootstrap tools are loaded (no live meta-tools yet). Show this before triggering the OAuth flow so the user can tell why we're not walking the full install.

Lead with the status line so it's obvious nothing is being skipped silently:

> "Composio is already installed in your AI from a prior session — I just need to re-authorize it for this session, not walk through the full install. Here's the auth link:
>
> Composio's connected to your AI but needs re-authorizing for this session. Open the URL I'll send next in your browser, sign in, approve access, and tell me when you're done — your connected apps will light up automatically. If the redirect page fails to load (it usually does — that's just localhost), copy the full URL from your address bar and paste it back to me."

### Stuck help

Used at: `SKILL.md` Step 2a, when user picks (c) "Dashboard looks different" or otherwise needs visual help.

> "No worries. Send me a screenshot of what you see on your screen and tell me which device you're on (Mac / Windows / Linux). I'll walk you through it from there based on what you see."

### Capabilities not wired — pre-flight

Used at: `SKILL.md` Step 0 / Step 1, when detection finds capabilities are missing. Substitute `[bracketed]` placeholders with the specific missing capabilities at runtime.

Run detection silently first — only show this message if something is actually missing. If all capabilities are present, skip this section and proceed directly to Step 3 (Source picker).

> "Heads up — I don't see [a calendar tool / a task tool / a meetings tool / Slack / a notification email tool] connected to your AI right now. The audit can still run with just one of these connected, so you don't have to connect them all — but you'll need at least one. Want me to walk you through getting one connected?"

### Fast-path acknowledgement

Used at: `SKILL.md` Step 0 Branch table, when detection lands on the **Fast path** branch (capabilities connected, no config). Show this one-liner immediately before jumping to Step 3 so the user can tell that detection ran and decided to skip the install walk-through deliberately.

Substitute `[list of connected toolkits]` with the readable names from the slug map (e.g., "Google Calendar, Notion, Fathom, Gmail").

> "You already have [list of connected toolkits] connected through Composio. Skipping the install walk-through; jumping to the source picker."

### Source picker

Used at: `SKILL.md` Step 3. After the user responds, save their selections to the local config before continuing to Step 4. If the user picks nothing, re-ask once — don't proceed without at least one source selected.

> "Got it. Now — which of these do you want me to read for the audit? Pick whichever ones you actually use:
>
> - Calendar
> - Task tool
> - Meeting transcripts
> - Slack channels
>
> Pick any combination, but at least one. The audit will skip whatever you don't connect."

### Calendar list per account

Used at: `SKILL.md` Step 4a, after the multi-account choice is settled (or silently for a single-account user). Show one prompt per chosen account so each account's scan scope is set explicitly. Default option `primary` keeps the audit narrow if the user doesn't want to think about it.

Substitute `[account]` with the chosen account's identifier or alias and `[N]`, `[primary]`, and `[list]` from the list-calendars response.

> "Inside [account], I see [N] calendars: [primary], [list]. Which should I scan for the audit? Default: just your primary.
>
> You can pick: just primary, all of them merged, or specific ones (paste names or IDs)."

If the user picks "specific ones," accept either calendar names or IDs and resolve names to IDs via the same list-calendars response.

### Tasks — container + assignee discovery

Used at: `SKILL.md` Step 4b. Provider-agnostic — the SKILL introspects the chosen task toolkit at runtime to learn its shape, then uses these prompts to walk the user through what was found. Substitute `[provider]`, `[container concept]`, `[container name]`, `[field]`, `[type]`, `[options]` from the introspection results.

`[container concept]` = whatever the provider calls its task container — *project* (Asana, Todoist), *board* (Trello, Monday), *list* (ClickUp), *database* (Notion), *team* (Linear). If unsure, use the generic word *container*.

Four substeps: container discovery → assignee-property discovery → value capture → silent verification.

**Substep 1 — container ask.**

> "Where do your tasks live in [provider]? Most task tools group them into [container concept]s — a project tracker, a board, a list, a database, etc. Tell me by name and I'll resolve. You can name more than one."

After the user names them, resolve via the toolkit's list-containers tool, present the matches, and confirm before saving. If the provider has no container concept (a flat task list), skip this prompt and tell the user: *"`[provider]` doesn't group tasks into containers — I'll query your full task list."*

**Substep 2 — assignee-property ask** (depends on what introspection found).

- **Auto-detected exactly one plausible field** — tell the user, don't ask:
  > "In `[container name]`, I'll filter on `[field]` (a `[type]` field). Looks like the assignment field — say if I should pick a different one."
- **Auto-detected zero plausible fields** — show all fields and ask:
  > "I couldn't auto-detect an assignment field in `[container name]`. Here's everything in the schema: [list of field names + types]. Which one labels tasks assigned to you?"
- **Auto-detected two or more plausible fields** — show the candidates and ask:
  > "In `[container name]`, more than one field could label assignment: [list of candidate field names + their types]. Which one labels yours?"

**Substep 3 — value ask** (phrasing depends on the detected property type).

- **User-reference types** (`people`, fixed `assignee` fields) → *"Paste the email or user ID `[provider]` uses to label your tasks (likely your work email)."*
- **Categorical types with known options** (`select` / `multi_select` / `status` / custom dropdowns) → *"Your `[field]` field has options `[list of option values]`. Which one labels your tasks?"*
- **Relation type** → *"Your `[field]` field links to a `[linked container name]`. Tell me your row name there and I'll look up the id."*
- **Free-form text** → *"Your `[field]` field is free-form. Paste the exact string used for your tasks."*

After capture, confirm back: *"Got it — I'll filter the audit's task pull on `[field]` = `[value]` in `[container name(s)]`."*

**Substep 4 — verification (silent on success).**

After the value is captured, run one test query against the saved containers with the captured filter, bounded to the trailing 14 days. On success, save with no extra prompt. On zero-result, ask once:

> "I queried `[container name(s)]` for tasks where `[field]` = `[value]` in the last 14 days and got 0 results. That might be right (slow period) or wrong (e.g., the value doesn't actually label your tasks). Want me to try a different value, or save as-is?"

Don't loop more than once — accept the user's call.

### Notification account picker

Used at: `SKILL.md` Step 5, only when the chosen notification toolkit has more than one account connected (e.g., three Gmail aliases). Skip this prompt when the toolkit is single-account.

Substitute `[N]`, `[Gmail/Outlook]`, and the account list (use account IDs and aliases — both are useful when multiple aliases share an email address).

> "I see [N] [Gmail/Outlook] accounts connected to your AI: [account-id-1] (alias: `[alias-or-none]`), [account-id-2] (alias: `[alias-or-none]`), … . Which one should I send the audit-complete heads-up FROM? (If you're not sure, pick the one tied to the email scope that can send messages — usually the work-full-scope alias rather than a filters-only one.)"

Confirmation:

> "Got it — heads-up will send from `[chosen-account-id]`."

### Slack channel name-and-verify

Used at: `SKILL.md` Step 4d. Only show this section if the user selected "Slack channels" in the Source picker. Accept both `#name` format and raw channel IDs. Save the verified list (found channels only) to the local config.

Initial ask:

> "Which Slack channels do you want me to look at? Just list them — names like `#exec-team` work, channel IDs (`C0123ABCD`) also work. Paste as many as you want."

Resolution feedback (when some channels miss):

> "Found [N] of [M] channels: [list-of-found]. Couldn't find [list-of-missed] — check the spelling, or paste the channel ID and I'll look it up. (You can find a channel ID by right-clicking the channel name in Slack → 'View channel details' → it's at the bottom of the panel.)"

### Meetings source two-tier

Used at: `SKILL.md` Step 4c. Only show this section if the user selected "Meeting transcripts" in the Source picker. Ask the two questions in sequence — first check for a direct integration, then fall back to a Notion/Drive path if needed.

First ask:

> "For meetings, Composio connects directly to Fathom, Granola, and Fireflies. Do you use one of those?"

Fallback ask (if no):

> "Got it. Do you have a place where transcripts land — like a Notion database or a Drive folder I could read from? (You'll need Notion or Google Drive connected in Composio for me to read it; we can do that now if you don't.)"

### Notification channel + recipient

Used at: `SKILL.md` Step 5. Ask these two prompts in sequence — do not combine them into one message.

Note: Slack is **not** a notification target. Only Gmail and Outlook are supported. If the user asks about sending the ping somewhere else, acknowledge it and explain the audit always writes the report locally — email is the supported heads-up channel.

First ask (channel choice):

> "Where should I send a heads-up email when the audit finishes? Two options that work reliably for self-pings: Gmail or Outlook. (The audit always saves the report file locally to your output folder — the email is just the heads-up.)"

Second ask (address):

> "What email address should I send it to?"

Confirmation (after user provides both):

> "Got it — I'll send the audit-complete ping to [address] via [Gmail / Outlook]."

### Output folder

Used at: `SKILL.md` Step 6.

> "Where should audit reports save? Default: `audits/` inside this project — keeps the report path clickable straight from chat once the audit finishes. You can pick somewhere else if you'd like. I'll create the folder if it doesn't exist.
>
> If you'd rather keep audit reports in a single place across all your projects, pick a path like `~/Documents/Atlas/Energy-Audits/` instead — it'll just mean the audit-complete email link won't open directly from chat."

Confirmation (after user confirms or picks a path):

> "Got it — reports will save to [resolved-path]. I'll create the folder automatically on first run."

### Hand-off

Used at: `SKILL.md` final step. This is the final message in the setup skill. After delivering it, the setup skill is done — do not proceed into the audit run unless the user explicitly asks.

> "Setup done. Saved to `.claude/energy-audit.local.md`.
>
> The audit is designed to run on a bi-weekly cadence — if you want it recurring, schedule it in whichever AI tool you're using (Cowork's scheduler, Claude Code's `/schedule`, GitHub Actions, OS cron — all work). If you'd rather just trigger it manually, that's fine — say *'run my energy audit'* any time.
>
> Next, run `extract-energy-profile` to capture how you work — Zone of Genius, drains, recurring patterns, quarterly priorities. The audit reads both files when it runs."

---

*These messages were adapted from the ideal-week-ops setup canonical-messages reference. Drift is fine over time — but the starting point lives here so the careful tone work isn't lost on first implementation.*
