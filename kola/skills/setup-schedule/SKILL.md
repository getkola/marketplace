---
name: setup-schedule
description: >
  Set up a recurring Kola maintenance run — by default an hourly
  data-health sweep that de-duplicates people and flags wrongly-created
  contacts — created through Claude's own scheduler. Use when the user
  says "schedule Kola", "run Kola cleanup automatically", "keep my
  contacts clean", "automate Kola maintenance", "set up recurring data
  health", or right after customizing/installing the Kola plugin.
argument-hint: '[hourly|daily|weekly]'
---

# /kola:setup-schedule

Creates (or updates) a recurring Claude scheduled task that runs a Kola
maintenance pass on a cadence. **Default: an hourly data-health sweep**
— auto-merge provable duplicates, archive obvious junk contacts, and
queue anything uncertain for the user to review.

This skill does not do the cleanup itself. It *offers* a schedule, and on
the user's yes, hands a recurring routine to the `schedule` skill. The
actual work happens each tick, unattended.

## Read this first — the localhost-reachability rule

Kola's MCP server is local-only: it answers on `127.0.0.1:47900` **and
only while Kola.app is running on this machine**. A scheduled run that
fires when the app is closed — or that executes in a detached/cloud
environment — cannot reach it.

Two consequences you must honour:

- **Tell the user.** The hourly job is useful only when it runs in an
  environment that can reach the local app (their machine, app open).
  Say this plainly before creating the schedule. If their scheduler runs
  routines in the cloud, this will not reach Kola and is the wrong tool —
  point them at Kola's in-app Agents instead (Settings → Agents).
- **Fail quiet, not loud.** The scheduled routine (below) starts with a
  reachability probe and **stops silently** if Kola is unreachable. An
  unattended job must not error every hour while the laptop is asleep.

## Instructions

1. **Confirm intent and cadence — never create a schedule unprompted.**
   The default is **hourly**. State what you're about to create and ask
   for an explicit yes. For example:

   > "I can set up a recurring Kola data-health sweep that runs **every
   > hour** — it auto-merges clear duplicate people, archives obvious junk
   > contacts (no-reply addresses, etc.), and drops anything it's unsure
   > about into a 'Data health · review' list for you. It only runs while
   > Kola.app is open on this machine. Want me to schedule it hourly?"

   Honour an explicit cadence in the user's request (`daily` / `weekly`)
   if they gave one. Don't proceed on silence.

2. **Avoid duplicate schedules.** Before creating, invoke the `schedule`
   skill to list existing scheduled tasks. If one already runs a Kola
   maintenance sweep, **update its cadence** instead of adding a second —
   two overlapping hourly cleaners is a bug, not a feature.

3. **Create the schedule.** Invoke the `schedule` skill, asking it to run
   the routine in *"The scheduled routine"* section below at the agreed
   cadence (default: every hour). Pass the routine verbatim as the prompt
   the scheduled run should execute.

4. **Confirm back.** Report the cadence, the next run time, and how to
   change or stop it ("say 'reschedule Kola to daily' or 'stop the Kola
   schedule'"). Remind them once: it only does anything while Kola.app is
   running.

## The scheduled routine (what runs each tick)

Hand this to the `schedule` skill as the prompt to run on the cadence:

```
Kola data-health sweep. Work through the Kola MCP tools. This run is
UNATTENDED — there is no human to confirm — so apply only what the backend
marks safe, and queue or draft everything else.

0. Get the worklist. Call `get_data_health`. If it errors or Kola is
   unreachable, STOP immediately and output nothing — the app is closed;
   try again next tick. Do not report an error.

   You get back a list of `checks` — things that *look* off, NOT verified
   problems. Each is just `id`, `area` (people | company), `subjects`
   (the ids involved), and a free-text `reason`. The backend has NOT
   decided anything — there is no severity, no confidence, no suggested
   fix. That judgment is yours.

1. Investigate each check before doing anything. Pull the actual data for
   its subjects — get_person, get_person_emails, get_company, or the
   company's people. Use web search for external facts when the reason
   calls for it (a real company description; whether two company names are
   the same legal entity).

2. Decide for yourself, then act by how CERTAIN you are:
   - CERTAIN it's the same identity (two rows with an identical email,
     linkedin_url, or telegram id; or clearly the same legal entity after
     research) -> merge_people / merge_companies.
   - CERTAIN it's not a real person (no-reply / automated address, zero
     signal across every channel, no list membership, empty notes) ->
     archive_person. Never delete.
   - NOT certain (similar-but-not-identical, "probably the same", a
     plausible-looking duplicate): this run is unattended, so do NOT act.
     Add the subjects to a list named "Data health · review" (create_list
     if missing, then add_to_list) for the user to decide.
   - Missing company description: research it, then write your draft into
     the company NOTES via update_company prefixed "DRAFT (review): ".
     Do NOT overwrite the description field unattended.
   - False positive on inspection (it's fine): do nothing.

   **When you cannot judge, skip.** If you can't fetch the data you need
   (a lookup fails), web search is inconclusive, or the evidence simply
   isn't enough to be sure — do NOT guess. Leave it untouched (or queue it
   to the review list) and log it as skipped. Inaction is always the safe
   default; a wrong merge is far more expensive than a missed one.

3. Never hard-delete. Merges/archives only per step 2; notes and
   descriptions are reversible via update_company.

4. End with a RUN REPORT. This is the message someone sees when they open
   this run in the task's history, so make **planned-vs-changed** obvious.
   As you work through step 2, track for every check: what you looked at,
   what you *intended* to change, and what you *actually* did. Then output
   one table — planned change in one column, result in the next:

       Kola data-health run — <ISO date & time>

       | Check | Looked at | Planned change | Result |
       |-------|-----------|----------------|--------|
       | company-enrich-7 | Stripe — no domain/description | add domain stripe.com, set description | applied domain; description drafted to notes |
       | people-dup-12-34 | Ada L. / A. Lovelace @ Acme | merge 34 -> 12 | queued for review (not certain) |
       | people-noreply-99 | noreply@news.x, 0 signal | archive | archived |
       | company-enrich-5 | Foo Ltd — no domain | find domain | skipped (could not confirm) |

       Summary: A applied, B drafted, C queued, D skipped of N checks.

   For a merge, record the dropped row's name + emails in the "Looked at"
   cell BEFORE merging, so a bad merge can be traced. If the worklist came
   back empty, the report is one line:
   "Kola data-health run <datetime> — nothing to check."

5. Append the exact same report to `~/.kola/data-health-log.md` (create the
   folder/file if missing), under its timestamped heading — a durable
   cross-run ledger on top of this run's transcript. Always append, even
   on a no-op run.
```

## What this skill does

- Proposes a recurring maintenance cadence and, on confirmation, registers
  it with Claude's scheduler.
- Defaults to **hourly**; respects `daily` / `weekly` if asked.
- Updates an existing Kola schedule rather than stacking duplicates.

## What this skill does not do

- It does not create a schedule without an explicit yes.
- It does not run the cleanup directly — the scheduled routine does, each
  tick.
- It does not make the cleanup work when Kola.app is closed or when the
  scheduler runs detached from this machine — there is no remote Kola
  endpoint. For always-on cleanup independent of Claude, use Kola's
  in-app Agents (Settings → Agents) instead.
- It never hard-deletes people. Uncertain rows are queued, obvious junk is
  archived, only exact-identity duplicates are merged.
