---
name: recent-activity
description: >
  Show who the user has been talking to lately across every channel
  Kola tracks (Gmail, Telegram, WhatsApp, LinkedIn, calendar). Use when
  the user says "who did I talk to this week", "who has been in touch
  lately", "show recent activity", "what did I miss", or "who's
  reached out recently".
argument-hint: '[--days N] [--channel gmail|telegram|whatsapp|linkedin|calendar] [--limit N]'
---

# /kola:recent-activity

Cross-channel timeline of who the user has interacted with recently —
ranked by most-recent-touch, deduped by person.

## Instructions

1. **Resolve the window.** Default `--days 7`. Compute `since` as ISO
   `YYYY-MM-DD` for the SQL.

2. **Query `v_people_full`** via `query_people`. The view exposes
   per-channel `*_last_message_at` / `*_last_event_at` columns:

   ```sql
   SELECT
     id, display_name, company, position,
     email_last_message_at,
     telegram_last_message_at,
     whatsapp_last_message_at,
     linkedin_last_message_at,
     calendar_last_event_at,
     MAX(
       COALESCE(email_last_message_at, ''),
       COALESCE(telegram_last_message_at, ''),
       COALESCE(whatsapp_last_message_at, ''),
       COALESCE(linkedin_last_message_at, ''),
       COALESCE(calendar_last_event_at, '')
     ) AS last_touch
   FROM v_people_full
   WHERE archived = 0
     AND (
       email_last_message_at >= :since OR
       telegram_last_message_at >= :since OR
       whatsapp_last_message_at >= :since OR
       linkedin_last_message_at >= :since OR
       calendar_last_event_at >= :since
     )
   ORDER BY last_touch DESC
   LIMIT :limit
   ```

   First call `describe_people_schema` to confirm the exact column
   names — Kola's view evolves, so verify before composing.

3. **Filter by channel** when `--channel <name>` is set: drop the
   other channels from the WHERE / SELECT.

4. **Render** as:

   ```
   | When | Who | Where | Channel |
   |---|---|---|---|
   | 2 days ago | Jane Smith | Acme · CTO | Telegram |
   | 3 days ago | John Doe | — | Gmail |
   ```

   `When` is a relative date ("today", "yesterday", "3 days ago",
   "last week"); compute against the current date. `Channel` is whichever
   column equaled `last_touch` for that row.

5. **Offer drill-downs.** "Tell me what we said" → defer to
   `/kola:meeting-brief <name>` for the message history, or
   `get_person_<channel>_messages` directly for a single channel.

## Examples

```
/kola:recent-activity
/kola:recent-activity --days 30
/kola:recent-activity --channel telegram --days 14
/kola:recent-activity --limit 5
```

## Output

A ranked table — newest interaction first. Caps at `--limit` (default
20). Empty result is reported plainly ("No interactions in the last N
days"); don't widen the window without being asked.
