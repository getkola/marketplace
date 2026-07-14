---
name: customize
description: >
  Customize the Kola plugin for this user. For now it does only one
  thing — allow all of Kola's MCP tools — and asks nothing else. Use when
  the user clicks "Customize" on the Kola plugin or says "customize Kola",
  "set up Kola", or "configure the Kola plugin".
argument-hint: ""
---

# /kola:customize

## Scope — read this first

Customizing the Kola plugin does exactly **one** thing right now, and
nothing else: allow all of Kola's MCP tools so the plugin can run without
per-tool permission prompts.

**Do NOT ask anything else** — not the company name, not the user's role,
not default lists, not custom fields, not channels, not meeting-brief
emphasis, not any schedule or cadence. Those are not part of plugin
customization right now. Render only the step below.

## What to do

1. **Tell the user to allow all Kola MCP tools — they must do this, you
   can't.** You cannot grant tool permissions yourself; only the user can,
   through the client's permission UI. So your job here is to *instruct*,
   not to authorize. Say, in plain terms:

   > To use Kola without being asked each time, allow its tools server-wide.
   > When a "Claude wants to use … from kola:kola" prompt appears, open the
   > dropdown and choose the option that allows **all tools from kola:kola**
   > (the whole server), rather than approving one tool at a time. You can
   > also pre-allow the Kola server in the client's settings beforehand.

   Be explicit that approving a single tool only covers that one tool — each
   new tool (archive_company, merge_companies, archive_person, …) will
   prompt again until the whole server is allowed.

2. Confirm what was set up (tools allowed yes/no) and stop. Don't continue
   into any broader configuration.

## Why so narrow

Everything else about Kola — lists, custom fields, who's on what list —
is managed live in the app and through the other `/kola:*` skills, not
baked in at customize time. When more plugin-level settings become
customizable, add them here.
