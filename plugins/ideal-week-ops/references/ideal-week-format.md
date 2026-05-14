# Ideal Week Document Format

The canonical schema. The `extract-ideal-week` skill writes this format; the `scan-ideal-week` skill parses it. Both depend on the structure below being stable.

## File location

Default: `client-profile/ideal-week.md` in the user's workspace.
Override: `ideal_week_path:` in `client-profile/ideal-week-ops.local.md`.

## File format

Markdown with YAML frontmatter for machine-readable fields, and structured markdown sections for human-readable content. The scan parser reads both.

## Frontmatter (required)

```yaml
---
exec_name: <string>                  # display name
extracted_at: <ISO 8601 date>        # when this version was created
extracted_by: <string>               # ea name or "self"
version: <integer>                   # increments on every save
timezone: <IANA timezone>            # e.g., "America/New_York"
workday_start: <HH:MM>               # e.g., "08:00"
workday_end: <HH:MM>                 # e.g., "16:30"
buffer_minutes: <integer>            # default buffer between meetings, e.g., 15
weekend_policy: <string>             # "off" | "on-call" | "available"
---
```

All frontmatter fields are required. The scan fails loudly if any are missing.

## Section 1 — Day-by-day rhythm (required)

One subsection per weekday (Monday through Sunday). Each subsection contains:

- **Theme** — one-line description of what the day is for
- **Time blocks** — table of time ranges with block type and activity
- **Protected blocks** — list of non-negotiable ranges within the day (these duplicate into Section 3)

```markdown
## Monday

**Theme:** Strategic alignment, leadership sync

| Time | Block Type | Activity |
|------|------------|----------|
| 7:00-8:00 | Protected | Thinking time |
| 8:00-8:15 | Recurring | EA standup |
| 8:15-9:00 | Strategic | Top priorities work |
| 9:00-10:00 | Available | Open for meetings |
| 10:00-11:30 | Recurring | Leadership sync |
| 11:30-12:30 | Protected | Lunch |
| ...

**Protected blocks:** thinking time (7-8am), lunch (11:30am-12:30pm)
```

For days that should have **zero meetings** (deep-work days), the entire day is one protected block:

```markdown
## Tuesday

**Theme:** NO-MEETING DAY — protected deep work

| Time | Block Type | Activity |
|------|------------|----------|
| All day | Protected | Deep work / strategic projects |

**Protected blocks:** entire day except 8:00-8:15am EA standup
```

Saturday and Sunday can be a single line if not worked:

```markdown
## Saturday & Sunday

**Theme:** Off (no work, no meetings)
```

## Section 2 — Recurring commitments (optional)

Anything that already exists on the calendar as a recurring event and should not be flagged. Format as a list:

```markdown
## Recurring commitments

- **EA daily standup** — Mon-Fri 8:00-8:15am (15 min)
- **Leadership sync** — Mondays 10:00-11:30am (90 min)
- **Product team sync** — Thursdays 9:00-9:30am (30 min)
- **All-hands** — Fridays 8:00-9:30am (90 min)
- **Coaching calls** — Wednesdays 9-10am and 11am-12pm, bi-weekly
```

The scan suppresses flags on events that match items in this list (by title + cadence + time).

## Section 3 — Protected blocks (required)

A consolidated list of every protected block, with severity. This is what the scan checks against. Duplicates from the rhythm section are intentional — this is the canonical enforcement list.

```markdown
## Protected blocks

| Block | Day/Time | Severity | Reason |
|-------|----------|----------|--------|
| Thinking time | Daily 7-8am | block | Personal strategic reflection |
| Deep work | Tuesday all day | block | Sacred deep-work day |
| Deep work or coaching | Wednesday all day | block | Either deep work or coaching calls |
| Lunch window | Daily 11am-1pm | block | 1-hour break, timing flexible within window |
```

## Section 4 — Rule buckets (required)

Four subsections, one per bucket. Each rule includes its severity in parentheses.

```markdown
## Rules

### ALWAYS
- Maintain 15-minute buffer between all meetings (block)
- No meetings before 9:00 AM (block)
- Block 15-minute pre-call prep for all discovery or intro calls (warning)
- Block 1-hour lunch in 11am-1pm window (block)
- Limit meetings to 3-4 hours max per day; 3 is target, 4 is ceiling (warning at 3, block at 4)

### NEVER
- No meetings on Tuesdays or Wednesdays except coaching (block)
- No back-to-back meetings without 15-min buffer (warning)
- Never exceed 4 hours of meetings per day (block)
- No work or meetings during thinking time 7-8am (block)

### PREFER
- Batch operational tasks together (nudge)
- Schedule strategic blocks early on meeting-heavy days (nudge)
- Push meetings to later in the day when possible (nudge)
- No more than 2 external calls per day (warning)

### FLEXIBLE
- Atlas clients receive priority in scheduling
- Co-founder may be scheduled as needed, including ad hoc when required
```

## Section 5 — Default meeting lengths (required)

```markdown
## Default meeting lengths

| Meeting type | Default | Scheduling window | Buffer required |
|--------------|---------|-------------------|-----------------|
| Internal 1:1 | 30-60 min | Thursday afternoons | 15 min |
| External call | 30 min | Mon, Thu, Fri | 15 min prep |
| Team meeting | 30-60 min | Mon, Thu, Fri | 15 min |
| Interview | 30 min | Mon, Thu, Fri | 15 min |
| Coaching call | 60 min | Wednesday morning | 60 min prep |
```

The scan flags meetings that exceed the default for their type without an obvious reason in the title (e.g., "deep dive", "review").

## Section 6 — VIP overrides (required, can be empty)

```markdown
## VIP overrides

| Name | Relationship | Can override | Notes |
|------|--------------|--------------|-------|
| <Name> | Co-founder, board chair | Any rule | Same-day, any time |
| <Name> | Director of Operations | Any rule | Same-day, any time |
| Atlas clients | Active client engagements | Tuesday/Wednesday rule | Same-week when necessary |
```

When the scan flags an event, it checks attendees and organizer against this list. If a VIP is involved and the rule is in their override scope, severity downgrades by one level.

## Section 7 — Zone of genius (optional but recommended)

```markdown
## Zone of genius

**Highest-and-best (only you):**
- Strategic vision and direction
- Key external partnership conversations
- Final calls on hiring senior roles

**Drains (capable but costly):**
- Meeting scheduling logistics
- Repetitive financial reviews
- Status update meetings

**Delegate or stop:**
- Calendar management → assistant
- Routine vendor calls → Director of Operations
- Recurring status syncs → convert to async written updates
```

The scan surfaces this section as context when flagging meetings that consume drain time.

## Versioning and history

Every save increments `version` in the frontmatter. The scan only reads the latest version. If you want to keep history, copy the file to `client-profile/ideal-week.v<N>.md` before re-extraction.

## Validation rules the scan enforces

When the scan parses the document, it fails loudly if any of these are violated:

- Frontmatter is present and complete
- Section 1 (rhythm) has at least Monday-Friday
- Section 3 (protected blocks) is present, even if empty
- Section 4 (rule buckets) has all four subsections, even if empty
- Section 5 (meeting lengths) is present
- Section 6 (VIP overrides) is present, even if empty

Sections 2 (recurring) and 7 (zone of genius) are optional. Everything else is required.

If parsing fails, the scan does not run — it returns the parse error and instructs the user to re-run `extract-ideal-week`.
