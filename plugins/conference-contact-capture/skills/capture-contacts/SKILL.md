---
name: capture-contacts
description: Main orchestrator for the conference-contact-capture plugin. Takes your post-event brain dump (free-form text on people you met), processes each contact through 7 phases — pre-flight → intake → LinkedIn research → company research → brief generation → email drafting → CRM routing → humanized chat summary. Capability-based: uses whatever email / web / CRM tools the host has wired (Composio MCP is the validated default). CRM target database + field mapping live in the profile (filled during `setup`), so capture-contacts inserts CRM records silently from its first run forward.
when_to_use: After `setup` has been run (profile exists). Trigger phrases — "capture contacts from [event]", "process my contacts from [event]", "I have notes from [event] to process", "follow up on [event] contacts". Resume trigger after rate limit — "continue conference capture for [event]".
atlas_methodology: opinionated
---

# capture-contacts

Main orchestrator. Reads the profile, processes a brain dump end-to-end, surfaces a humanized summary at the end. Per-contact failures never halt the run.

## Purpose

You return from a conference, networking event, or any other meeting with notes on people you met. This skill turns those notes into structured briefs (per person), drafted personalized emails (in your voice), and CRM records — all routed through whatever tools the host has wired.

## Inputs

- **Profile** at `client-profile/exec-conference-profile.md` (created by `setup`)
- **Email voice** at `client-profile/conference-email-voice.md` (created by `setup`)
- **Exec themes** at `client-profile/exec-themes.md` (created by `setup`)
- **Available tools** in the host's loaded tool list — email read+draft, web search+extract, CRM (optional)
- **Brain dump** from the user — free-form text, captured during Phase 1

## Required capabilities

(Abstract — see plugin README §4 for validated default wiring.)

- Detection of available tools (same as setup skill)
- Email read (pull recent emails not used here, but the same email tool is needed for drafts)
- Email draft create
- Web search (find homepage URL)
- Web extract single URL (read homepage + sub-pages)
- CRM list databases / get schema / insert record (optional)
- Subprocess (invokes embedded `linkedin-research` skill)
- File read + write
- Conversational interview

## Phases

> **Note on paths.** All `client-profile/...` paths in this skill resolve relative to the **workspace** — the user's current working directory at the time the skill is invoked (i.e. where the host runtime was started). Do NOT resolve them against the plugin's install directory; that location is read-only and shared across users. If a `client-profile/...` file appears missing, confirm the lookup is happening in the workspace before reporting setup as incomplete.

### Phase 0 — Pre-flight

Verify in order:

1. **Profile exists** at `client-profile/exec-conference-profile.md`. If not → use canonical-messages §10.5 from 2026-04-29 spec ("first time" friendly walkthrough); offer to run `setup` right there.
2. **Required capabilities are wired** in the host's tool list (email + optionally web + optionally CRM, based on what's in the profile). If any required capability missing → use canonical-messages §10.4-Phase0; offer to walk user through wiring.
3. **LinkedIn skill onboarded** — check `~/.linkedin-scraper-config.json` exists and is valid. If not → use canonical-messages §10.5; offer to run LinkedIn setup right there.

If any pre-flight check fails and the user declines the offered fix, halt cleanly with no partial state.

### Phase 1 — Intake

**Event name — or skip.** Ask: *"What event was this? If it wasn't a formal event — a one-off meeting, casual intro, anything informal — just say 'casual' and I'll group it under the month."* Accept either an event name or a "casual" / "no event" answer. If casual, default the folder slug to `meetings-<YYYY-MM>` (e.g. `meetings-2026-04`); contacts from different casual meetings in the same month land in the same folder.

**Brain dump.** Ask for free-form notes using canonical-messages §10.6 from 2026-04-29 spec.

**Parse — default to action, don't loop on confirmation.** Parse what you've been given into `[{name, company, conversation_notes}, ...]`. Work with what's there; flag any assumptions or gaps in the Phase 7 summary, NOT in mid-flow check-ins:

- Partial company name (e.g. "ethical something") → leave as-given; LinkedIn (Phase 2) usually resolves it.
- Cut-off sentences, vague conversation hooks → keep verbatim; the brief preserves what was actually said.
- Missing email / exact title / other fields the user can recover later → continue without; flag in Phase 7.

Show the parsed list once — briefly, no "look right?" prompt — and proceed to Phase 2. A busy executive's time is better spent reviewing the final output than confirming an intermediate parse.

**Only ask one targeted clarifying question if you are genuinely blocked from making progress** — meaning the dump is ambiguous about who the contacts are (e.g. one paragraph mentions two unnamed people and you literally cannot tell whether they're the same person or two different ones). Ask that one question, get the answer, proceed. Do NOT use clarifying questions to verify partial data, fill cosmetic gaps, or paraphrase back for confirmation.

### Phase 2 — LinkedIn research (per contact)

For each contact, invoke the embedded `linkedin-research` skill via subprocess: `python <plugin-root>/skills/linkedin-research/linkedin_scraper.py scrape "<contact name>" [--company "<hint>"]`.

**Hard cap: maximum 2 scrape attempts per contact.** Every invocation of `linkedin_scraper.py scrape` for a given contact counts toward the cap, regardless of whether the variation is a name form, a company hint, a role hint, or any combination. After the second attempt, do NOT scrape this contact again — route to the "Person not findable" failure-mode bullet below (continue with brain-dump notes only, flag in Phase 7).

**Attempt 1 — best initial hint.** Always pass a `--company` hint when you have one. The scraper picks the first search result by default, which is frequently wrong when the name is common — e.g. a common name like "Sarah Chen" can return 10+ candidates, and the unhinted scrape picks the first one, which is often the dominant search-result hit (e.g. someone at a large company sharing the name) rather than the actual contact (often someone in a niche role at a smaller company). Sources for the hint, in priority order:

1. Company name (or partial) from the brain dump — even fragments like "ethical something" work; pass `--company "Ethical"`.
2. Industry / role keyword from the dump (e.g. "client success consultant", "founder of an ML startup").
3. Any URL or other anchor mentioned in the dump.

If none of those exist, scrape on name alone for attempt 1.

**Attempt 2 (only if attempt 1 returned a dossier that contradicts the dump, OR returned no usable match at all).** Pick the single best alternative signal and re-scrape ONCE. Examples:

- Attempt 1 used `--company "Acme"` → attempt 2 drops the company hint and uses a role keyword instead, OR uses an alternate name form (initials, maiden name, etc.) if the dump suggests one.
- Attempt 1 used name only → attempt 2 adds whatever weak hint exists (industry, role, URL fragment).

Do NOT loop through multiple hint combinations. Pick one alternative and commit.

**After attempts 1+2 (or after attempt 1 if it succeeded):**

- Dossier matches the dump → continue to "Cleanup after the subprocess returns" below.
- Both attempts contradicted the dump, OR both errored with "no LinkedIn match" → route to the "Person not findable" failure-mode bullet below. Do NOT scrape a third time.

If a re-scrape (attempt 2) was used, note the cost in Phase 7 — it consumed 1 of the daily 25 LinkedIn lookups.

**Cleanup after the subprocess returns.** The script writes a **raw dossier** to `<output_dir>/<name-slug>-<timestamp>-raw.md` — messy, includes sidebar suggestions, video-player UI strings, footer chrome, and repeated author names within posts. Because we're calling the CLI directly (not loading the LinkedIn skill's body), the LinkedIn skill's own cleanup step (4c in `linkedin-research/SKILL.md`) doesn't run. Produce the cleaned file here so the output stays consistent with standalone LinkedIn-skill use:

1. Read the raw file at the returned `raw_dossier_path`.
2. Apply the KEEP / DROP rules from `linkedin-research/SKILL.md` step 4c — keep the profile owner's data (headline, About, Experience, Education, Recent posts, Search candidates considered); drop sidebar entries, language picker entries, footer chrome, video player UI, repeated author names, "X followers" lines that belong to suggestion cards.
3. Write the cleaned dossier to the same directory with the same filename minus the `-raw` suffix (e.g. `<person-slug>-<timestamp>.md`).

Phase 4 (brief synthesis) should read from the raw (richer); the cleaned file exists for the user's later reference.

Per-contact failures:
- Person not findable → continue with brain-dump notes only; flag in Phase 7 summary
- Rate limit hit (25/day cap) → save checkpoint to `<output_folder>/<event-slug>/.checkpoint.json` (contacts processed + contacts pending + timestamp); halt the loop; use canonical-messages §10.11 from 2026-04-29 spec for the resume message
- Chrome session broken / login expired → halt loop, save checkpoint, offer to re-run LinkedIn setup

### Phase 3 — Company research (per contact)

**3.1 Get homepage URL.**

In order, the first one that yields a URL wins:
1. From LinkedIn dossier if it includes a company URL
2. From the brain dump if a URL is mentioned there
3. Else: use the available web search tool with query `"[Company name] official website [industry/role hint from dossier]"` (the disambiguator is critical — a name like "PFL" alone returns the dominant brand on the web, not the small company you want). Pick top result whose domain plausibly belongs to the company.

If no clear homepage found → skip Phase 3 for this contact; brief notes "company URL not findable"; continue.

**3.2 Extract homepage content.**

Use the available web extract tool on the homepage URL. Get clean markdown content.

If extract returns thin/empty content (< 200 chars) → try one sub-page extract (see 3.3); if still thin, omit company section; flag in Phase 7.

**3.3 (Optional) Extract sub-pages.**

Parse the homepage markdown for in-domain sub-page links. Pick up to 3 sub-pages whose URL or anchor text matches one of: `/about`, `/team`, `/leadership`, `/products`, `/customers`, `/case-studies`, `/blog`, `/news`. Use the available web extract tool on each. Synthesize the homepage + sub-page content into the brief's company section.

**Do NOT use:**
- A web crawl tool with natural-language instructions (`TAVILY_CRAWL` and equivalents return empty for typical small-company / SPA-style sites — empirically validated)
- A site-map tool (`TAVILY_MAP_WEBSITE` returned empty for the same sites)

**Fallbacks:**
- Web tools not wired at all → skip company research entirely for the run; one-time note in Phase 7 summary

### Phase 4 — Brief generation (per contact)

Inputs: LinkedIn dossier + extracted company content + your conversation notes from the brain dump + your themes file.
Method: `references/atlas-conference-followup-methodology.md` (Atlas-opinionated structure).
Brief structure:

```markdown
# [Person name] — [Title], [Company]

## Person summary
[2–3 sentences from LinkedIn dossier]

## Company snapshot
**What they do:** [2–3 sentences]
**Target customer:** [who they sell to]
**Differentiation / value prop:** [what makes them stand out]
**Recent activity:** [news, launches, milestones]

## Conversation context
[Your notes from the brain dump, paraphrased cleanly]

## Common ground
[Where your themes intersect this person's work]

## Suggested follow-up angle
[The angle the email leads with]

## Source
[Event name, date]
```

Saved to `<output_folder>/<event-slug>/<person-slug>.md`.

If brief synthesis errors → auto-retry once silently. If both fail, use canonical-messages §10.7 from 2026-04-29 spec.

### Phase 5 — Email drafting (per contact)

Inputs: brief + email-voice + exec-themes + methodology.
Atlas-opinionated structure: personal opener (mentions where met + something specific from conversation notes) → substance (the common-ground angle from the brief) → soft CTA (next step) → signoff (from profile).

**Recipient resolution.** LinkedIn does NOT return email addresses (LinkedIn doesn't expose them publicly to scrapes — even in the cleaned dossier, the "Contact info" link exists but the actual email behind it is gated). Sources for the recipient, in priority order:

1. Email mentioned in the brain dump (rare but check).
2. Email surfaced in any company-page content already extracted in Phase 3 (sometimes `/about`, `/team`, `/contact` pages list one — only use what was already pulled; do NOT trigger extra extracts purely for email hunting).

If neither produces an email, **create the draft with the `to` field empty.** Drafts can be saved without recipients. Flag in Phase 7 that the recipient needs to be added before sending. Do NOT block draft creation, do NOT ask the user to provide an email mid-run.

Use the available email tool to create the draft, passing the configured account identifier from the profile (`Email tool > Account`). Drafts are ready to review and send; never auto-sent.

If voice profile missing → use generic professional tone; flag in Phase 7.
If themes missing → skip common-ground angle; flag in Phase 7.
If draft creation fails → save email content as `<output_folder>/<event-slug>/<person-slug>.email.md`; use canonical-messages §10.8 from 2026-04-29 spec.

### Phase 6 — CRM routing (per contact)

CRM target database and field mapping are captured during `setup` (step 3.2 of `setup/SKILL.md`), not here. Phase 6 reads the saved values from the profile and inserts; it does not list databases, ask which to use, or prompt for field mapping mid-run.

**Three states the orchestrator handles:**

**1. Normal path** — profile has `crm_provider` set (not `none`), AND `crm_target` (database ID) populated, AND `crm_field_map` populated with all 4 mappings.

Use the available CRM tool to insert the record with the mapped values, passing the configured CRM account identifier from the profile (`CRM > Account`). Per-contact failures inside this path:

- Single record fails (transient) → skip CRM for that contact, continue → flag in Phase 7
- Rate limit → wait + retry once, else save record details locally for manual entry; flag in Phase 7
- Auth expired mid-run → halt CRM phase for the run; finish briefs / drafts for remaining contacts; tell the user to re-auth in their MCP / CRM connection

**2. CRM provider set but target/mapping incomplete** — profile names a CRM provider but `crm_target` or `crm_field_map` is empty. Setup didn't finish the CRM step (e.g. the user had no databases at the time and skipped that sub-step).

Halt the CRM phase for the entire run (other phases continue normally). In Phase 7, deliver canonical-messages §10.6-CRM-incomplete naming the provider. Briefs and email drafts complete and save normally; the only thing skipped is the CRM record write.

**3. Stale mapping** — profile is fully populated, but the insert call returns a "field not found" / "property not found" error (a mapped field was renamed or deleted in the user's CRM since setup).

Halt the CRM phase for the run, finish briefs / drafts for remaining contacts. Use canonical-messages §10.9 from 2026-04-29 spec to surface the issue and recommend re-running setup to fix the mapping. Records that already inserted earlier in this run stay; un-inserted contacts get re-tried on the next capture-contacts call after setup is fixed.

**Skip silently** — profile says `crm_provider: none`. No record write, no Phase 7 mention (this is the configured-and-expected state).

### Phase 7 — Humanized chat summary

Reads internal per-contact result rows + brief content (for the one-line personal sentence) + failures.

Format (canonical-messages §10.10 from 2026-04-29 spec is the source of truth for wording):

```
Done — processed [N] contacts from [event name].

[Person name] — [Title], [Company]
[1–2 sentences on the person, drawn from brief]. Brief covers [what's in it,
including the conversation reference if present]. [Email tool] draft ready —
opens with [the personal opener angle], leads into [the substance angle].
Added to your [CRM] [target name].

[Repeat per contact. For failures, called out warmly inline.]

All briefs: [output_folder]/[event-slug]/
[Email tool] drafts are in your Drafts folder, ready to review and send.
```

## Resumable runs (LinkedIn rate limit)

If LinkedIn cap hit mid-run:
1. Checkpoint written to `<output_folder>/<event-slug>/.checkpoint.json` with: contacts processed (with results), contacts pending, timestamp.
2. Use canonical-messages §10.11 from 2026-04-29 spec for the rate-limit chat output.
3. The capture-contacts skill recognizes `"continue conference capture for [event]"` as a literal resume trigger phrase. On match, check for a `.checkpoint.json` matching the event slug; resume from pending contacts.
4. On full completion: delete the checkpoint.

## Output

- Per-person briefs at `<output_folder>/<event-slug>/<person-slug>.md`
- Per-person email drafts in user's email Drafts folder (or `<output_folder>/<event-slug>/<person-slug>.email.md` fallback)
- CRM records in user's CRM (or none, if `crm_provider: none`)
- Humanized chat summary (Phase 7)

## Customization

To change brief structure: edit `references/atlas-conference-followup-methodology.md`.
To change Phase 7 wording: edit canonical-messages §10.10.
To change resume trigger phrase: edit `when_to_use` above and the `Resumable runs` section.
