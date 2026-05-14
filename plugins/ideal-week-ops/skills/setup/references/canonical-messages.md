# Canonical user-facing messages — ideal-week-ops setup

These messages were adapted from the `conference-contact-capture` plugin's setup skill and scoped to the two capabilities ideal-week-ops needs (calendar read + notification send). The two copies are independent — future updates to the conference version do not auto-propagate.

Section labels are descriptive plain names — not the cryptic numbering used in some sister plugins. Each section corresponds to a specific point in the `setup/SKILL.md` flow.

> The setup skill loads this reference and uses these messages verbatim at the points specified in `SKILL.md`. They were drafted during the design phase to preserve a careful, explicit, non-jargony voice. Substitute `[bracketed]` placeholders at runtime; otherwise preserve the wording.

`[bracketed]` = substitute at runtime (names, paths, errors).

---

### Intro — MCP-aware Composio introduction

Used at: `SKILL.md` Step 1, when introducing the wiring choice.

> "This plugin needs two capabilities wired into your AI: **read your calendar**, and **send one notification per scan**.
>
> The easiest way to get both is to install **Composio MCP** — a free service that connects your AI to your tools (Google Calendar, Outlook, Slack, Gmail, etc.) in one place. We'll walk through the setup together; takes about 5 minutes.
>
> If you'd rather wire your own MCPs (e.g., the Google Calendar MCP, your own Slack MCP, etc.), that works too — just let me know what you have and I'll check.
>
> Which would you like to do? (Default is Composio.)"

### Composio install walkthrough

Used at: `SKILL.md` Step 2a, when user picks the Composio path. Lifted verbatim from conference-contact-capture's `setup/SKILL.md` §2a, with the apps list scoped to this plugin's two capabilities.

Direct the user (don't run shell commands for them):

> "Here's the Composio setup, in order:
>
> 1. Sign up at composio.dev (free).
> 2. Click **Connect Apps** in the left sidebar.
> 3. Connect the apps you want to use:
>    - **Google Calendar OR Outlook Calendar (required, for calendar reads)**
>    - **At least one of Slack, Gmail, or Outlook (required, for sending notifications)**
> 4. Click **Install** in the left sidebar.
> 5. Pick the install card for your AI (Claude Code, Claude Desktop, Codex, ChatGPT, etc.) and follow the steps on that page — they're tailored to your AI and OS. The install uses **OAuth (one browser-based 'authorize' click)**; no API keys to copy or paste.
>
> Where are you?
> - **(a) Done — installed and authenticated.** I'll re-check what's wired.
> - **(b) Got stuck on a specific step.** Tell me which one and what you saw.
> - **(c) Composio's dashboard looks different from what I described.** Send a screenshot."

### Stuck help — install troubleshooting

Used at: `SKILL.md` Step 2a, when user picks (c) "Dashboard looks different" or otherwise needs visual help.

> "No worries. Send me a screenshot of what you see on your screen and tell me which device you're on (Mac / Windows / Linux). I'll walk you through it from there based on what you see."

### Capabilities not wired — pre-flight

Used at: `SKILL.md` Step 0 / Step 1, when detection finds capabilities are missing. Substitute `[bracketed]` placeholders with the specific missing capabilities at runtime.

> "Heads up — I don't see [a calendar tool / a notification tool] wired into your AI right now. This plugin needs that to [read your calendar for the daily scan / send the daily ping when calendar conflicts are flagged].
>
> Want me to walk you through getting one connected? The easiest path is Composio MCP — takes about 5 minutes. If you have your own MCP for [calendar / notifications], that works too — just let me know what you have and I'll check."

---

### Legacy-path migration

Used at: `SKILL.md` Step 0, only when an existing `.claude/ideal-week-ops.local.md` is detected and no new-path config exists. Surfaced once, after the file copy completes, before continuing into the three detection checks.

> "Migrated your wiring config from `.claude/ideal-week-ops.local.md` to `client-profile/ideal-week-ops.local.md`. You can delete the old file once you've confirmed things work."

---

*These messages were adapted from the conference-contact-capture design phase. Drift is fine over time — but the starting point lives here so the careful tone work isn't lost on first implementation.*
