# Per-source detection guidance

Per-source query patterns, recipient extraction rules, and provider-specific hints for each sub-agent. The methodology file (`atlas-followup-detection-methodology.md`) carries the *what* and *why* of detection; this file carries the *where to look* for each source.

## How sub-agents use this file

Each sub-agent reads only its own section here plus the methodology file. There is no cross-source coupling — the email-scan agent does not know what the messaging-scan agent is doing, and the per-meeting-scan agent does not know about tasks. Cross-source resolution happens later in the orchestrator (Step 5c of the work skill), not inside any sub-agent. This keeps each agent's context minimal and its prompt focused on a single source's quirks.

## Email source

For the email-scan agent. The agent runs in two modes — open-loop and today's-new — and merges results.

**Open-loop query syntax (Gmail-flavored).** `is:unread in:inbox -from:me older_than:1d`. The unread filter catches threads the user hasn't engaged with; the inbox filter avoids archived threads; the negative `-from:me` filter excludes drafts the user sent that bounced back; the `older_than:1d` filter avoids the freshest-of-the-fresh, which the user is probably about to act on anyway.

**Direction-confirmation rule.** Once the open-loop query returns candidate threads, fetch each thread and confirm that the latest message in the thread is from someone *other than the user*. Match against `user_identity.primary_email` and any aliases listed in `user_identity.aliases`. If the latest message is from the user, drop the thread — the user already replied and the open loop is closed even if the thread is still marked unread for some reason.

**Today's-new query.** `from:me after:YYYY/MM/DD` where the date is the activity window's start, formatted as Gmail expects (slashes, year first). Returns messages the user sent in the window — these are the source for today's-new commitment detection.

**Outlook variant.** For provider `outlook`, the equivalent of the open-loop query is a folder-and-state filter: `Inbox + UnRead + ReceivedTime > <window.start>`. The today's-new equivalent uses the Sent Items folder filtered by `SentDateTime > <window.start>`. The agent prompt itself should not branch on provider — the agent reads `email.provider` from the scoped config and translates the abstract query intent ("unread in inbox, not from me, older than a day") into the provider's specific filter syntax. Keep the prompt provider-agnostic in shape; let provider-specific hints live here.

## Messaging source

For the messaging-scan agent. Runs today's-new mode only — open-loop detection on messaging isn't enabled (chat threads don't have the same "awaiting reply" state as email).

**Today's-new query (Slack-flavored).** Use `SLACK_SEARCH_MESSAGES` (or the equivalent) with `from:me after:<window.start as Slack timestamp>` plus `in:#<channel>` modifiers for each channel listed in `messaging.channels`. If the toolkit's search rejects channel-scope modifiers, fall back to the bare `from:me after:<ts>` query and filter the results client-side against `messaging.channels` channel IDs.

**Recipient extraction rules.** Different message contexts produce different recipients:

- **DM** — the recipient is the other person on the conversation. Pull the other party's user ID and name from the channel metadata.
- **Channel** — the recipient is the explicit @-mention or named addressee in the message text. If the user wrote "@sam I'll get back to you Thursday", the recipient is Sam. If the user wrote "I'll get back to you Thursday" with no @-mention but the message is in a 2-person channel context, the recipient is the other channel member. If neither — multiple people in the channel, no @-mention, no named addressee — set `person` to the channel name and confidence to medium at most; the addressee is genuinely ambiguous.

For provider `microsoftteams`, channels are called "channels" and DMs are called "chats" — same extraction logic applies, just under different terminology.

## Meeting transcript source

For the per-meeting-scan agent. One agent runs per meeting in `today_meetings`; each scans exactly one transcript.

**Speaker identification.** Match against `user_identity.name` from the config — case-insensitive, allow first-name-only matches if the first name is unambiguous in the meeting. If the meeting had two attendees with the same first name (e.g., two Sams), fall back to email-based labels via `user_identity.meeting_attendee_emails`. If the transcript only labels speakers by initials or by speaker number ("Speaker 1: ..."), use the attendee-list ordering and the meeting platform's own speaker mapping if exposed.

**Common transcript shapes.** The agent should expect any of these:

- `Speaker Name: text` — most common with Fathom, Granola, Fireflies.
- `Name (timestamp): text` — common with manually-edited transcripts and some integrations.
- Structured JSON with a `speaker` field per segment — direct toolkit API responses, especially from `fathom` and `fireflies`.

**Multi-speaker disambiguation.** If two attendees have the same first name and only first names appear in the transcript, fall back to matching against `user_identity.meeting_attendee_emails`. Some platforms label by email when the platform has SSO context — the agent should check for that pattern first before guessing. If disambiguation is genuinely impossible (two Sams, no email labels, no other distinguishing data), drop the meeting from the run with a `notes[]` entry explaining the skip; do not guess.

## Task source

For the task-scan agent. The provider determines what "container" means and how the assignee filter resolves.

**Per-provider container concept.** What `tasks.container_ids` refers to:

- **Notion** — a database. Container IDs are database UUIDs.
- **Asana** — a project. Container IDs are project IDs.
- **Linear** — a team. Container IDs are team IDs.
- **Trello** — a board. Container IDs are board IDs.
- **ClickUp** — a list. Container IDs are list IDs.
- **Todoist** — there is no per-project container concept aligned with this skill's needs; for Todoist, `container_ids` carries the literal sentinel `"all"` and the agent queries the user's full task list.
- **Monday** — a board. Container IDs are board IDs.

The skill body itself uses the provider-neutral term `container_ids` and asks the agent to introspect each container's schema before applying filters. The mapping above is for the agent's interpretation — not for branching skill behavior.

**Property-type-aware filters.** Use the operator-per-type cheatsheet from `atlas-followup-detection-methodology.md` (`## Filter operator cheatsheet`). Build the filter using the type listed in `tasks.assignee_property_type` from the config; do not guess the type from the property name. The schema introspection step that runs at setup pinned the type already.

**Relation-type assignment.** When `tasks.assignee_property_type` is `relation` (Notion's "Person" via a related Members database, for example, or any task system that links assignees through a relation rather than a typed people field), the filter needs the *linked row's UUID*, not the user's email. Resolve the linked row first by querying the related container for the user's email or name, capture the row UUID, cache it for the run, and use the cached UUID in all subsequent assignee filters. Without the cache, the agent would run one extra resolution query per container — wasteful for runs against multiple containers in the same workspace.

## What every agent must do regardless of source

The contract every sub-agent honors:

- **Return only structured findings.** Never return raw email bodies, raw transcript content, raw message threads, or raw task descriptions. The `commitment` field is a short quote or paraphrase; the `topic` field is a short summary. The orchestrator's context stays clean.
- **Topic strings stay under 10 words.** The dedupe and cross-source resolution algorithm depends on first-5-meaningful-words matching; longer topics undermine that and also bloat the report.
- **Source refs are stable and back-resolvable.** `gmail:thread:<id>`, `slack:#<channel>:<ts>`, `<provider>:meeting:<id>:<timestamp>`, `notion:<container>:<page>` — pick a stable shape per source so the user can paste the ref into the source UI and find the originating record.
- **Apply confidence rules from the methodology.** Every finding gets exactly one of `high` / `medium` / `low`. The drafting logic depends on it.
- **Apply the voice profile to drafts.** Read the section appropriate to the draft's destination channel. If `voice_profile_missing` is set on the run, draft in default tone and prepend a `[VERIFY VOICE]` flag to the draft body.
