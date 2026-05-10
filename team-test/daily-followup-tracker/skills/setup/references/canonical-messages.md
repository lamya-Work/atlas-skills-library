# Canonical messages — daily-followup-tracker setup

These messages are scoped to the four data sources the daily-followup-tracker can read (email, messaging, meeting transcripts, task system) plus notification (Gmail, Outlook, or Slackbot).

The setup skill loads this reference and uses these messages verbatim at the points specified in `SKILL.md`. They were drafted to preserve a careful, non-jargony voice. Substitute `[bracketed]` placeholders at runtime; otherwise preserve the wording.

`[bracketed]` = substitute at runtime (names, paths, errors).

---

## Intro

Used at: `SKILL.md` Step 1, full-onboarding branch only.

> "This plugin runs a daily sweep of your last 24 hours — checking your email, messages, meetings, and task list — and produces a report showing what needs a follow-up, with draft replies ready to copy and send.
>
> It can read from any combination of: your **email**, your **messaging** (Slack), your **meeting transcripts**, and your **task list** — pick whichever ones you actually use, at least one.
>
> The easiest way to connect those tools is **Composio** — a free service that connects your AI to your apps in one place. Walk through the setup once and you're done; takes about 5 minutes.
>
> If you'd rather connect your own tools (e.g., a direct Gmail tool, your own Slack tool, etc.), that works too — just let me know what you have and I'll check.
>
> Which would you like to do? (Default is Composio.)"

## Composio install walkthrough

Used at: `SKILL.md` Step 2a, when user picks the Composio path.

Direct the user (don't run shell commands for them):

> "Here's the Composio setup, in order:
>
> 1. Sign up at composio.dev (free).
> 2. Click **Connect Apps** in the left sidebar.
> 3. Connect the apps you want to use:
>    - **Email** (required, for reading your inbox and sending the report heads-up) — Gmail OR Outlook
>    - **Messaging** (optional, for reading selected Slack channels) — Slack
>    - **Meetings** (optional, for reading meeting transcripts) — Fathom, Granola, or Fireflies
>    - **Task tool** (optional, for reading your assigned tasks) — Notion, Asana, Linear, ClickUp, Todoist, Trello, or Monday
> 4. Click **Install** in the left sidebar.
> 5. Pick the install card for your AI (Claude Code, Claude Desktop, Codex, ChatGPT, etc.) and follow the steps on that page — they're tailored to your AI and OS. The install uses **OAuth (one browser-based 'authorize' click)**; no API keys to copy or paste.
>
> Where are you?
> - **(a) Done — installed and authenticated.** I'll re-check what's connected.
> - **(b) Got stuck on a specific step.** Tell me which one and what you saw.
> - **(c) Composio's dashboard looks different from what I described.** Send a screenshot."

## Composio re-auth (Stage 1.5)

Used at: `SKILL.md` Step 0 / detection callout, Stage 1.5 — when the Composio MCP server is installed in the AI but only the auth-bootstrap tools are loaded (no live meta-tools yet). Show this before triggering the OAuth flow so the user can tell why we're not walking the full install.

Lead with the status line so it's obvious nothing is being skipped silently:

> "Composio is already installed in your AI from a prior session — I just need to re-authorize it for this session, not walk through the full install. Here's the auth link:
>
> Composio's connected to your AI but needs re-authorizing for this session. Open the URL I'll send next in your browser, sign in, approve access, and tell me when you're done — your connected apps will light up automatically. If the redirect page fails to load (it usually does — that's just localhost), copy the full URL from your address bar and paste it back to me."

## Stuck help

Used at: `SKILL.md` Step 2a, when user picks (c) "Dashboard looks different" or otherwise needs visual help.

> "No worries. Send me a screenshot of what you see on your screen and tell me which device you're on (Mac / Windows / Linux). I'll walk you through it from there based on what you see."

## Capabilities not wired — pre-flight

Used at: `SKILL.md` Step 0 / Step 1, when detection finds capabilities are missing. Substitute `[bracketed]` placeholders with the specific missing capabilities at runtime.

Run detection silently first — only show this message if something is actually missing. If all capabilities are present, skip this section and proceed directly to Step 3 (Source picker).

> "Heads up — I don't see [an email tool / a messaging tool / a meetings tool / a task tool / a notification tool] connected to your AI right now. The sweep can still run with just one of these connected, so you don't have to connect them all — but you'll need at least one. Want me to walk you through getting one connected?"

## Fast-path acknowledgement

Used at: `SKILL.md` Step 0 Branch table, when detection lands on the **Fast path** branch (capabilities connected, no config). Show this one-liner immediately before jumping to Step 3 so the user can tell that detection ran and decided to skip the install walk-through deliberately.

Substitute `[list of connected toolkits]` with the readable names from the slug map (e.g., "Gmail, Slack, Fathom, Notion").

> "I see you already have [list of connected toolkits] connected through Composio — let's pick which sources to sweep."

## Source picker

Used at: `SKILL.md` Step 3. After the user responds, save their selections to the local config before continuing to Step 4. If the user picks nothing, re-ask once — don't proceed without at least one source selected.

> "Got it. Which of these do you want me to check for follow-ups? Pick whichever ones you actually use:
>
> - Email
> - Messaging (Slack)
> - Meeting transcripts
> - Task list
>
> Pick any combination, but at least one. I'll skip whatever you don't connect."

## Email account picker

Used at: `SKILL.md` Step 4a, when more than one email account is connected to the chosen email toolkit. Skip this prompt when only one account is connected.

Substitute `[N]`, `[Gmail/Outlook]`, and the account list (use account IDs and aliases — both are useful when multiple aliases share an email address). Show "all merged" as an option for users who treat multiple inboxes as one.

> "I see [N] [Gmail/Outlook] accounts connected to your AI: [account-id-1] (alias: `[alias-or-none]`), [account-id-2] (alias: `[alias-or-none]`), … . Which one should I check for follow-ups?
>
> You can also pick **all of them merged** if you treat these inboxes as one — I'll read across all of them and de-duplicate in the report."

Confirmation:

> "Got it — I'll check [chosen-account-id / all accounts merged] for follow-ups."

## Messaging scope

Used at: `SKILL.md` Step 4b, first ask (before channel list). Captures whether the user wants channel scope, DM scope, or both. Save the answer as `messaging.scope` before moving to channel list capture.

> "What should I sweep on Slack? Pick one:
>
> - **Channels you name** — I'll only scan the specific channels you list (next ask).
> - **All your DMs** — I'll scan your direct messages with people, no channels.
> - **Both** — channels you name plus all your DMs.
>
> Most people pick **DMs** if their commitments mostly happen 1:1, **channels** if a few specific channels are where work gets discussed, or **both** if you're equally active across both."

Confirmation:

> "Got it — I'll sweep [channels you name / your DMs / channels + DMs]."

## Slack channel name-and-verify

Used at: `SKILL.md` Step 4b. Only show this section if the user selected "Messaging" in the Source picker. Accept both `#name` format and raw channel IDs. Save the verified list (found channels only) to the local config.

Initial ask:

> "Which Slack channels do you want me to look at? Just list them — names like `#exec-team` work, channel IDs (`C0123ABCD`) also work. Paste as many as you want."

Resolution feedback (when some channels miss):

> "Found [N] of [M] channels: [list-of-found]. Couldn't find [list-of-missed] — check the spelling, or paste the channel ID and I'll look it up. (You can find a channel ID by right-clicking the channel name in Slack → 'View channel details' → it's at the bottom of the panel.)"

## Meetings source two-tier

Used at: `SKILL.md` Step 4c. Only show this section if the user selected "Meeting transcripts" in the Source picker. Ask the two questions in sequence — first check for a direct integration, then fall back to a Notion/Drive path if needed.

First ask:

> "For meetings, Composio connects directly to Fathom, Granola, and Fireflies. Do you use one of those?"

Fallback ask (if no):

> "Got it. Do you have a place where transcripts land — like a Notion database or a Drive folder I could read from? (You'll need Notion or Google Drive connected in Composio for me to read it; we can do that now if you don't.)"

## Tasks — container + assignee discovery

Used at: `SKILL.md` Step 4d. Provider-agnostic — the SKILL introspects the chosen task toolkit at runtime to learn its shape, then uses these prompts to walk the user through what was found. Substitute `[provider]`, `[container concept]`, `[container name]`, `[field]`, `[type]`, `[options]` from the introspection results.

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

After capture, confirm back: *"Got it — I'll filter the sweep's task pull on `[field]` = `[value]` in `[container name(s)]`."*

**Substep 4 — verification (silent on success).**

After the value is captured, run one test query against the saved containers with the captured filter, bounded to the trailing 14 days. On success, save with no extra prompt. On zero-result, ask once:

> "I queried `[container name(s)]` for tasks where `[field]` = `[value]` in the last 14 days and got 0 results. That might be right (slow period) or wrong (e.g., the value doesn't actually label your tasks). Want me to try a different value, or save as-is?"

Don't loop more than once — accept the user's call.

## User identity capture

Used at: `SKILL.md` Step 5. Wording for the prompts that confirm who you are — used to match your name in meeting transcripts and link tasks and emails back to you.

Most values are pulled silently from your connected accounts. Only ask when something is ambiguous.

**Name confirmation** (after pulling from connected account profile):

> "For matching speakers in your meeting transcripts, you appear as `[name]` — is that right, or do you go by a different display name in meetings?"

**Email aliases** (after setting `primary_email` from the connected account):

> "Do you appear under any other email addresses in meeting transcripts — like a personal email or a work alias? If yes, paste them separated by commas. If not, just say no."

Confirmation after both are captured:

> "Got it — I'll look for `[name]` and `[email(s)]` when matching your commitments in transcripts."

## Voice profile capture

Used at: `SKILL.md` Step 6. Explains what we're doing and why, and confirms the result with the user before saving.

**Pre-capture explanation** (show once before reading any messages):

> "To make the follow-up drafts sound like you — not generic AI — I'm going to read a sample of your recent sent emails and Slack messages. I won't store the messages themselves, just the patterns: how you open, how you close, sentence length, that sort of thing. This takes about 30 seconds."

**Post-capture confirmation** (show a 2–3 sentence summary of what was found, per channel):

For email:

> "Your email voice: [summary of patterns — e.g., 'warm, casual greeting, signs off with "Best," usually, paragraph-form replies']. Does that feel right?"

For Slack:

> "Your Slack voice: [summary of patterns — e.g., 'short messages, lowercase, occasional emoji, no formal sign-off']. Does that sound right?"

**On "looks good":** save and continue.

**On "redo":** read more messages (40 sent emails / 100 Slack messages) and try again. If the second pass still feels off, save with a note that it may need tweaking, and tell the user where the file is so they can edit it by hand:

> "Saved what I found to `[voice profile path]` — you can open that file and adjust the descriptions if anything feels off. Re-running setup later will re-capture fresh."

## Meeting follow-up default channel

Used at: `SKILL.md` Step 7. Single question asked only when both meetings and at least one of email/messaging are wired.

> "When you follow up on something committed in a meeting, do you usually do it via email or Slack?"

Confirmation:

> "Got it — I'll draft meeting follow-ups as [email / Slack messages] by default."

## Notification channel + recipient

Used at: `SKILL.md` Step 8. Ask these two prompts in sequence — do not combine them into one message.

Note: your regular Slack account is **not** a notification option here. Sending a message from your own Slack account to yourself doesn't create a notification ping — it just adds to your message history silently. For Slack notifications, we use a **Slack bot** (a separate app that sends you an actual inbound ping). If you'd prefer not to set up a bot, email works great.

First ask (channel choice):

> "Where should I send a heads-up when your daily sweep finishes? Options:
> - **Gmail** — I'll send a quick email to yourself
> - **Outlook** — same, via Outlook
> - **Slack (bot)** — I'll ping you via a Slack bot. Note: this is different from your regular Slack login — the bot sends you an actual notification, whereas sending from your own account wouldn't ping you.
> - **None** — just save the report file, no heads-up needed
>
> (The report always saves locally to your output folder regardless of what you pick here — this is just the ping.)"

Second ask (recipient, only if channel ≠ none):

For email (Gmail / Outlook):

> "What email address should I send it to?"

For Slack bot:

> "What's your Slack user ID or a channel I should post to? (Your user ID usually looks like `U0XXXXXXXX` — you can find it in Slack under your profile → 'More' → 'Copy member ID'.)"

Confirmation (after both answers):

> "Got it — I'll send the sweep-complete heads-up to [address / channel] via [Gmail / Outlook / Slack bot]."

## Notification account picker

Used at: `SKILL.md` Step 8, only when the chosen notification toolkit has more than one account connected (e.g., three Gmail aliases). Skip this prompt when the toolkit is single-account.

Substitute `[N]`, `[Gmail/Outlook]`, and the account list (use account IDs and aliases — both are useful when multiple aliases share an email address).

> "I see [N] [Gmail/Outlook] accounts connected to your AI: [account-id-1] (alias: `[alias-or-none]`), [account-id-2] (alias: `[alias-or-none]`), … . Which one should I send the sweep-complete heads-up FROM? (If you're not sure, pick the one tied to the email scope that can send messages — usually the work-full-scope alias rather than a filters-only one.)"

Confirmation:

> "Got it — heads-up will send from `[chosen-account-id]`."

## Output folder

Used at: `SKILL.md` Step 9.

> "Where should daily sweep reports save? Default: `followups/` inside this project — keeps the report path clickable straight from chat once the sweep finishes. You can pick somewhere else if you'd like. I'll create the folder if it doesn't exist.
>
> If you'd rather keep reports in a single place across all your projects, pick a path like `~/Documents/Atlas/Followups/` instead — it'll just mean the sweep-complete link won't open directly from chat."

Confirmation (after user confirms or picks a path):

> "Got it — reports will save to [resolved-path]. I'll create the folder automatically on first run."

## Hand-off

Used at: `SKILL.md` Step 11. This is the final message in the setup skill. After delivering it, the setup skill is done — do not proceed into a sweep run unless the user explicitly asks.

> "Setup done. Saved to `client-profile/daily-followup-tracker.local.md`.
>
> The sweep is designed to run daily — checking the last 24 hours each time. To run it on a schedule, set it up in whichever scheduler your AI host provides: Claude Code's `/schedule` command, GitHub Actions cron, your OS's cron, or any other scheduler your host exposes. If you'd rather just kick it off manually, say *'run my daily follow-up sweep'* any time.
>
> The report lands in `[output_folder]` and includes draft replies you can copy and send directly."

---

*These messages were adapted from the energy-audit setup canonical-messages reference. Drift is fine over time — but the starting point lives here so the careful tone work isn't lost on first implementation.*
