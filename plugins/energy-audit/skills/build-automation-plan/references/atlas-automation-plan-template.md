# Atlas Automation Plan — Template

The opinionated template used by `build-automation-plan` to produce every Automation Plan, whether invoked from an audit or independently. Read this before customizing the template structure or the readiness rules.

## What an Automation Plan is *for*

An Automation Plan is a document that lets a different person — Atlas teammate, external builder, another agent — pick up a candidate cold, build it, and verify it works, without coming back to ask questions. Three things follow:

1. **Show the work, don't gesture at it.** A step that says "automate the lookup" is not a step. A step that says "given a contact name, query the CRM via the configured API, return the matching record's email and last interaction date, fall back to a Google search if no match" is a step. The bar is: a builder reading the plan can act, not interpret.
2. **Concrete success criteria, not vibes.** "Saves time" is not a success criterion. "Reduces the executive's weekly investor-update prep from 90 minutes to under 15 minutes, with an unchanged email being sent" is. Plans without verifiable success criteria are vapor.
3. **Scale to complexity.** A simple automation gets a simple plan — three sections, half a page. A complex one gets a complete spec — every section filled, edge cases enumerated. Don't pad. Don't truncate. Match the artifact to what the work actually requires.

## The solution-form decision (required first step)

Before filling in any other section, answer this question and write the answer in the plan: **what form does the solution take?**

> **Design for the optimal form, not the confirmed one.**
> The audit only sees the calendar and task tool. The full stack is unknown. Pick the form that fits the work best, then surface what needs to be true in the Wiring & Prerequisites section. Narrowing to the safest option because tools aren't confirmed produces weak plans — the builder validates and adjusts.

The valid forms are:

| Form | When to choose | Example |
|---|---|---|
| **Skill** | A repeatable workflow that runs inside an existing agent runtime, triggered by phrase or schedule. The work is multi-step but stays in one conversational context. | "Generate the weekly metrics post" — fetch data, draft post, ask for sign-off, send. |
| **Plugin** | A bundle of skills + supporting files + integrations. Use when the workflow has multiple sub-skills (onboarding, the main action, a refresh path) or when configuration needs to live alongside the skills. | "Investor reporting plugin" with onboarding-investor-cadence + draft-investor-update + send-investor-update skills. |
| **Micro-app** | A standalone web or local app with its own UI. Use when the workflow needs persistent state across sessions, multiple users, or interactive review of complex data that markdown can't show. | "Conference contact triage app" — drag-and-drop names from voice notes, surface enriched profiles, batch-send to CRM. |
| **Workflow integration** | A glue automation between two SaaS tools, often runnable in n8n / Zapier / Make / native automation features of an existing tool. The AI's role is the transformation step, not the orchestration. | "When a Notion task tagged 'follow-up' is created, draft a Gmail reply using the linked context, save as a draft." |
| **Automation script** | A standalone script (Python, Node, Apps Script) that runs on a schedule or trigger. Use when the work is mechanical enough that conversation overhead would slow it down. | "Weekly: pull last week's calendar to a sheet, format by category, email the CSV." |
| **Prompt template** | A reusable prompt the executive or EA invokes inside their existing AI tool. Use when the work is genuinely a one-shot judgment that varies per occurrence and needs no orchestration or triggering. Do not default to this form because other tools aren't confirmed — only choose it when the work is inherently one-shot. | "Strategic positioning memo template — paste the meeting transcript, produce a one-pager in Atlas voice." |

If the answer is "it depends" or "could be a few of these," pick the optimal one and name the alternative in the plan with a one-sentence reason why it was not chosen. Don't write a plan that's neutral about its form — builders need to know what they're building.

If no form fits (the candidate genuinely needs a non-AI solution), say so and abandon the plan. A plan that admits it's the wrong tool is more useful than a plan that pretends to fit.

## Required template structure

Every Automation Plan written by this skill has these sections, in this order. Section content scales to complexity — a simple plan can have one-line sections; a complex plan can have full subsections.

### 1. Goal (one sentence)

Single sentence describing what the automation accomplishes from the executive's point of view. Not what it *is* — what it *does for them*.

- **Good:** "Reduce the executive's weekly investor-update prep from 90 minutes to under 15 minutes by drafting the update from latest metrics and last week's narrative, with the executive only reviewing and sending."
- **Bad:** "Build a tool that helps with investor updates." *(too vague — what does "help" mean, what's the bar)*
- **Bad:** "Use AI to automate investor reporting." *(describes what it is, not what it does)*

### 2. Solution form

The optimal form for this work (Skill / Plugin / Micro-app / Workflow integration / Automation script / Prompt template), with one to three sentences explaining why this form fits the work best. The choice is based on what the automation should do — not on which tools are confirmed to exist in the stack today. If the candidate could plausibly take two forms, name the alternative and explain in one sentence why the chosen form was preferred.

### 3. The problem today

Two to four sentences describing the manual workflow as it currently happens — pulled from the candidate brief in the audit, plus any context the brief carried.

- What triggers the work today
- What the executive (or whoever owns it now) does step by step
- Where the time goes
- Why the current shape is suboptimal (drain, repetition, error-prone, blocking other work)
- *When the candidate came from an audit, cite the source(s) and evidence strength as observed by the audit (e.g., "observed in 4 Slack threads + 2 calendar events; evidence: moderate"). Don't bury this — the builder reading the plan needs to know how confident the audit was.*

Specifics matter. "Investor reporting takes too long" is not a problem statement. "Every Monday morning, the executive pulls metrics from three dashboards by hand, copies them into a Google Doc template, writes a 200-word narrative, formats the doc, exports as PDF, and emails to the investor list. The metrics-pulling alone is ~45 minutes" is a problem statement.

### 4. The proposed automation

Three to six sentences describing what the automation does end-to-end.

- **Trigger** — what event or schedule starts it.
- **Inputs** — what data it reads and from where.
- **Processing** — what it does with the inputs (drafting, summarizing, looking up, transforming).
- **Outputs** — what it produces and where it lands.
- **Human checkpoints** — where the executive (or another human) reviews/approves before final action.

This section is the spec. Writing it well is the difference between a plan that gets built and a plan that gets re-asked.

### 5. Capabilities required

Bulleted list of the technical capabilities the automation needs, named abstractly (not by tool):

- Read calendar events for a date range
- Send email on the executive's behalf
- Query a task tool's API filtered by assignee
- Write a structured markdown document to a chosen folder
- Render a chart from numeric data

This list maps to "what the builder needs to wire up." Don't name specific tools yet — that's for the next section.

### 6. Wiring & Prerequisites

This section answers two questions in one place: which tools handle each capability, and what needs to be true for the plan to work. Both subsections are required. Plans missing either are not finalizable.

#### Wiring

For each capability listed in section 5, name the optimal concrete tool or service. The wiring describes what would best do the job — the audit's known tools (calendar, task) are not the constraint. Tools confirmed in the energy profile are tagged **Confirmed**. Tools the audit has no view into are tagged **Validate** so the builder can check.

| Capability | Suggested wiring | Status |
|---|---|---|
| Read calendar events | Google Calendar via Composio | Confirmed |
| Send email | Gmail via Composio | Confirmed |
| Trigger on new CRM record | HubSpot / Pipedrive / Notion (Composio or native webhook) | Validate |
| Render chart | Python in build environment, output PNG | No integration required |

Do not silently drop a capability because the profile doesn't list its tool. Flag it as **Validate** and continue.

#### Prerequisites

What needs to be true for this plan to work — beyond just tool availability. Prerequisites cover three kinds of things, and a useful list usually includes more than one kind:

- **Tool / connection** — a specific integration, scope, or auth that must be in place
- **Trigger / event** — a defined event in some system that the workflow can fire on
- **Process / data condition** — something about how the work happens or how data is shaped that must hold (e.g., calendar events have descriptive titles; the EA reviews drafts before send; client intake captures specific fields)

Each prerequisite uses one of three labels:

- **Required** — non-negotiable for the plan to work as designed
- **Assumed** — the plan presumes this is true; if it isn't, behavior changes
- **Fallback if [X] is not met** — what to build or do instead

Every plan has at least one prerequisite. A list with only tool prerequisites usually means the broader requirements weren't thought through — re-examine before finalizing.

**Examples (mixed kinds):**
- *"Required (trigger): A 'new client signed' event must exist in the stack — a CRM record creation, an intake form submission, or a manual signal the EA can fire. Without one, the workflow has nothing to fire on."*
- *"Required (process): The EA reviews each generated draft before send. The plan assumes a human-in-the-loop and is not safe to wire as fully autonomous."*
- *"Assumed (data): Calendar events use descriptive titles ('Q2 strategy review') rather than placeholders ('Block'). If titles are sparse, categorization quality drops and the builder should add a manual review step."*
- *"Fallback if no CRM trigger exists: build the workflow as a manually-fired skill the EA invokes with the client name. Upgrade to event-triggered when a CRM is added."*

### 7. Build plan

Three to six concrete steps a builder can execute. Each step:

- **Names the artifact being created or modified** (file path, integration, sheet, script).
- **Names the tool or method** being used.
- **Has a verifiable end-state** ("the script runs locally and prints the formatted output," "the skill responds to the trigger phrase with a draft").

Steps should be sized so that each takes a builder under a few hours of focused work. If a step is "build the integration," break it down further — "set up auth," "test the read endpoint with a known input," "wire the read into the draft step."

For plans whose solution form is "Skill" or "Plugin," include a step like:

> *Use Anthropic's standard skill-authoring practices for the SKILL.md — trigger-rich description, when_to_use with three or more concrete trigger phrases, imperative skill body. If the `superpowers` plugin is installed, run the build through `superpowers:writing-plans` for the planning pass and `superpowers:test-driven-development` for the implementation pass — these produce higher-confidence builds. Without superpowers, follow the same discipline manually.*

This is the soft-dependency hook. The plan still works without superpowers; it works better with it.

### 8. Success criteria

A bulleted list of three to five verifiable conditions. Each must be objectively checkable — by reading the artifact, running it on a known input, or comparing before/after metrics.

- **Good:** "When the executive runs the trigger phrase 'draft this week's investor update,' the skill produces a complete draft (subject + body + chart) within 90 seconds, using the latest metrics from the configured sources, in the voice of the previous three updates."
- **Good:** "Time spent by the executive per occurrence drops from a baseline of 90 minutes to under 15 minutes, measured over four consecutive runs."
- **Bad:** "The automation works correctly." *(unverifiable)*
- **Bad:** "It saves time." *(unmeasured)*

### 9. Edge cases

Bulleted list of three or more concrete failure scenarios the builder should handle, with the expected behavior. Don't write "handle edge cases" — name them.

- "What happens when a metric source returns no data for the period? — The draft still produces, with a placeholder for that metric and a flag in the body that the metric is missing."
- "What happens when the executive runs the trigger twice in the same week? — The second run produces a fresh draft (does not error), but flags in the response that an update has already been sent this week."
- "What happens if the email send fails? — The draft is preserved in the configured output folder and the executive is notified with the error and the draft path."

### 10. Effort estimate

S / M / L bucket with a one-line rationale.

- **S** (about a day) — single-step automation, existing connections, minimal testing.
- **M** (about a week) — multi-step workflow, possibly new connection, real testing across edge cases.
- **L** (about a month) — multi-skill plugin or micro-app, new architecture, ongoing iteration with the executive.

Be honest. An L plan that says S to look easier produces broken commitments and lost trust.

### 11. Build companion (optional)

One sentence noting that the `superpowers` plugin (`/plugin install superpowers`) is recommended for the build phase but not required. Skip this section in plans whose solution form is Workflow integration, Automation script, or Prompt template — superpowers is for skill/plugin/app builds.

### 12. Quarterly priorities footnote (conditional)

If the candidate brief's `quarterly_stale: true` flag was set:

> *"This plan was generated using the executive's quarterly priorities last updated in Q[X]. We're now in Q[Y]. The plan's priority alignment may not match the current quarter's actual priorities. Refresh the energy profile (`refresh my energy profile`) for sharper alignment in next month's audit."*

Otherwise, skip this section.

---

## Anti-patterns the template forbids

The skill rejects plans that contain any of the following. If a draft contains any of these, revise before finalizing:

- **"TBD" / "TODO" / "fill in later" / "similar to above" / "see implementation."** Every section is filled with concrete content or explicitly omitted with a stated reason.
- **Vague success criteria.** "Works correctly," "saves time," "looks clean," "is reliable." Replace with verifiable conditions.
- **Vague problem statements.** "Takes too long," "is annoying," "doesn't scale." Replace with specifics: how often, how long, what the steps are.
- **Steps that describe what without showing how.** "Build the integration," "automate the workflow," "use AI to generate the report." Break down to action level.
- **Padding to length.** A one-page plan is fine if the work is one page. Don't pad to look thorough.
- **Hedging language as substitute for thought.** "Could perhaps consider potentially" — replace with a recommendation and why, or with a stated alternative.
- **Single-option proposals when alternatives exist.** If the solution form could plausibly be two things, name the rejected option and why. Single-option plans without trade-offs surfaced are a tell that thinking didn't happen.
- **Premature scope expansion.** A plan for "investor reporting" that secretly contains a plan for "investor reporting + board prep + fundraising pipeline" is four plans. Split or scope down.
- **"Handle edge cases" as a phrase.** Edge cases are enumerated by name with expected behavior. The phrase itself is forbidden.
- **Narrowing the solution form because tools aren't confirmed in the profile.** The audit sees only the calendar and task tool; the rest of the stack is unknown. Pick the optimal form for the work and surface what needs to be true in section 6. Defaulting to "prompt template" because no CRM is listed is a tell that the form decision was made by elimination, not by fit.
- **Wiring & Prerequisites section that is empty, or that lists only tool prerequisites.** Every plan has at least one prerequisite, and most have non-tool ones (a trigger event, a process step, a data condition). A list with only tool entries usually means the broader requirements weren't thought through.

## Voice and craft

Plans are written in direct, declarative prose. No marketing voice. No hype. The reader is a builder who will execute, not a buyer who needs to be sold. Specifics over adjectives.

- **Forbidden:** "robust," "seamless," "powerful," "next-generation," "revolutionary," "best-in-class."
- **Forbidden:** "leverage," "enable," "empower" (when used as filler verbs).
- **Preferred:** "do," "produces," "writes," "reads," "sends," "fails when X, falls back to Y."

Plans address the builder, not the executive. The executive is the customer of the *automation* — the plan's audience is whoever will build it.

## When to abandon a plan mid-draft

If, while drafting, you discover any of the following, stop and report rather than finishing the plan:

- The solution-form decision is genuinely impossible — no form fits, the work is human-only.
- The capabilities required don't exist in current AI tooling (e.g., requires a level of judgment AI cannot reliably provide).
- The candidate's manual workflow can't be described concretely enough to automate — the executive's process is implicit and varies per occurrence.
- The "buildable" criterion fails on inspection — what looked like an automation candidate turns out to need ten months of model training.

In any of these cases, the audit's automation-readiness check failed in retrospect. Replace the plan section in the report with: *"On closer inspection, this candidate did not meet the automation-readiness criteria. Reason: [specific reason]. Not building a plan."* This is honest output. It's better than a plan no one will trust.
