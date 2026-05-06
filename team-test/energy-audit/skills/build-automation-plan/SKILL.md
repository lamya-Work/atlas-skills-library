---
name: build-automation-plan
description: Takes a single candidate — either passed from a running energy-audit invocation or described inline by the user — and produces a complete Atlas Automation Plan. The plan picks a solution form (Skill / Plugin / Micro-app / Workflow integration / Automation script / Prompt template), fills the twelve required sections from the Atlas template, applies anti-pattern checks during draft, and either appends to the audit report file (when invoked by an audit) or saves as a standalone plan file (when invoked independently).
when_to_use: Invoked automatically by `energy-audit` for each top candidate after the audit report is written. Also triggerable independently for ad-hoc planning. Trigger phrases for independent use — "build an automation plan for [X]", "spec out automating [X]", "draft an automation plan", "plan an automation", "write an automation spec for [X]". Do not invoke this skill before `extract-energy-profile` has produced an exec energy profile — the plan needs the profile for capability wiring and tool naming.
atlas_methodology: opinionated
---

# build-automation-plan

Take a candidate, choose a solution form, fill the twelve template sections, write the plan to the right place.

## Purpose

Produce one complete Automation Plan per invocation. The plan is concrete enough for a builder — Atlas teammate, external developer, another agent — to pick up cold and execute without coming back to ask questions. Two invocation paths share the same drafting logic; only the output destination differs.

## Inputs

This skill has two invocation paths. The skill body branches in step 0 based on which path is active.

### Path A — from audit (auto-invoked)

The energy-audit skill passes:

- **Candidate brief** — task name, frequency, time per occurrence, profile alignment, manual workflow description, and any pattern notes. The brief also includes three fields added by the audit's dedup pass:
  - `source` — string. One of `"calendar"`, `"tasks"`, `"meetings"`, `"slack"`, or a multi-source string like `"meetings + slack"` (when the audit's dedup pass merged candidates from two sources). Indicates which connected source(s) surfaced the candidate.
  - `source_refs` — array of strings. Per-source references — calendar event IDs, task IDs, transcript IDs, or Slack message permalinks. Used to cite specific evidence in plan section 3 ("The problem today").
  - `evidence_strength` — one of `"strong"` / `"moderate"` / `"weak"`. The audit's confidence in the pattern signal:
    - `strong` — observable in multiple places or has explicit recurrence markers
    - `moderate` — appears recurring but only in one source
    - `weak` — single-mention pattern with limited evidence

  The skill body uses these fields to cite the audit's evidence in plan section 3 ("The problem today") — see the plan template at `references/atlas-automation-plan-template.md` for guidance on phrasing. Don't bury the source attribution; the builder reading the plan needs to know how confident the audit was.

- **Full energy profile content** — used to wire capabilities to the executive's actual tools
- **Report file path** — `[output-folder]/[YYYY-MM-DD]_energy-audit.md` written by the audit
- **Quarterly-staleness flag** — `quarterly_stale: true | false` and `last_updated_quarter` if applicable

### Path B — independent (user-triggered)

The user provides:

- **A description of the task to plan an automation for** — captured inline through conversation. The skill asks for: the task name, how often it happens, what the manual steps are today, what triggers it, what the desired output is.
- **Optional reference to a past audit** — *"plan an automation for the second candidate in my last audit"* — in which case the skill reads `[output-folder]` from the profile, finds the most recent or named audit file, extracts the candidate brief, and proceeds.

## Required capabilities

- **File read** — read the methodology and template references; read the energy profile; read the audit report file (Path A) or a referenced past audit file (Path B optional)
- **File read-modify-write** — for Path A, read the current audit report file in full, insert the new plan into the `## Automation Plans` section without disturbing other sections, and write the file back. Atomic where possible to avoid partial writes.
- **File write** — for Path B, write a new standalone plan file to the output folder
- **Conversational interview** — for Path B, capture the task description through a short structured conversation if the user does not provide all the brief fields up front

## Steps

### 0. Path detection

Detect which invocation path is active:

- If invoked with a candidate brief and a report file path passed in (Path A) → go to step 1.
- Otherwise (Path B) → go to step 1A first.

### 1A. Capture the brief inline (Path B only)

If the user provided a past-audit reference, read that audit file from the output folder and extract the candidate brief from the relevant `### Candidate N` section. Confirm with the user: *"You're asking me to plan the candidate `[task name]` from `[audit date]`. Correct?"*

Otherwise, walk through the brief fields one at a time:

1. *"What's the task you want to plan an automation for?"*
2. *"How often does it happen — recurring, or triggered by an event? What's the trigger or pattern?"*
3. *"Walk me through what you (or whoever owns it) does today, step by step."*
4. *"What's the desired output? What does 'this is done' look like?"*
5. *"Roughly how much time does it take per occurrence?"*

Capture answers as a structured brief. Paraphrase back before continuing: *"So the candidate is `[summary]`. Going to plan it. Stop me if I have it wrong."*

### 1. Read references and profile

> **Note on "workspace".** "Workspace" = the user's current working directory at the time the skill is invoked. The audit report file (Path A: passed in by `energy-audit`; Path B: created by this skill at the path the user picks) and any user-supplied profile resolve relative to this workspace, NOT the plugin install directory. Plugin-internal references (the plan template at `references/atlas-automation-plan-template.md` and any methodology docs) use `../../` from this skill folder to mean plugin root. If a passed-in path doesn't exist, surface that as an error rather than falling back to the plugin folder.

Load:

- `references/atlas-automation-plan-template.md` — the full template structure, the twelve required sections, the anti-pattern list, and the voice rules
- `../../client-profile/templates/exec-energy-profile.template.md` (plugin-relative) — for context on what the profile looks like (the actual profile content is provided in inputs for Path A, or read from the workspace-relative `client-profile/exec-energy-profile.md` for Path B)
- The energy profile content itself (Path A: passed in; Path B: read from disk at the workspace-relative path)

### 2. The solution-form decision

Before drafting any other section, decide what form the solution takes. Pick exactly one from the template's six valid forms:

- **Skill** — repeatable workflow inside an existing agent runtime, triggered by phrase or schedule
- **Plugin** — bundle of skills + supporting files + integrations
- **Micro-app** — standalone web or local app with its own UI
- **Workflow integration** — glue between two SaaS tools (n8n / Zapier / Make / native automations)
- **Automation script** — standalone script (Python, Node, Apps Script) on schedule or trigger
- **Prompt template** — reusable prompt the executive or EA invokes inside their existing AI tool

**Pick the form that fits the work, not the form that fits the confirmed tool stack.** This skill only knows the calendar and task tool from the profile — the rest of the executive's stack is unknown. Choosing the safest form because a tool isn't listed is an anti-pattern (defaulting to "prompt template" because no CRM is listed is the canonical example). Pick the optimal form, then surface what needs to be true in section 6 (Wiring & Prerequisites). The builder validates from there.

Use the criteria in the template reference (`references/atlas-automation-plan-template.md` → "The solution-form decision"). If the candidate could plausibly take two forms, pick the optimal one and note the alternative with a one-sentence reason in the plan's section 2.

If no form fits — the work genuinely needs a non-AI solution — go to step 8 (honest abandon).

### 3. Draft the twelve sections

Walk through the template sections in order. Each section's content is filled per the rules in `references/atlas-automation-plan-template.md`. Sections are required:

1. **Goal** (one sentence — what the automation does for the executive)
2. **Solution form** (the optimal form from step 2 + 1–3 sentences of why; name the alternative if one was rejected)
3. **The problem today** (manual workflow, pulled from the candidate brief and expanded)
4. **The proposed automation** (trigger, inputs, processing, outputs, human checkpoints)
5. **Capabilities required** (abstract — what the builder needs to wire)
6. **Wiring & Prerequisites** (two subsections in one — the wiring table with Confirmed/Validate status, and a Prerequisites list covering tool, trigger/event, and process/data conditions)
7. **Build plan** (3–6 concrete steps, each with artifact + tool/method + verifiable end-state)
8. **Success criteria** (3–5 verifiable conditions, no "works correctly")
9. **Edge cases** (3+ concrete scenarios with expected behavior, named — not "handle edge cases")
10. **Effort estimate** (S / M / L with a one-line rationale)
11. **Build companion** (one-sentence superpowers note for skill/plugin/app builds; skip for the other forms)
12. **Quarterly priorities footnote** (only if `quarterly_stale = true`)

For section 6, both subsections (Wiring and Prerequisites) are required. A wiring table without prerequisites, or prerequisites that contain only tool entries (no trigger/event or process/data conditions), fails the anti-pattern check.

For section 7, when the solution form is `Skill` or `Plugin`, include the standard step about Anthropic skill-authoring practices and the optional superpowers companion (exact phrasing in the template reference). For other forms, the build plan steps map to that form's typical build path (script: write, test, schedule; workflow: configure, test, deploy; etc.).

### 4. Anti-pattern check

Before finalizing the draft, scan for the anti-patterns the template forbids:

- "TBD" / "TODO" / "fill in later" / "similar to above" / "see implementation"
- Vague success criteria ("works correctly," "saves time")
- Vague problem statements ("takes too long," "is annoying")
- Steps that describe what without showing how
- The phrase "handle edge cases" instead of named scenarios
- Forbidden voice words ("robust," "seamless," "leverage," "empower," "next-generation," etc. — full list in the template reference)
- Single-option proposals when alternatives exist
- Premature scope expansion (the plan secretly contains two or more plans)
- Solution form picked by elimination (e.g., defaulting to "prompt template" because no CRM is confirmed) instead of by fit
- Section 6 with no Prerequisites subsection, or Prerequisites containing only tool entries (no trigger/event or process/data conditions)

If any anti-pattern appears, revise the offending section before writing the plan. This is not a soft check — anti-patterns block finalization.

### 5. Write the plan

Branch on path:

#### Path A — append to the audit report

1. Read the full content of `[report file path]` into memory.
2. Locate the `## Automation Plans` section. The first time this skill writes, the section contains the literal text `(pending)`. Subsequent writes (if there are two top candidates) find the section already populated with the first plan.
3. Replace `(pending)` with the new plan content, OR append the new plan content under the previous plan if `(pending)` is no longer there.
4. The plan content is wrapped in a subsection: `### Plan: [Candidate task name]` followed by the twelve template sections.
5. Write the full file back. Do not modify any other section of the report.
6. Read the file back one more time to confirm the new plan section is present and well-formed (the candidate name appears as a `### Plan:` heading inside the `## Automation Plans` section).

If the read-modify-write fails — file lock, write error, the section structure is malformed — return an error to the calling audit skill rather than silently producing partial output.

#### Path B — write standalone

Compute the file path: `[output-folder]/[YYYY-MM-DD]_[task-slug]_automation-plan.md` where YYYY-MM-DD is the run date and `task-slug` is a kebab-cased version of the task name (lowercase, spaces to hyphens, drop non-alphanumeric).

Write the plan as a complete standalone document. Top-level header is `# Automation Plan: [Task name]`. Followed by the twelve template sections. Followed by a footer noting the date and that this plan was generated independently (not from an audit).

### 6. Confirm and report

Path A: return a one-line confirmation to the calling audit skill — *"Plan appended for `<candidate>` to `<report path>`. Length: [N] sections."*

Path B: print the path to the standalone plan file to the user — *"Plan written to `<path>`. Open the file to review."*

### 7. (Reserved — no action)

### 8. Honest abandon

If at any point during drafting the candidate fails on closer inspection — the solution-form decision yields no fit, the capabilities required don't exist in current AI tooling, the manual workflow can't be described concretely enough to automate, or the buildable criterion fails — stop drafting.

Path A: instead of appending a plan, replace the candidate's plan section with: *"On closer inspection, this candidate did not meet the automation-readiness criteria. Reason: `<specific reason>`. Not building a plan."* Return to the calling audit skill with this outcome rather than a fabricated plan.

Path B: tell the user directly: *"On closer inspection, the candidate doesn't meet the automation-readiness bar — `<specific reason>`. Not building a plan. The candidate may still be the right thing to fix, but it needs a non-AI approach."* Do not write a file.

A plan that admits it's the wrong tool is more useful than a plan that pretends to fit.

## Output

- **Path A:** the audit report file at the path passed in is updated with the new plan section. Skill returns a confirmation line to the audit.
- **Path B:** a new standalone file is written to the output folder. Skill returns the path to the user.

In both paths, when honest abandon triggers, the skill produces no plan content — only the abandon message and reason.

## Customization

- **The twelve template sections.** Defined in `references/atlas-automation-plan-template.md`. Edit the template to add, remove, or reorder sections — the skill body reads section requirements from there.
- **The solution-form list.** The six valid forms are defined in the template reference. Adding a seventh form (e.g., "Browser extension," "CLI tool") requires editing the template and adding the form to the decision logic in step 2.
- **The anti-pattern list.** Forbidden phrases and voice words are in the template reference's "Anti-patterns" and "Voice and craft" sections. Edit there to change.
- **Filename pattern (Path B).** Default `YYYY-MM-DD_[task-slug]_automation-plan.md`. Match the audit's filename convention for consistency. Change in both skills together if customizing.
- **Build companion (superpowers note).** Soft dependency. The plan still works without superpowers installed. Edit the template to remove the note or change the recommended companion.

## Why opinionated

The twelve-section structure, the solution-form decision as the gating step, the anti-pattern blocklist, and the voice rules are Atlas's specific approach to writing Automation Plans. The full template lives in `references/atlas-automation-plan-template.md`. Clients wanting a different plan structure should fork the template — keep the skill body stable.
