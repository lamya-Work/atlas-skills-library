# energy-audit

> A bi-weekly review that finds what to automate next — and builds the plan for it.

## What it does

Every two weeks, the energy-audit plugin reviews the executive's recent work across calendar, tasks, meeting transcripts, and selected Slack channels — whichever sources you've connected — and surfaces the one or two workflows most ready to hand off to a builder.

Here's what happens during an audit:

- **Reads recent activity from connected sources.** Calendar and task data are pulled inline. Meeting transcripts and Slack channels are processed in parallel by separate sub-agents, each returning structured candidate briefs with source attribution. At least one source is required; the audit skips anything that isn't connected.
- **Applies five readiness criteria filtered through the executive's energy profile.** Candidates that don't pass all five are dropped. Drains rank highest; quarterly priorities break ties. Anything that pulls toward the executive's Zone of Genius is left alone.
- **Picks at most two candidates.** The cap is two — never more. Ranking, not enumeration.
- **Builds a complete automation plan for each top candidate.** The audit hands each one to the `build-automation-plan` skill, which produces a twelve-section plan: problem, proposed automation, capabilities, wiring, build steps, success criteria, edge cases, and effort estimate.
- **Saves one report file and sends one notification email.** The audit report and the embedded plans land in a single durable file. One real email goes to the configured recipient — Gmail or Outlook only.

Because meeting transcripts and Slack messages are content-sensitive, the audit only reads sources you've explicitly connected and configured in `setup`. Nothing is pulled from sources outside that list.

A thirty-minute review every two weeks. One or two automation plans you can hand to anyone — engineering, ops, an outside contractor — and they can build it.

## Who it's for

Two audiences: executives who want to find what to delegate or automate without conducting manual workflow audits themselves, and executive assistants who own the audit cadence on the executive's behalf.

The plugin is built for executives who already have a digital footprint in at least one of the four sources — calendar, task tool, meeting transcripts, or Slack. If the executive doesn't use any of these tools consistently, the audit won't have enough signal to work with.

The plugin is conservative by design. One or two plans per cycle, not a backlog. If no candidate meets the bar this period, the audit says so and produces no plan.

## Required capabilities

The host agent needs these capabilities wired up. Names are abstract — map them to whatever tools your runtime provides.

- **Calendar read** *(optional, but at least one of the four data-source capabilities is required)* — list events for a date range, per account
- **Task tool read** *(optional)* — list tasks assigned to a user for a date range
- **Meetings read** *(optional)* — list meeting transcripts for a date range, OR list files in a Notion DB or Google Drive folder
- **Slack read** *(optional)* — list channel messages for a date range, resolve channel IDs
- **File read + write** — for the energy profile, local config, run log, and output reports
- **Sub-agent dispatch** — meeting and Slack sources are split across parallel sub-agents at audit time
- **Notification email send** — Gmail or Outlook (Slack is a data source only — not a notification channel)
- **Skill invocation** — the audit calls `build-automation-plan` for each top candidate
- **Recurring trigger** *(optional but recommended)* — the host's scheduler triggers the audit on cadence; the plugin records the anchor but does not own the scheduler

At least one of the four data-source capabilities must be connected. The audit skips any source that isn't.

## Suggested tool wiring

| Capability | Validated default (Composio MCP) | Alternatives |
|---|---|---|
| Calendar read | `googlecalendar`, `outlook` toolkits via Composio | Anthropic Google Calendar MCP, direct Outlook MCP |
| Task tool read | `notion`, `asana`, `linear`, `clickup`, `todoist`, `trello`, `monday` toolkits via Composio | Direct vendor MCPs (Notion MCP, Linear MCP, etc.) |
| Meetings read | `fathom`, `granola`, `fireflies` direct toolkits OR `notion` / `googledrive` (transcript folder fallback) via Composio | Direct vendor MCPs where available |
| Slack read | `slack` toolkit via Composio | Direct Slack MCP |
| Notification email send | `gmail` or `outlook` toolkit via Composio | Direct Gmail MCP, direct Outlook MCP |

**Recurring trigger** is runtime-agnostic — the plugin records the schedule anchor at `setup`; the host's scheduler does the actual triggering. Use Cowork Scheduled tasks, Claude Code `/schedule`, GitHub Actions cron, or OS cron — whatever your host supports.

Composio MCP is the validated default because it covers all four data sources plus notification email through one onboarding flow. Direct vendor MCPs work too — `setup` detects either path. Slack does not appear in the notification column because it is a data source only.

## Installation

```
/plugin marketplace add colin-atlas/atlas-skills-library
/plugin install energy-audit@atlas
```

After install, run `setup` to connect your tools and pick which sources the audit reads.

## First-run setup

1. **Run `setup`.** Detection-first onboarding — only walks you through what's missing. Captures which sources to use, your Slack channel list, where meeting transcripts live, your notification email channel and recipient, and your output folder. Saves to `.claude/energy-audit.local.md` in your workspace.

2. **Run `extract-energy-profile`.** Captures the executive's Zone of Genius, drains, recurring patterns, and quarterly priorities. If you already have a prefs doc, OKRs, or a "how I work" page, paste it and the skill drafts the profile from that — it only asks the questions the doc doesn't already answer. Saves to `client-profile/exec-energy-profile.md` in your workspace.

3. **(Optional) Wire the recurring trigger.** The audit is designed to run on a bi-weekly cadence. The plugin does NOT register the recurrence itself — that's wired in your AI host's scheduler. Pick whichever your host supports:
   - **Cowork Scheduled tasks** — schedule "run the energy audit" on a bi-weekly cadence
   - **Claude Code `/schedule`** — `/schedule "run the energy audit"` on a bi-weekly cadence
   - **GitHub Actions / OS cron** — invoke the audit skill every 14 days

After these three steps, run `energy-audit` any time to produce a report — or wait for the scheduled run to fire.

## Skills included

**`setup`** — *neutral.*
One-time onboarding wizard. Detects connected calendar, task, meetings, Slack, and notification-email tools. Walks through Composio install if anything is missing. Captures which sources to use, Slack channels, meetings location, notification settings, and output folder. Resumable and re-runnable — re-run at any time to refresh any field. Recurring-audit cadence is registered in your AI host's scheduler, not in `setup` — the Hand-off message lists the options.
**Trigger phrases:** *"set up energy audit"*, *"configure energy audit"*, *"refresh energy audit setup"*

---

**`extract-energy-profile`** — *opinionated.*
Captures the four content fields the audit reads — Zone of Genius, drains, recurring patterns, and quarterly priorities — into `client-profile/exec-energy-profile.md`. Doc-upload-first: if you have an existing prefs doc, the skill drafts from it and only asks the questions the doc doesn't answer. Resumable.
**Trigger phrases:** *"extract energy profile"*, *"refresh my energy profile"*, *"redo energy profile"*

---

**`energy-audit`** — *opinionated.*
The audit itself. Reads the trailing 14 days from connected sources — calendar and tasks inline, meetings and Slack via sub-agent fan-out. Applies the five readiness criteria filtered through the energy profile, ranks survivors, takes the top one or two (hard cap), writes a structured report, hands each top candidate to `build-automation-plan`, and sends one notification email. Detects run-overlap if the previous audit was less than 14 days ago.
**Trigger phrases:** *"let's run an energy audit"*, *"run the energy audit"*, *"what can we automate"*, *"find automation opportunities"*

---

**`build-automation-plan`** — *opinionated.*
Produces a twelve-section automation plan from a candidate brief. Auto-invoked by `energy-audit` for each top candidate (Path A). Also runnable directly when you want to plan something the audit didn't surface (Path B). Cites the audit's source attribution and evidence strength in the plan body.
**Trigger phrases:** *"build an automation plan for X"*, *"draft an automation plan"*, *"spec out automating X"*

## Customization notes

- **Source mix.** Configurable in `.claude/energy-audit.local.md` (written by `setup`). Each source block (`calendar`, `tasks`, `meetings`, `slack`) is omitted entirely if you don't want that source read.
- **Audit cadence.** Default is bi-weekly (every 14 days). The plugin does NOT register the recurrence — your AI host's scheduler does. Change the cadence in your scheduler. The methodology assumes a 14-day window; if you change to weekly or monthly, also adjust `references/atlas-energy-audit-methodology.md` to keep the framing consistent.
- **Notification.** Gmail or Outlook only. Re-run `setup` (refresh mode) to change channel or recipient. Set `notification_channel: none` if you only want the report file written without an email ping.
- **Output folder.** `output_folder` field in `.claude/energy-audit.local.md`. Default `audits/` (workspace-relative — keeps the report path clickable from chat). For a cross-project library, override to an absolute path like `~/Documents/Atlas/Energy-Audits/` — the trade-off is that chat-link previews won't open the report directly.
- **Sub-agent extraction prompt.** The meetings and Slack sub-agent prompt template lives at `skills/energy-audit/references/sub-agent-extraction-prompt.md`. Edit there to change extraction behavior; the `energy-audit/SKILL.md` body stays stable.
- **Methodology.** Readiness criteria, ranking weights, signals breakdown rules, short-window mode rule, and report structure all live at `skills/energy-audit/references/atlas-energy-audit-methodology.md`. Edit there to fork the methodology.
- **Plan template.** The twelve-section automation plan structure lives at `skills/build-automation-plan/references/atlas-automation-plan-template.md`. Edit there to change plan section guidance.
- **Setup wording.** User-facing copy lives at `skills/setup/references/canonical-messages.md`. Edit there to change tone or wording without touching the skill body.

## Atlas methodology

The audit encodes a few principles that are easy to loosen but exist for a reason:

- **Five readiness criteria.** Every candidate must pass all five: (1) repetitive or triggered by a clear event; (2) AI can execute end-to-end or handle a substantial chunk; (3) clear inputs and outputs; (4) profile alignment — a drain or a quarterly priority; (5) buildable with current tooling. Failing any one criterion drops the candidate — it doesn't lower its rank.
- **Hard cap of two plans.** The audit picks at most two top candidates. Forces ranking rather than enumeration. More than two plans dilutes execution focus.
- **Profile as filter.** Drains rank highest; quarterly priorities are the tiebreaker. Candidates that pull toward the executive's Zone of Genius are not automated — that's their best work, not a burden.
- **Conservative output.** A zero-plan audit is correct output when the data doesn't support a confident recommendation. The methodology does not lower the bar to keep the report from looking empty.
- **Short-window mode.** When a manual run lands less than 14 days after the previous audit, you can choose to look only at the new days. In that mode, criterion 1 (repetitive) is interpreted softly — a single observation passes only if the pattern has explicit recurrence markers.
- **Sub-agent extraction.** Meeting and Slack content is split across parallel sub-agents. Each sub-agent receives the same prompt template and returns structured candidate briefs with source attribution and evidence strength. Keeps the main context window manageable when sources are noisy.

Full methodology details live at `skills/energy-audit/references/atlas-energy-audit-methodology.md`. Fork it to override the Atlas defaults.

## Troubleshooting

**"I don't see a connected tool" / setup says capabilities are missing.**
Run `setup` (refresh mode walks through detection). Check your Composio dashboard at composio.dev → Connect Apps. If Composio shows the app as connected but `setup` doesn't see it, restart your AI client to pick up newly-added MCP servers.

---

**Audit reports "no energy profile found."**
Run `extract-energy-profile` first. The audit reads `<workspace>/client-profile/exec-energy-profile.md` — if you've never run the extract skill, that file doesn't exist yet.

---

**Audit ran but produced zero plans.**
Not necessarily a bug. The methodology rejects weak candidates. Check the report's signals breakdown section — if it shows healthy volume, the data may not have a clearly automatable pattern this period. If it shows low volume, your connected sources may not be capturing enough activity. Consider connecting more sources via `setup`.

---

**Notifications aren't arriving.**
Check `notification_channel` in `.claude/energy-audit.local.md`. If it's set to `none`, the audit completes silently — only the report file is written. If it's `gmail` or `outlook`, verify the email tool is connected by re-running `setup`. Slack is not supported as a notification channel — if you set one up with an older version, switch to Gmail or Outlook via `setup` refresh mode.

---

**Plan generation halts mid-audit.**
The audit reads the report file back after each `build-automation-plan` call to verify the plan was appended. If it halts saying "Plan generation incomplete," the report file is saved with the candidates section but without the plan(s). Check the `build-automation-plan` skill's diagnostic output for the specific failure — typically a malformed candidate brief or a write-permission error on the output folder.

---

**Audit reports "no profile found" but the file exists.**
Workspace-versus-plugin path issue. The audit resolves `client-profile/exec-energy-profile.md` against your current working directory when you invoke the audit — not the plugin install folder. Confirm you're invoking from the workspace where the profile actually lives. The audit deliberately does not auto-fall-back to the plugin folder.
