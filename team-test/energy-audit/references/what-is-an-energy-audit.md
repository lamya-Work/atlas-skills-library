# What is an Energy Audit?

> Read this first. The plugin's setup and audit skills assume this shared definition — the term means different things to different people, so it's worth being explicit before anything else runs.

## The short version

An **energy audit** is a bi-weekly review of an executive's trailing 14 days — calendar events, tasks, meeting transcripts, and Slack messages (any combination, at least one) — that identifies repetitive operational work an AI can take off their plate. It produces a small number of complete Automation Plans (one or two, never more than two), each ready to hand to a builder.

It's not a time tracker. It's not a wellness assessment. It's not a generic productivity report. It's a focused exercise: find what's repetitive, filter through what the executive said drains them or supports their quarterly priorities, pick the strongest one or two candidates, write a real plan for each.

## What it does

A complete audit produces one file: `[output-folder]/YYYY-MM-DD_energy-audit.md`. That file contains six sections:

1. **Header** — the audit window, the mode (full or short-window), the connected sources used, profile completeness, and a banner if quarterly priorities are stale.
2. **Signals breakdown** — per-source one-line summary of volume and shape (e.g., *"Calendar: 47 events, ~23 hours"*, *"Slack: 312 messages across 3 selected channels"*), plus a 2–4 line prose summary of what concentrated over the period.
3. **Drain alignment** — which observed signals across the connected sources match the drains the executive named in onboarding.
4. **Automation candidates table** — every candidate evaluated, with source attribution, evidence strength, and score, ending with the explicit selection line for the top one or two.
5. **Top candidates** — the candidates that passed the automation-readiness criteria, with a structured brief for each (source, evidence strength, frequency, manual workflow, time cost, profile alignment, notes).
6. **Automation Plans** — one full plan per top candidate, written by the build-automation-plan skill and appended into this same file.

That's the whole output. One file per audit run. A complete, durable artifact the executive can read in fifteen minutes and a builder can act on without asking questions.

## What it isn't

- **It isn't a measure of real energy levels.** The plugin can't tell how the executive actually feels. It uses what the executive said in onboarding (Zone of Genius, drains, recurring patterns) as a stated filter. It does not infer energy from calendar patterns.
- **It isn't a comprehensive task review.** Tasks that should be delegated to a person, or eliminated entirely, are out of scope for this plugin. The output is automation candidates only. Other categories of work the audit notices are not surfaced.
- **It isn't continuous.** Run every two weeks. Default cadence is every 14 days. Running more often produces noisier candidates — repetitive patterns need two weeks of data to be visible.
- **It isn't optional in scope.** The audit is conservative on purpose. One or two strong candidates per run is the target. If the audit can't find one strong candidate, it says so and produces no plan rather than fabricating one.

## What "automation" means here

Broad. The Automation Plan can recommend any AI-buildable solution: a skill, a plugin, a micro-app, a workflow integration, an automation script, a prompt template, or a combination. The build-automation-plan skill judges what fits the candidate. The first analytical step in every plan is "what form does the solution take?" — answered before the rest of the plan is filled in.

What this rules out: human-only solutions, hardware, organizational restructuring, hiring. Those aren't AI-buildable, so they don't get plans. They might still be the right answer for a given task, but this plugin won't write the spec for them.

## Conservative output is the principle

The audit is held to a hard rule: **at most two Automation Plans per run.** Even if the audit finds five candidates that look automatable, only the top one or two pass.

The reasoning: builders ship. Five plans on a list that nobody builds are worth less than two plans that get built. Atlas's experience running these audits is that executives and builders both ignore long lists. A short list with high-confidence candidates moves work forward.

If only one candidate passes the automation-readiness criteria, the audit produces one plan. If none pass, the audit reports the time/energy breakdown and drain alignment without plans, and notes that no candidate met the bar this run.

## How the audit uses the energy profile

The audit reads two things from the exec energy profile:

- **Drains** — candidates that match a drain the executive named get higher weight in the ranking.
- **Quarterly priorities** — candidates that support a stated quarterly priority get bonus weight, and the audit flags the report if priorities are stale (a different calendar quarter from `Quarterly priorities last updated`).

The profile is a filter, not a constraint. A strong automation candidate that doesn't match a stated drain or priority can still pass — but candidates that match drain or priority alignment, all else equal, rank higher.

## What happens after the audit

Once the audit completes:

1. The file is saved to the configured output folder.
2. A real notification is sent to the configured email channel — Gmail or Outlook only. The notification includes a one-line summary and the file path. (If the executive set `notification_channel: none` at setup, the report file is still written but no email is sent.)
3. The executive reads the report at their convenience. The Automation Plans inside are complete enough that a builder (the EA, an Atlas teammate, an external developer, or another agent) can pick one up and start.
4. If quarterly priorities are flagged as stale, the executive can run `refresh my energy profile` to update them before the next audit.

The plugin does not auto-build the automations it plans. It produces the spec. Building is a separate decision the executive makes.

## A note on "audit"

The word "audit" can sound bureaucratic. It's not. The point is to spend thirty minutes reading a bi-weekly artifact, decide if one or two automations are worth building, and either greenlight them or set them aside. The audit's job is to make that thirty-minute review productive — by being conservative about what makes the cut, complete about what the candidate looks like, and direct about what would need to be built.
