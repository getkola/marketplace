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
   > hour** — it enriches companies (descriptions, domains), merges clear
   > duplicates, archives obvious junk contacts, and acts only when it's
   > confident. It only runs while Kola.app is open on this machine. Want
   > me to schedule it hourly?"

   Honour an explicit cadence in the user's request (`daily` / `weekly`)
   if they gave one. Don't proceed on silence.

2. **Find an existing Kola schedule — update, don't duplicate.** The
   scheduled task has a fixed title: **`Kola data-health sweep`**. Invoke
   the `schedule` skill to list scheduled tasks and look for that title.
   If it exists, you are **updating** it, not creating a second one.

3. **Create or update the schedule** via the `schedule` skill, always
   under the title `Kola data-health sweep`:
   - Set the cadence to the agreed value (default hourly).
   - **Use the cheapest capable model.** This sweep is mechanical work —
     read a row, check a fact, apply a reversible edit or skip — not deep
     reasoning, so it does not need a frontier model and running it on one
     every tick is the main cost. When the scheduler exposes a model
     setting, pick the cheapest model that can still follow the routine:
     **prefer Haiku; fall back to Sonnet** if Haiku isn't selectable.
     Never schedule this on Opus. If the scheduler offers no model knob,
     say so in the confirm-back so the user can set it themselves — don't
     silently let it inherit an expensive default.
   - Set the prompt to the routine in *"The scheduled routine"* below,
     **verbatim and in full** — even when updating an existing schedule.
     The cron stores a *snapshot* of the routine text, so refreshing the
     prompt is the only way a plugin update to the routine reaches an
     already-scheduled task. (Updating just the cadence would leave the
     old logic running.)

4. **Confirm back.** Report the cadence, the model the run uses (or that
   the user must pick a cheap one manually if the scheduler has no model
   setting), the next run time, and how to change or stop it ("say
   'reschedule Kola to daily' or 'stop the Kola schedule'"). Remind them
   once: it only does anything while Kola.app is running.

## The scheduled routine (what runs each tick)

Hand this to the `schedule` skill as the prompt to run on the cadence:

```
Kola data-health sweep. Work through the Kola MCP tools. This run is
UNATTENDED and NOBODY reviews it afterwards — so there is no point drafting
notes or queueing rows for later. The rule is binary: when you are
confident, apply the real change; when you are not, skip it. Nothing in
between.

0. Get the worklist. Call `get_data_health`. If it errors or Kola is
   unreachable, STOP immediately and output nothing — the app is closed;
   try again next tick. Do not report an error.

   You get back a list of `checks` — things that *look* off, NOT verified
   problems. Each is just `id`, `area` (people | company), `subjects`
   (the ids involved), and a free-text `reason`. The backend has NOT
   decided anything — there is no severity, no confidence, no suggested
   fix. That judgment is yours.

   The worklist is prioritised and self-advancing, and ONE tick does ONE
   kind of work: the batch you get back is either all `company-enrich-<id>`
   (companies missing a domain/description) OR all `people-archive-<id>`
   (contacts surfaced for archive review) — the two kinds alternate from
   tick to tick, so don't expect both in the same run. Just work the batch
   you got; the next tick does the other kind and picks up where it left
   off. The people batch is ordered worst-first: role/automated junk
   addresses, then contacts with no engagement at all, then active contacts.

1. Investigate each check before doing anything. Pull the actual data for
   its subjects FIRST — get_person, get_person_emails, get_company, or the
   company's people. Decide from that internal data whenever you can.

   Web search is the most expensive thing you do here, so ration it
   HARD. Search only when an external fact is genuinely required (a real
   company description; whether two company names are the same legal
   entity) AND the internal data can't settle it. When you do search:
   **at most 2 web operations per check** — prefer a single fetch of the
   company's own site (its domain), and at most one fallback search. If
   it's still unconfirmed after that, STOP and skip the row; do not keep
   searching. Never run a multi-query research loop on one company — one
   skipped row is cheaper than ten searches, and it'll resurface next
   tick anyway.

2. Decide, then APPLY the real change when confident. Apply it for real —
   no "DRAFT" notes, no review lists. Reversible fields (description,
   name, domain, archive) you can change on solid evidence; the one lossy
   op (merge) needs near-certainty.
   - Missing/weak description, and you've confirmed what the company is
     (its own site or a reliable source) -> set the real `description`
     via update_company. Write the description field itself — NOT notes,
     and no "DRAFT" prefix.
   - Missing domain, and you've confirmed the canonical domain ->
     add_company_domain, then update_company(primary_domain_id=…) to make
     it primary.
   - Primary domain is a subdomain (e.g. mail.anthropic.com) while the
     canonical apex (anthropic.com) is already one of the company's
     domains -> set the apex as primary via update_company. Fix this
     confidently — it's exactly the Anthropic case.
   - Name is clearly wrong/abbreviated and you know the canonical name ->
     update_company(name=…).
   - Two rows are unambiguously the same identity (identical email /
     linkedin_url / telegram id, or clearly the same legal entity after
     research) -> merge_people / merge_companies.
   - The "company" is not a real organisation — a placeholder bucket
     ("Self-employed", "Freelance", "Stealth", "NDA", and their
     translations like "Фриланс" / "Предприниматель") or a bad-data
     parsing artifact (e.g. a "Com" row with unrelated *.com.xx domains)
     -> archive_company. Reversible, and it stops the row recurring every
     run.
   - A `people-archive-<id>` check is a contact surfaced for archive review.
     Pull get_person + get_person_emails and decide:
     - Clearly not a real, reachable person — a no-reply / automated /
       role address (noreply@, notifications@, a bounce token), or a
       junk/dead row with zero signal -> archive_person.
     - A real person you simply have no recent activity with -> KEEP
       (skip). "No engagement" is not a reason to archive someone real;
       only genuine non-people and junk get archived. A role address you
       actually correspond with (a `support@` with a real thread) is NOT
       junk -> keep.
     When in doubt, keep. Archive is reversible, but needlessly archiving a
     real contact is exactly the kind of churn an unattended run must avoid.

   **When you cannot confirm, skip — full stop.** If web search is
   inconclusive, the evidence is thin, or you'd be guessing, leave the row
   untouched and log it as skipped. Do NOT draft a note and do NOT queue
   it anywhere — an unreviewed draft is just clutter. Confidence is the
   only bar: act on it, or skip.

3. Never hard-delete. Archive (reversible) is the strongest removal;
   description / name / domain edits are all reversible in the app.

4. End with a RUN REPORT. This is the message someone sees when they open
   this run in the task's history, so make **planned-vs-changed** obvious.
   As you work through step 2, track for every check: what you looked at,
   what you *intended* to change, and what you *actually* did. Then output
   one table — planned change in one column, result in the next. A run's
   checks are all one kind; below are the two shapes a report can take.

   A company-enrich tick:

       Kola data-health run — <ISO date & time>

       | Check | Looked at | Planned change | Result |
       |-------|-----------|----------------|--------|
       | company-enrich-7 | Stripe — no domain/description | add domain + set description | applied: domain stripe.com (primary) + description |
       | company-enrich-1888 | Anthropic — primary was mail.anthropic.com | set apex primary | applied: primary -> anthropic.com |
       | company-enrich-31 | "NDA" — placeholder, 38 people, no domain | archive (not a company) | archived |
       | company-enrich-5 | Foo Ltd — no domain | find domain | skipped (could not confirm) |

   A people-archive tick:

       | Check | Looked at | Planned change | Result |
       |-------|-----------|----------------|--------|
       | people-archive-99 | noreply@news.x, 0 signal | archive | archived |
       | people-archive-204 | real contact, no recent activity | keep | skipped (real person) |

       Summary: A applied, B archived, C skipped of N checks.

   For a merge, record the dropped row's name + emails in the "Looked at"
   cell BEFORE merging, so a bad merge can be traced. If the worklist came
   back empty, the report is one line:
   "Kola data-health run <datetime> — nothing to check."

   Do NOT write the report (or anything else) to disk — the run's
   transcript in the task history is the durable record. No log file.
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
- It never hard-deletes. It applies confident changes directly (description,
  domain, name, archive, exact-identity merge) and skips anything it can't
  confirm — it does not draft notes or queue rows for a review that never
  happens.
