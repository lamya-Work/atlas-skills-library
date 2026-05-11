# meeting-performance-analyzer

A weekly Atlas plugin that turns the past week of meeting transcripts into a tight performance brief. Scores each meeting on Atlas's quality dimensions, applies the Meeting Worth Matrix to recurring meetings, and surfaces constructive leadership reflection — all in one runnable artifact you can drop into your weekly review.

## 1. What it does

Once a week, this plugin:

- Pulls all meeting transcripts from the past 7 days (since your last run, capped at 7 days)
- Sends each meeting to its own focused sub-agent for a quality review
- Scores meetings on seven dimensions: decisions, action items, time discipline, participation balance, async potential, agenda quality, ownership/deadlines
- Applies the Atlas Meeting Worth Matrix to recurring meetings (KILL / DELEGATE / TRANSFORM / PROTECT) with a reclaimable-time estimate per recommendation
- Synthesizes a leadership reflection across the week — strengths and growth areas in how the executive showed up
- Writes a two-tier brief (executive summary + detail sections) and sends one notification email when the run completes

It does not route summaries, send action items downstream, or drive a scheduler. Analysis and reporting only.

## 2. Who it's for

Executive assistants and chiefs of staff who want to:

- Identify meetings that have outlived their usefulness
- Surface constructive feedback for the executive about how they show up in meetings
- Quantify the time reclaimable from a meeting-pruning exercise
- Run the Atlas Meeting Worth Matrix on real transcript evidence rather than from memory

Works for any executive whose meetings are recorded in a transcript-capable tool (Fathom, Granola, Fireflies, or equivalent).

## 3. Required capabilities

(Abstract list — no specific tool names. Specific Claude Code tools and MCP servers that satisfy these capabilities are listed in §4.)

- **Meeting tool — read.** List meetings in a date window; fetch transcript and metadata (title, attendees, duration, recurrence-marker if available) for each meeting.
- **Email — send.** Send one notification email when the audit run completes.
- **Toolkit-discovery and connection-management.** Required for setup detection — used to figure out what the user has connected and to enumerate accounts when more than one is wired (e.g., multiple Gmail aliases).
- **File read / write / list against the user's workspace.** For the local config, preferences file, last-run marker, and output briefs.
- **Sub-agent fan-out.** The audit dispatches one sub-agent per meeting for the per-meeting review.

## 4. Suggested tool wiring (Claude Code default)

The validated default is Composio MCP — a single MCP server that brokers connections to Fathom, Granola, Fireflies, Gmail, and many other apps. The setup wizard walks you through installing Composio if you haven't yet.

| Required capability | Concrete wiring |
|---|---|
| Meeting tool — read | Composio MCP with one of {Fathom, Granola, Fireflies} connected |
| Email — send | Composio MCP with Gmail connected (or any direct Gmail MCP) |
| Toolkit discovery / connection management | Composio MCP meta-tools (`COMPOSIO_SEARCH_TOOLS`, `COMPOSIO_MANAGE_CONNECTIONS`) |
| File read / write / list | Claude Code's built-in Read / Write / Glob / Grep tools against the workspace |
| Sub-agent fan-out | Claude Code's built-in Task tool |

**Direct vendor MCPs work too.** If you already have a direct Fathom MCP or a direct Gmail MCP wired, the setup wizard's own-MCPs branch will verify each capability with a real test call and let you use what you already have.

## 5. Installation

```
/plugin marketplace add colin-atlas/atlas-skills-library
/plugin install meeting-performance-analyzer@atlas
```

(Until promotion to `plugins/`, the plugin is installed by checking out the repo locally and pointing your plugin loader at `team-test/meeting-performance-analyzer/`.)

## 6. First-run setup

Run the wizard once:

```
Use the setup skill from meeting-performance-analyzer
```

The wizard auto-detects what's already connected and walks you through what's missing. First-time users with no Composio connection are walked through the full sign-up at composio.dev → Connect Apps → Install for AI host. Returning users who already installed Composio but are signed out get a quick one-click sign-in instead of a full re-install.

The wizard captures:

1. The connected meeting tool (or which one to use, if more than one is connected)
2. The notification email account (or which Gmail alias, if more than one is wired)
3. The executive's name as it appears in transcripts (and optional email — used to attribute leadership observations correctly)
4. The output folder for briefs (default `meeting-briefs/`, workspace-relative)

After setup, run `extract-meeting-preferences` once to capture your executive's known opinions about recurring meetings (non-negotiables, already-flagged-to-cut, restricted types).

## 7. Skills included

| Skill | Methodology stance | Purpose |
|---|---|---|
| `setup` | neutral | One-time onboarding wizard. Detects connections, runs full Composio install if needed, captures account selection, exec identity, and output folder. Resumable mid-flow; re-runnable to refresh any field. |
| `extract-meeting-preferences` | opinionated | Captures the executive's known opinions in one canonical preferences file (non-negotiables, already-flagged-to-cut, restricted types). Doc-upload-first: parses any prefs doc you provide, only asks for what's missing. |
| `meeting-performance-audit` | opinionated | The weekly run itself. Pulls past-7-days transcripts, fan-out one sub-agent per meeting, applies quality scoring + Worth Matrix to recurring meetings, synthesizes leadership reflection, writes the brief, sends one notification email. |

## 8. Customization notes

The three Atlas methodology references — `atlas-meeting-worth-matrix.md`, `atlas-meeting-quality-dimensions.md`, `atlas-leadership-reflection-tone.md` — live under `skills/meeting-performance-audit/references/`. To adapt the methodology to a specific client, fork the plugin and edit those reference files. The SKILL bodies are stable; reference docs are the override surface.

The setup wizard's wording (intro, Composio walkthrough, stuck-step recovery, refresh prompts, hand-off message) lives in `skills/setup/references/canonical-messages.md` — edit there to change the voice without touching the SKILL flow.

The default brief output folder is `meeting-briefs/` relative to the workspace. To use a different folder (including an absolute path for cross-project libraries), re-run `setup` and pick a different folder at the output-folder step.

## 9. Atlas methodology

This plugin operationalizes two pieces of Atlas methodology:

**The Meeting Worth Matrix.** A 2×2 framing — *Is this meeting necessary?* × *Does the executive need to be there?* — yielding four verdicts:

- **KILL** — Eliminate entirely. Replace with an async update.
- **DELEGATE** — Send someone else. Brief the executive after.
- **TRANSFORM** — Change the format or frequency (60 min → 30 min, weekly → biweekly).
- **PROTECT** — Keep and give it full prep. These are worth the time.

The Matrix applies to recurring meetings only. One-offs cannot be killed/delegated/transformed/protected — they already happened. One-offs are still quality-scored.

**The seven quality dimensions** (applied to every meeting, recurring or one-off):

1. Decision output — were strategic decisions made in the room?
2. Action items — were owners and deadlines assigned for each next step?
3. Time discipline — did the meeting start on time, end on time, stay on the agenda?
4. Participation balance — did the right voices speak; was airtime distributed appropriately?
5. Async potential — could this have been a Slack post, an email update, or a doc?
6. Agenda quality — was there a stated purpose; was it referenced?
7. Ownership / deadlines — were follow-ups concrete and assigned?

The leadership reflection layer reads across the week's meetings and surfaces patterns in how the executive shows up — strengths to keep and growth areas to develop. Constructive tone, not critical.

Full methodology documents live in `skills/meeting-performance-audit/references/`.

## 10. Troubleshooting

**"The audit produced an empty brief."**
The window had no meetings, OR every meeting in the window matched a `restricted_types` entry in the preferences file (and was therefore excluded). The brief writes the executive summary either way and the notification email still sends — check the brief file: it should explicitly say "No meetings in this window" or "All meetings in this window were marked restricted."

**"The leadership reflection mentions someone who isn't the executive."**
The transcript-attribution step couldn't reliably identify the executive in the transcripts. Re-run `setup`, refresh the exec-identity field, and provide the optional email field — the email match is more reliable than name-only when transcripts use only first names or initials. If the meeting tool's transcript doesn't include attendee email metadata, the SKILL falls back to name match against the title and speaker labels.

**"The audit fails partway through with a sign-in error."**
Composio sessions can expire between runs. Re-run `setup` — it will detect that you're already installed but signed out and walk you through a quick one-click sign-in (no need to redo the whole install). The `last-run.json` is NOT updated when the audit fails, so the next successful run picks up from the same window.

**"I ran the audit twice in the same week and got two briefs covering different windows."**
That's expected. The audit window is dynamic: each run covers "since the last successful run, capped at 7 days." Two runs two days apart will produce two briefs whose windows don't overlap. To re-analyze the same window, manually delete `last-run.json` from the workspace before re-running.

**"The brief's exec summary says 'You haven't run this in 23 days.'"**
That's the gap-flag — surfaces when the last run was more than 7 days ago. Only the last 7 days are analyzed in the current run; anything between 7 and 23 days ago is silently dropped. Re-run more frequently to keep coverage continuous.
