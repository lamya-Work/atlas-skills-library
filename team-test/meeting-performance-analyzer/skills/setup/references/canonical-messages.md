# Canonical setup messages

This file holds the verbatim wording of every multi-line message the setup wizard delivers. The SKILL flow loads sections from here by name. Edit this file to change the wizard's voice without touching the SKILL flow.

---

## Intro

> "This plugin needs two capabilities connected to your AI: read your meeting transcripts, and send one email when the audit completes.
>
> The easiest way to get both is to install **Composio MCP** — a free service that connects your AI to your tools (Fathom, Granola, Fireflies, Gmail) in one place.
>
> If you'd rather use your own MCPs (e.g., a direct Fathom MCP, a direct Gmail MCP), that works too — just let me know what you have connected and I'll verify it works.
>
> Which would you like to do? (Default is Composio.)"

---

## Composio install walkthrough

> "Here's the Composio setup, in order:
>
> 1. Sign up at composio.dev (free).
> 2. Click **Connect Apps** in the left sidebar.
> 3. Connect the apps you want to use:
>    - **One of Fathom, Granola, or Fireflies** (required — for meeting transcripts)
>    - **Gmail** (required — for the audit-complete email)
> 4. Click **Install** in the left sidebar.
> 5. Pick the install card for your AI (Claude Code, Claude Desktop, Codex, ChatGPT, etc.) and follow the steps. You'll be asked to confirm with one click in your browser — no API keys or passwords to copy or paste.
>
> Where are you?
> - **(a) Done — installed and signed in.** I'll re-check what's connected.
> - **(b) Got stuck on a specific step.** Tell me which one and what you saw.
> - **(c) Composio's dashboard looks different from what I described.** Send a screenshot."

---

## Stuck help

> "No worries — Composio's UI changes occasionally and the steps above might not match exactly what you're seeing.
>
> Send a screenshot of the screen you're on and I'll walk you through it from there. Or describe what's on screen and what you've tried; I can usually narrow it down from a description.
>
> If you're blocked on the very first step (signup), let me know — sometimes the issue is corporate SSO or a network restriction, and we can route around that."

---

## Composio re-auth (Stage 1.5)

> "Looks like you have Composio installed already, but this session isn't signed in. No need to redo the whole setup — I'll just send you a quick sign-in link.
>
> One moment — getting that link for you now."

(Followed by the call to the authenticate tool. After the user signs in: "Re-checking what's connected...")

---

## Capabilities not connected — pre-flight

> "I tested the connections you described and one capability isn't ready:
>
> - **[capability name]** ([what was tested]): [error description]
>
> Two options:
> - **Fix it in your tool** — usually a quick sign-in again from the [tool] dashboard. Tell me when you've done that and I'll re-test.
> - **Use Composio as a backup for this capability** — ~2 minutes; I'll walk you through connecting [app name] in Composio and re-checking.
>
> Which would you like?"

---

## Connection choice

> "Looks like you have these connected:
> - **Meeting tool**: [list — e.g., 'Fathom via Composio', 'Granola via Composio', 'Fathom direct MCP']
> - **Email**: [list — e.g., 'Gmail via Composio']
>
> Which should this plugin use?"

(If only one option per capability, this prompt is skipped — the SKILL confirms-and-skips with: *"Using [the one option] — sound good?"*)

---

## Fast-path acknowledgement

> "Looks like you've already got everything connected: [list of connected toolkits]. I'll skip the install walk-through and go straight to the setup questions."

(The list is substituted from the connected-toolkit slug list — readable names, not raw slugs.)

---

## Hand-off

> "Setup is complete. Saved to `.claude/meeting-performance-analyzer.local.md`.
>
> Next:
> 1. Run `extract-meeting-preferences` once to capture the executive's known opinions about recurring meetings (non-negotiables, already-flagged-to-cut, restricted types).
> 2. Then `meeting-performance-audit` is ready to produce the weekly brief.
>
> The audit is designed to run weekly. To schedule it, use whichever scheduler your AI host provides — for example Claude Code's `/schedule`, Cowork's scheduler, GitHub Actions, or an OS-level cron job. The plugin itself does not drive a schedule.
>
> Re-run `setup` any time to change a connection, swap the meeting tool, or update the executive's identity."

---

## Refresh mode prompts

For each field, the SKILL asks one of:

> "Current `<field>`: `<current value>`. Keep this, or change?"

If the user says change, the corresponding capture question fires (per Steps 4, 5, 6).
