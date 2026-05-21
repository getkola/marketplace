# Kola plugin

Use [Kola](https://getkola.app) — the AI-native people-memory layer — from
Claude Code.

Kola lives as a local desktop app. It indexes your Gmail, Telegram,
WhatsApp, LinkedIn, and calendar into a unified people graph, and exposes
that graph through a local MCP server. This plugin wires Claude Code to
that server and adds workflow skills around the common things people
actually ask: "prep me for tomorrow's call," "who do I know who has
done X," "remember this person I just met."

> **Requirement:** Kola.app must be running. The MCP server is bound to
> `127.0.0.1:47900` — there is no remote endpoint, nothing leaves the
> machine.

## Install

```bash
# In Claude Code
/plugin marketplace add <path-to-this-repo>
/plugin install kola@kola-marketplace
```

Then make sure Kola.app is running (Settings → MCP in Kola shows the
same endpoint this plugin's `.mcp.json` points at). Restart Claude Code
once after install so the MCP server connects.

## Skills

| Command | What it does |
|---|---|
| `/kola:meeting-brief <person>` | One-page prep memo on a person — identity, custom fields, lists they're on, recent messages across every channel, suggested topics to bring up. |
| `/kola:network-search <query>` | Find people by who-they-are (structured filters over Kola's wide view) or what-they-know (semantic search over message bodies). Hybrid is supported. |
| `/kola:save-contact <details>` | Capture a new person or patch an existing one from a paste, a name, or freeform notes. Dedupes against existing rows before creating. |
| `/kola:lists <subcommand>` | Create, add to, remove from, show, rename, delete lists. |
| `/kola:custom-fields <subcommand>` | Manage the per-install custom-field schema — your own columns on every person row. |
| `/kola:recent-activity [--days N]` | Cross-channel timeline of who you've interacted with lately, deduped by person and sorted by most-recent-touch. |

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

The plugin talks to the local Kola MCP server, which exposes:

- People — `list_people`, `search_people`, `query_people` (read-only
  SQL over `v_people_full` / `v_lists` / `v_custom_field_defs`),
  `semantic_search`, `describe_people_schema`, `get_person`,
  `create_person`, `update_person`, `archive_person`, `unarchive_person`,
  `merge_people`, `list_archived`.
- Per-channel history — `get_person_emails`,
  `get_person_telegram_messages`, `get_person_whatsapp_messages`,
  `get_person_linkedin_messages`.
- Lists — `list_lists`, `create_list`, `get_list`, `rename_list`,
  `delete_list`, `add_to_list`, `remove_from_list`.
- Custom fields — `list_custom_fields`, `create_custom_field`,
  `update_custom_field`, `delete_custom_field`,
  `set_custom_field_value`.
- Files — `list_files`, `upload_file`, `delete_file`.
- Accounts — `list_accounts`.

You can also call these directly from Claude Code (`mcp__kola__<name>`)
without going through the skills — the skills exist to wrap the common
multi-tool workflows.

## Privacy

The MCP server is local. The skills only call local tools. No data
leaves your machine through this plugin.
