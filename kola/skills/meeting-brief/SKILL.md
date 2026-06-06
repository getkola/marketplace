---
name: meeting-brief
description: >
  Brief the user on a person before a meeting — pulls everything Kola knows
  about them and renders a one-page prep memo. The user is multilingual;
  trigger on Russian as readily as English. Use when the user says
  "I have a call with X tomorrow" / "у меня завтра созвон с X",
  "prep me for the meeting with Y" / "подготовь меня к встрече с Y",
  "what do I know about Z" / "что я знаю о Z",
  "remind me who X is" / "напомни, кто такой X", or "background on
  [person]" / "расскажи про [человека]".
argument-hint: '<name | email | linkedin url | telegram handle> [--depth full|fast]'
---

# /kola:meeting-brief

Pulls Kola's full memory on one person into a single, scannable prep memo.

## Instructions

1. **Resolve the person.** Pick the best Kola tool for the input:
   - Plain name → `search_people` (substring on display_name).
   - Email → `query_people` with `WHERE email1 = ? OR email2 = ? OR email3 = ? OR email4 = ?` over `v_people_full`.
   - LinkedIn URL or Telegram handle → `query_people` against `linkedin_url` / `telegram_handle`.
   - Multiple matches → list the top 5 by recency (`COALESCE(updated_at, created_at) DESC`) and ask the user which one.
   - Zero matches → say so and stop. Offer to capture them via `/kola:save-contact`.

2. **Hydrate.** Call `get_person` for the canonical row FIRST — it carries
   the skip-signals that decide what else is worth fetching:
   `channel_message_counts` ({linkedin, telegram, whatsapp, slack}),
   `email_count`, and `mentioned_in_notes` (the full list of notes that
   mention this person). Then, in parallel, fetch ONLY the non-empty
   sources — never call a channel tool whose count is 0:
   - `get_person_emails` (if `email_count` > 0) — recent Gmail history (cap to last 10 most recent).
   - `get_person_telegram_messages` (if `channel_message_counts.telegram` > 0) — recent DMs (cap to last 10).
   - `get_person_whatsapp_messages` (if `channel_message_counts.whatsapp` > 0) — recent WhatsApp (cap to last 10).
   - `get_person_linkedin_messages` (if `channel_message_counts.linkedin` > 0) — recent LinkedIn DMs (cap to last 10).
   - `get_person_slack_messages` (if `channel_message_counts.slack` > 0) — recent Slack DMs (cap to last 10).
   - `get_note` for the freshest few `mentioned_in_notes` entries (you already have their ids and titles — no searching needed).
   - `list_custom_fields` — for any per-install fields set on this person, format the values via the person's `custom_fields` object.
   - `semantic_search_notes` — query by the person's name (and company) to surface notes that discuss them *without* a mention link. Dedupe against `mentioned_in_notes` and `people.notes`, which you already have.
   - `semantic_search_recordings` — query by the person's name (and company) to surface call transcripts where they came up; fetch the full text with `get_recording_transcript` only for the closest hit.

   On an older Kola app `channel_message_counts` may be absent from the
   row — only then fall back to calling the channel tools blind.

   Notes and recordings aren't person-scoped, so a semantic hit is a
   *candidate* — keep only snippets that plainly reference this person,
   and drop the rest.

   If `--depth fast`, skip the message-history fetches and the
   notes/recordings semantic searches, and use only what `get_person`
   already returned (`channel_message_counts`, `email_count`,
   `calendar_event_count` + `calendar_event_count_12m` +
   `calendar_last_event_at` for meeting recency, `mentioned_in_notes`
   titles).

3. **Render the memo** in this exact order — headers omitted when their section is empty:

   ```
   # <display_name>
   <position> @ <company> · <location>

   ## Identity
   - Email: …
   - LinkedIn: …
   - Telegram: …
   - WhatsApp: …
   - Phone: …

   ## Custom fields
   <key>: <value>
   …

   ## Lists
   <name>, <name>

   ## Recent threads (last <N> across channels, newest first)
   [<date> · <channel>] <subject or first line>
     <one-line summary>

   ## Notes
   <freeform notes from people.notes, verbatim>

   ## Mentioned in (notes & calls)
   [note · <date>] <title> — <one-line gist>
   [call · <date>] <recording title> — <one-line gist of the relevant moment>
   ```

4. **Closing line — what to ask about.** From the recent thread summaries,
   propose 2–3 specific topics worth bringing up. These should be
   evidence-anchored ("you mentioned X on Y date") — not generic openers.

## Examples

```
/kola:meeting-brief Jane Smith
/kola:meeting-brief jane@acme.com
/kola:meeting-brief https://www.linkedin.com/in/janesmith --depth fast
```

## Output

A self-contained markdown memo. No external links beyond what's stored on
the person row. If Kola has nothing for that person, the memo says so
plainly rather than padding.
