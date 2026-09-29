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
   - Email → `query_people` over `v_people_full`. The view carries ONE email
     column, `email1` (the primary). Every other address for that person
     lives in `emails_csv`, which is comma-wrapped so a lookup needs no join:
     `WHERE email1 = :email OR emails_csv LIKE '%,' || :email || ',%'`.
     There is no `email2` / `email3` / `email4` column — naming one fails the
     whole query with `no such column`.
   - LinkedIn URL or Telegram handle → `query_people` against `linkedin_url`
     / `telegram_username`. The column is `telegram_username`, not
     `telegram_handle`.
   - **Verify column names with `describe_people_schema` before writing SQL.**
     `v_people_full` evolves; this list is a starting point, not a contract.
   - Multiple matches → list the top 5 by recency (`COALESCE(updated_at, created_at) DESC`) and ask the user which one.
   - Zero matches → say so and stop. Offer to capture them via `/kola:save-contact`.

2. **Hydrate.** Call `get_person` for the canonical row FIRST — it carries
   the skip-signals that decide what else is worth fetching:
   `channel_message_counts` ({linkedin, telegram, whatsapp, slack}),
   `email_count`, `mentioned_in_notes` (the full list of notes that
   mention this person), and `relationship` (`depth`, `freshness`,
   `momentum`, `coverage` — see step 3). Then, in parallel, fetch ONLY the non-empty
   sources — never call a channel tool whose count is 0:
   - `get_person_emails` (if `email_count` > 0) — recent Gmail history (cap to last 10 most recent).
   - `get_person_telegram_messages` (if `channel_message_counts.telegram` > 0) — recent DMs (cap to last 10).
   - `get_person_whatsapp_messages` (if `channel_message_counts.whatsapp` > 0) — recent WhatsApp (cap to last 10).
   - `get_person_linkedin_messages` (if `channel_message_counts.linkedin` > 0) — recent LinkedIn DMs (cap to last 10).
   - `get_person_linkedin_activity` — their recent LinkedIn posts, reposts
     and comments. This is what they have said in PUBLIC, which is often the
     freshest signal on a person you have not messaged lately: a new job, a
     launch, a subject they keep returning to. Unlike the DM tools it is
     worth calling even when `channel_message_counts.linkedin` is 0 —
     activity is scraped from their profile, not from a conversation with
     you, so the two counts are unrelated.
   - `get_person_slack_messages` (if `channel_message_counts.slack` > 0) — recent Slack DMs (cap to last 10).
   - `get_note` for the freshest few `mentioned_in_notes` entries (you already have their ids and titles — no searching needed).
   - `list_custom_fields` — for any per-install fields set on this person, format the values via the person's `custom_fields` object.
   - `semantic_search_notes` — query by the person's name (and company) to surface notes that discuss them *without* a mention link. Dedupe against `mentioned_in_notes` and `people.notes`, which you already have.
   - `semantic_search_recordings` — query by the person's name (and company) to surface call transcripts where they came up; fetch the full text with `get_recording_transcript` only for the closest hit.
   - `get_person_circle(person_id, limit=10)` — who else turns up alongside
     them: co-recipients, co-attendees, shared groups, same company. Each row
     has per-channel shared counts and `count`. Pass `min_strength=40` to keep
     only people the USER is also close to (40 is the "warm" band Kola's
     person card uses); people Kola has not measured are then left out.
   - `get_person_telegram_groups(person_id)` (if they have Telegram) — the
     groups you share with them. Local and cheap; this is often the "where do
     I know them from" answer.
   - `get_company_linkedin_news([company_id])` — optional, and only for the
     ONE company they work at. Find the id with `query_companies` or
     `semantic_search_companies` on the row's `company`. It fetches live from
     LinkedIn, is slow, and shares a cap of 30 live LinkedIn fetches an hour
     with `get_person_linkedin_activity`. Read `status` per company; on
     `rate_limited`, `busy` or `budget_exhausted` skip the section rather
     than retry.

   On an older Kola app `channel_message_counts` may be absent from the
   row — only then fall back to calling the channel tools blind.

   Notes and recordings aren't person-scoped, so a semantic hit is a
   *candidate* — keep only snippets that plainly reference this person,
   and drop the rest.

   If `--depth fast`, skip the message-history fetches, the LinkedIn
   activity and company-news fetches, the circle and Telegram-group calls,
   and the notes/recordings semantic searches, and use only
   what `get_person` already returned (`channel_message_counts`,
   `email_count`, `calendar_event_count` + `calendar_event_count_12m` +
   `calendar_last_event_at` for meeting recency, `mentioned_in_notes`
   titles).

3. **Render the memo** in this exact order — headers omitted when their section is empty:

   ```
   # <display_name>
   <position> @ <company> · <location>
   Strength <depth>/100 · <momentum in words> · last real conversation <date>

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

   ## Around them
   <name> — <shared: 12 emails, 3 meetings>   (from get_person_circle)
   Telegram groups: <title>, <title>

   ## Recent threads (last <N> across channels, newest first)
   [<date> · <channel>] <subject or first line>
     <one-line summary>

   ## Publicly, on LinkedIn
   [<date>] <post, repost or comment> — <one-line gist>

   ## Company news (<company>, LinkedIn, <live|cache>)
   [<date>] <one-line gist>

   ## Notes
   <freeform notes from people.notes, verbatim>

   ## Mentioned in (notes & calls)
   [note · <date>] <title> — <one-line gist>
   [call · <date>] <recording title> — <one-line gist of the relevant moment>
   ```

   **The strength line.** Print `depth` as the one number and say the rest
   in words: momentum `growing` / `cooling` / `stable` → "getting closer" /
   "cooling off" / "steady" (`unknown` → leave momentum out); `last_meaningful_interaction_at` from the row
   as the date. **A missing key means Kola could not measure it, never
   zero** — write "not measured yet" instead of a number. If `coverage`
   marks a source `partial`, add "(may be low: <source> is only partly
   synced)". Omit the line when none of it is measured.

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
