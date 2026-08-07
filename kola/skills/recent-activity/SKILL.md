---
name: recent-activity
description: >
  Show who the user has been talking to lately across every channel
  Kola tracks (Gmail, Telegram, WhatsApp, Slack, LinkedIn, calendar,
  notes). The user is
  multilingual; trigger on Russian as readily as English. Use when the user
  says "who did I talk to this week" / "с кем я общался на этой неделе",
  "who has been in touch lately" / "кто выходил на связь в последнее время",
  "show recent activity" / "покажи недавнюю активность", "what did I miss" /
  "что я пропустил", or "who's reached out recently" / "кто недавно
  писал".
argument-hint: '[--days N] [--channel telegram|whatsapp|slack|linkedin|calendar] [--limit N]'
---

# /kola:recent-activity

Cross-channel timeline of who the user has interacted with recently —
ranked by most-recent-touch, deduped by person.

## Instructions

1. **Resolve the window.** Default `--days 7`. Compute `since` as ISO
   `YYYY-MM-DD` for the SQL.

2. **Query `v_people_full`** via `query_people`.

   **`last_interaction_at` is the ranking column.** Kola maintains it as the
   most recent touch across EVERY channel — email, meetings, LinkedIn,
   Telegram, WhatsApp, Slack and note mentions. Rank on it and the answer is
   complete; build your own MAX out of the per-channel columns and it is not,
   because email has no per-channel column to include.

   ```sql
   SELECT
     id, display_name, company, position,
     last_interaction_at,
     telegram_last_message_at,
     whatsapp_last_message_at,
     slack_last_message_at,
     linkedin_last_message_at,
     calendar_last_event_at,
     email_count
   FROM v_people_full
   WHERE archived = 0
     AND last_interaction_at >= :since
   ORDER BY last_interaction_at DESC
   LIMIT :limit
   ```

   **There is no `email_last_message_at` column.** Naming it fails the whole
   query with `no such column`, which is what this skill used to do on every
   run. `email_count` says how much mail there is, never when it arrived.

   Call `describe_people_schema` first to confirm the live column names —
   the view evolves, and it is also where the `cf_*` columns are listed.

3. **Filter by channel** when `--channel <name>` is set. The five channels
   with their own dated column are `telegram`, `whatsapp`, `slack`,
   `linkedin` and `calendar`; swap the WHERE clause to that one column
   (e.g. `WHERE archived = 0 AND telegram_last_message_at >= :since`) and
   order by it.

   **Gmail cannot be filtered this way** — the view dates no email column.
   If the user asks for email specifically, say so plainly and offer the
   thing that does work: `/kola:meeting-brief <person>` or
   `get_person_emails` for one named person.

4. **Render** as:

   ```
   | When | Who | Where | Channel |
   |---|---|---|---|
   | 2 days ago | Jane Smith | Acme · CTO | Telegram |
   | 3 days ago | John Doe | — | Email or note |
   ```

   `When` is a relative date ("today", "yesterday", "3 days ago",
   "last week"); compute against the current date.

   `Channel` is whichever per-channel column equals `last_interaction_at`
   for that row. When none of them does, the touch came from email or from a
   note mention — neither is dated per-channel in the view — so write
   `Email or note` rather than guessing one. Naming a channel the data does
   not support is worse than naming two.

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
