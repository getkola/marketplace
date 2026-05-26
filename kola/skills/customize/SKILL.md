---
name: customize
description: >
  Customize the Kola plugin for this user. For now it does only two
  things — allow all of Kola's MCP tools, and choose how often the
  data-health sweep runs — and asks nothing else. Use when the user
  clicks "Customize" on the Kola plugin or says "customize Kola", "set up
  Kola", or "configure the Kola plugin".
argument-hint: ""
---

# /kola:customize

## Scope — read this first

Customizing the Kola plugin does exactly **two** things right now, and
nothing else:

1. Allow all of Kola's MCP tools so the plugin can run without prompts.
2. Choose how often the recurring data-health sweep runs.

**Do NOT ask anything else** — not the company name, not the user's role,
not default lists, not custom fields, not channels, not meeting-brief
emphasis. Those are not part of plugin customization right now. Render
only the two steps below.

## What to do

1. **Tell the user to allow all Kola MCP tools — they must do this, you
   can't.** You cannot grant tool permissions yourself; only the user can,
   through the client's permission UI. So your job here is to *instruct*,
   not to authorize. Say, in plain terms:

   > Before the scheduled sweep can run hands-off, you need to allow Kola's
   > tools server-wide — it runs unattended and can't answer permission
   > prompts, so a per-tool approval will stall it on the first new tool.
   > When a "Claude wants to use … from kola:kola" prompt appears, open the
   > dropdown next to **"Allow for all scheduled runs"** and choose the
   > option that allows **all tools from kola:kola** (the whole server),
   > rather than approving one tool at a time. You can also pre-allow the
   > Kola server in the client's settings beforehand.

   Be explicit that approving a single tool ("Allow for all scheduled
   runs" without the server-wide option) only covers that one tool — each
   new tool (archive_company, merge_companies, archive_person, …) will
   prompt again until the whole server is allowed. If they don't allow
   server-wide, the unattended sweep will stall mid-run the first time it
   reaches an un-approved tool.

2. **Ask the data-health cadence** (a single-select):

   > **How often should the data-health sweep run?**
   > *(de-dupes people, flags junk contacts)*
   >
   > Hourly · Daily · Weekly · Don't schedule it

   Default the selection to **Hourly**. Do not add any other fields,
   text inputs, or follow-up questions to the form.

3. On the cadence answer:
   - **Hourly / Daily / Weekly** → run the `setup-schedule` skill to
     register the recurring data-health sweep at the chosen cadence
     (it handles the offer/confirm and the `/schedule` call). Pass the
     cadence through.
   - **Don't schedule it** → make no schedule. Confirm that nothing was
     scheduled and that they can run `/kola:setup-schedule` later.

4. Confirm what was set up (tools allowed yes/no; cadence + that it only
   runs while Kola.app is open) and stop. Don't continue into any broader
   configuration.

## Why so narrow

Everything else about Kola — lists, custom fields, who's on what list —
is managed live in the app and through the other `/kola:*` skills, not
baked in at customize time. Customization stays to the two things that
must be set up front for the plugin to work hands-off: the tool
permissions (so the unattended sweep doesn't stall) and the sweep
cadence. When more plugin-level settings become customizable, add them
here.
