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

1. **Allow all Kola MCP tools.** Ask the user to permanently allow every
   tool from the Kola MCP server (the `mcp__kola__*` tools) — "always
   allow", not per-call. Explain why in one line:

   > Allow Kola's tools to run without asking each time? The scheduled
   > data-health sweep runs unattended, so it can't answer permission
   > prompts — it needs these pre-approved or it will stall.

   On yes, grant it the way this client does: add an allow rule for the
   whole Kola server to settings (e.g. `mcp__kola` in `permissions.allow`),
   or accept the client's "always allow for this server" affordance. Do
   not enumerate tools one by one — allow the server as a whole. If the
   user declines, continue, but warn that the unattended sweep will stall
   on the first tool call until the tools are allowed.

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
