# Kola plugin

Use [Kola](https://getkola.app) — the AI-native people-memory layer — from
Claude Code.

Kola lives as a desktop app. It indexes your Gmail, Telegram,
WhatsApp, LinkedIn, and calendar into a unified people graph, and exposes
that graph through an MCP server. This plugin wires Claude Code to that
server and adds workflow skills around the common things people
actually ask: "prep me for tomorrow's call," "who do I know who has
done X," "remember this person I just met."

## How the plugin reaches Kola

The plugin connects to Kola's remote MCP endpoint,
`https://mcp.getkola.app/mcp`. Your data stays in the database on your
Mac: the Kola app opens an outgoing tunnel to Cloudflare, and each call
travels down that tunnel and is answered on the Mac.

```
Claude Code ──HTTPS + OAuth──► mcp.getkola.app/mcp ──tunnel──► Kola.app on your Mac
```

So a call works only when all of these are true:

1. **You signed in.** The first time Claude Code connects, run `/mcp`,
   pick `kola`, and choose Authenticate. A browser opens; sign in with
   your Kola account and approve the client.
2. **Remote access is on for your account.** Open
   <https://my.getkola.app/settings> and turn it on.
3. **"Any address" is on** on the same page. The "Claude" option only
   allows calls from Anthropic's servers (the claude.ai connector).
   Claude Code calls from your own internet address, so without "Any
   address" every call is refused with "This address is not on the allow
   list". With it on, the OAuth sign-in is the only check.
4. **Kola is running on a Mac with remote access allowed.** In Kola, open
   Settings → MCP and turn on **Allow remote access**. If the Mac sleeps
   or Kola quits, calls fail with "Kola is not connected".

`/kola:run` starts the app and maps each error message to the switch
that fixes it.

## Install

```bash
# In Claude Code
/plugin marketplace add <path-to-this-repo>
/plugin install kola@kola-marketplace
```

Restart Claude Code once after install, then run `/mcp` and
authenticate the `kola` server.

## Local fallback (no remote access)

The plugin itself connects only to the remote endpoint. If you do not
want remote access, for example no Kola account or "Any address" is not
acceptable for you, you can connect Claude Code straight to the Kola app
on the same Mac. The app bundle ships a small bridge for this:

```bash
claude mcp add kola -- /Applications/Kola.app/Contents/Resources/backend/kola-backend/kola-backend mcp-bridge http://127.0.0.1:47900/mcp
```

What changes with the fallback:

| | Remote (plugin default) | Local fallback |
|---|---|---|
| Works from | any machine | only the Mac that runs Kola |
| Needs a Kola account and the switches above | yes | no |
| Kola.app location | anywhere | must be in `/Applications`, because the command names that path |
| Tool names | `mcp__plugin_kola_kola__<name>` | `mcp__kola__<name>` |

The skills and the `contact-suggester` agent work with both tool names.
In `/mcp`, disable the plugin's remote `kola` server while you use the
fallback, so each tool does not appear twice.

## Skills

| Command | What it does |
|---|---|
| `/kola:meeting-brief <person>` | One-page prep memo on a person — identity, custom fields, lists they're on, recent messages across every channel, suggested topics to bring up. |
| `/kola:network-search <query>` | Find people by who-they-are (structured filters over Kola's wide view) or what-they-know (semantic search over message bodies). Hybrid is supported. |
| `/kola:save-contact <details>` | Capture a new person or patch an existing one from a paste, a name, or freeform notes. Dedupes against existing rows before creating. |
| `/kola:lists <subcommand>` | Create, add to, remove from, show, rename, delete lists. |
| `/kola:custom-fields <subcommand>` | Manage the per-install custom-field schema — your own columns on every person row. |
| `/kola:recent-activity [--days N]` | Cross-channel timeline of who you've interacted with lately, deduped by person and sorted by most-recent-touch. |
| `/kola:summarize <call\|note\|topic>` | Professional summary of a recorded call's transcript or a note — TL;DR, key points, decisions, action items. Summarizes in the source language by default (handles Russian/Ukrainian/mixed), strips transcription noise, and can save the recap back as a note. |
| `/kola:customize` | Plugin customization (also the Cowork "Customize" button). For now does one thing — allows all of Kola's MCP tools so the plugin runs without per-tool prompts. |

## Commands

| Command | What it does |
|---|---|
| `/kola:run` | Start Kola.app if it isn't running, and explain which switch fixes a refused remote call. macOS only. |

## Agents

| Agent | What it does |
|---|---|
| **contact-suggester** | Silent by default. Fires at most once per conversation, and only when overwhelming multi-signal evidence (multiple recent matching messages + a structured corroboration, on a *specific* named topic) points at one person in your network. If unsure, says nothing — interrupting you with a weak guess is treated as worse than staying quiet. When it does fire, it appends one line with a real quoted snippet, the channel, and the date. Read-only. Tell it "stop suggesting" to stand down for the session. |

## How the skills connect

```
recent-activity ─┐
network-search ──┼──► meeting-brief ──► (your meeting)
                 │
save-contact ────┴──► lists / custom-fields ──► (better future briefs)

(rare, only on overwhelming evidence) ──► contact-suggester ──► one quoted name
```

`meeting-brief` is the central read surface. `save-contact`, `lists`,
and `custom-fields` are the write surfaces — they sharpen what
`meeting-brief` and `network-search` can recall on the next pass. The
`contact-suggester` agent is silent by default. It only emits a single
one-line suggestion when overwhelming, multi-signal evidence lines up —
the floor is intentionally high so the agent never disturbs you with a
weak guess.

## Underlying tools

The remote endpoint forwards every call to the Kola app unchanged, so
the plugin sees the app's full tool surface (about 110 tools). The main
groups:

- People — `list_people`, `search_people`, `resolve_person`,
  `query_people` (read-only SQL over `v_people_full` / `v_lists` /
  `v_custom_field_defs`), `describe_people_schema`, `get_person`,
  `create_person`, `update_person`, `archive_person`, `unarchive_person`,
  `merge_people`, `split_person`, `list_possible_duplicates`,
  `list_archived`.
- Relationships — `get_person_circle` (who turns up alongside a person),
  `get_shared_circle` (who two to five people all know), and the
  relationship-strength columns on `v_people_full`
  (`relationship_depth_score`, `relationship_momentum_label`, …).
- Per-channel history — `get_person_emails`,
  `get_person_telegram_messages`, `get_person_whatsapp_messages`,
  `get_person_linkedin_messages`, `get_person_slack_messages`,
  `get_person_linkedin_activity`. `get_person` returns
  `channel_message_counts` (plus `email_count` on the row) so callers
  skip channels that are empty instead of probing them.
- Telegram groups — `get_person_telegram_groups`, `search_telegram`
  (every chat at once), `search_telegram_chat`,
  `get_telegram_chat_messages`, `list_telegram_group_members`.
- Semantic recall — `semantic_search_messages`,
  `semantic_search_people`, `semantic_search_companies`,
  `semantic_search_notes`, `semantic_search_recordings`.
- Companies — `get_company`, `query_companies`,
  `get_company_linkedin_news`, and the create / update / merge / split
  tools.
- Notes and recordings — `list_notes`, `get_note`, `create_note`,
  `update_note`, `list_recordings`, `search_recordings`,
  `get_recording_transcript`.
- Lists and custom fields — `list_lists`, `create_list`, `get_list`,
  `rename_list`, `delete_list`, `add_to_list`, `remove_from_list`,
  `list_custom_fields`, `create_custom_field`, `update_custom_field`,
  `delete_custom_field`, `set_custom_field_value`.
- Also: files, workflows, settings, record history, and accounts.

The drawing tools (`show_network`, `show_map`, …) work only inside
Kola's own chat and refuse other callers.

You can also call these directly from Claude Code without going through
the skills — the skills exist to wrap the common multi-tool workflows.

**The tool name depends on where the server came from**, because the
prefix is chosen by the client, not by Kola:

| Kola's MCP server comes from | Tool name |
|---|---|
| this plugin | `mcp__plugin_kola_kola__<name>` |
| a project `.mcp.json` entry named `kola` | `mcp__kola__<name>` |

Installed through the plugin, it is the first form. If a tool call fails
with `No such tool available: mcp__kola__<name>`, that is this difference
and not a missing server.

## Privacy

Your contacts and messages stay in the database on your Mac. Nothing is
copied to Cloudflare; each call passes through the tunnel and the answer
is computed on the Mac.

What does leave the Mac: the text of each call and its answer, in transit
through `mcp.getkola.app`. The service records one analytics event per
call with your account id, the method, the client type and the tool name.
It does not record the arguments or the answer.

You can cut access at any time: turn remote access off at
<https://my.getkola.app/settings>, disconnect this client there, or turn
off **Allow remote access** in Kola on the Mac.
