---
name: network-search
description: >
  Find people in the user's network by who-they-are or what-they-know.
  Combines Kola's name resolver, structured filters (company / role /
  location / list / custom field), substring search, and TWO kinds of
  semantic search — over person PROFILES and over MESSAGE history. The
  user's network is multilingual; trigger on Russian as readily as English.
  Use when the user asks "who do I know who…" / "кто из моих контактов…",
  "find [name]" / "найди [имя]", "find someone at X" / "найди кого-то из X",
  "anyone in my network in [place]" / "кто у меня есть в [место]",
  "who's worked on Z" / "кто работал над Z", "who has experience with W" /
  "у кого есть опыт в W", or "introduce me to [kind of person]" /
  "познакомь меня с [тип человека]". Includes asks like "find Misha from
  Portugal in Kola" / "найди мишу из португалии в Kola".
argument-hint: '<natural-language query> [--limit N]'
---

# /kola:network-search

The single most common failure is **picking the wrong search tool**. Kola
has several distinct corpora; "find a person" and "find what was discussed"
are different searches. Route first, then run — and when in doubt, run
several tools in ONE batch and merge by `id`. Recall beats precision here:
a missed contact is worse than an extra candidate.

## The tools — what each one actually searches

| Tool | Corpus | Use it for |
|---|---|---|
| `resolve_person` | name identity (nickname / cross-script / phonetic + optional company/position hint) | "find <name>" — Миша, Misha, Michael are the SAME person here |
| `search_people` | literal substring over name, email, phone, company, position, **location/city/country, telegram bio, and notes** | exact tokens: a handle, a domain, a city string, "PT", "+351" |
| `semantic_search_people` | embedded PROFILE = display_name + position + company + **notes** + text custom fields | "who is X" by meaning — role, employer, bio, anything a person *is* |
| `semantic_search_messages` | embedded MESSAGE BODIES (Gmail/Telegram/WhatsApp/LinkedIn/Calendar) | "what was discussed" — topics, expertise demonstrated in conversation |
| `semantic_search_notes` | embedded NOTE text (your jottings + AI call notes) | "what was discussed" in your own notes — recall a person from something you wrote about a topic |
| `semantic_search_recordings` | embedded CALL TRANSCRIPTS | "what was discussed" on recorded calls — recall a person from what was actually said on a call |
| `query_people` → `v_people_full` | structured columns incl. `location/city/state/country`, `notes`, `company`, counts, lists, relationship strength, `cf_<key>` | precise filters: "founders in PT", "everyone in my Investors list", "people I'm close to at Acme" |
| `search_telegram` | Telegram's own search over EVERY chat and group at once (words only) | "who mentioned Postgres in Telegram" — group messages are not stored locally, so `semantic_search_messages` misses them |
| `get_shared_circle` | people who appear in the circles of ALL 2–5 given people | "who could introduce me to X", "who do Anna and I both know" |

**"кто это / who is this" → the first four. "о чём говорили / what was
discussed" → `semantic_search_messages`, and for topics that live in your
own notes or recorded calls, `semantic_search_notes` /
`semantic_search_recordings`.** None of these three see profiles — never
use them alone to find a person by location, role, or bio. That single
mistake makes well-populated contacts look missing. Note + recording hits
are also NOT person-scoped (they return `note_id` / `recording_id`, not a
`person_id`), so read the snippet to identify who it points at, then
confirm with `search_people` / `resolve_person`.

## Instructions

1. **Decode the query.** Most "find …" asks are person-identity or
   who-they-are, NOT message-topic. Classify:

   | Shape | Example | Tools (run together) |
   |---|---|---|
   | Named person | "find Misha from Lisbon", "where's Anna Petrova" | `resolve_person` (with name + any company/place as `position`/`company` hint) **and** `search_people` |
   | Who-they-are | "founders in Portugal", "designers I know", "anyone in fintech" | `query_people` (structured) **and** `semantic_search_people` **and** `search_people` |
   | What-was-discussed | "who's worked on rust compilers", "who pitched me on AI safety" | `semantic_search_messages`; add `semantic_search_notes` / `semantic_search_recordings` when the topic may live in your notes or on a recorded call |
   | Hybrid | "founders in PT who talked about fundraising" | structured/profile set first, then `semantic_search_messages` (+ notes/recordings), intersect by `id` |
   | Introduction path | "who could introduce me to Anna", "who do Anna and Boris both know" | resolve each named person first, then `get_shared_circle(person_ids=[…])`; add `min_strength=40` to keep only people the user is close to |

2. **Fan out across forms — Kola is multilingual and multi-source.** Names
   and places are stored inconsistently. Issue parallel calls in ONE tool
   batch covering plausible variants, then merge by `id`:
   - **Name**: nickname ↔ full ↔ Latin — Миша / Misha / Михаил / Michael;
     Саша / Sasha / Alexander. (`resolve_person` expands these itself; for
     `search_people` you must pass each form.)
   - **Place**: name ↔ native ↔ ISO code ↔ phone code — Portugal /
     Португалия / Lisbon / Лиссабон / **PT** / **+351**. Location often
     lives only as a 2-letter code in `notes` or as a phone country code,
     NOT as the word "Portugal" — so a substring search for "Portugal"
     alone WILL miss it. Search the code and the phone prefix too.

3. **Discover the schema before SQL.** For structured/hybrid, call
   `describe_people_schema` to learn live `cf_<key>` columns and lists.
   Don't guess column names.

4. **Run the search.**
   - **Structured** (`query_people`): single SELECT against `v_people_full`
     (also `v_lists`, `v_custom_field_defs`). It exposes `location`, `city`,
     `state`, `country`, `notes`, `company`, `position`, `headline`,
     `*_dm_count`, `email_count`, `calendar_event_count`,
     `last_interaction_at`, `list_ids_csv`, `list_names_csv`, and
     `cf_<key>`. Relationship strength is also there:
     `relationship_depth_score` (0-100, "warm" from 40),
     `relationship_momentum_label` (`growing` / `cooling` / `stable`) and
     `last_meaningful_interaction_at`. NULL means "not measured", never
     "weak" — so `WHERE relationship_depth_score >= 40` is right for "people
     I'm close to", but never filter `< 40` to mean "people I barely know". For location, OR across the structured columns AND a
     `notes LIKE` on the code: `WHERE country = 'PT' OR location LIKE
     '%Portugal%' OR notes LIKE '%Location: PT%' OR phone LIKE '+351%'`.

     **Two columns are comma-wrapped and matched with `LIKE '%,x,%'`**, not
     with `=`: `list_names_csv` (a person's lists) and `emails_csv` (every
     address they have). `email1` is the primary address and the ONLY
     numbered email column — there is no `email2`, `email3` or `email4`, and
     naming one fails the whole query. To find a person by any address:
     `WHERE email1 = :email OR emails_csv LIKE '%,' || :email || ',%'`.
   - **Profile semantic** (`semantic_search_people`): pass the
     natural-language description (role / place / bio). This is what catches
     "PT" in notes that substring "Portugal" misses.
   - **Substring** (`search_people`): one call per literal form (each name
     variant, the city, the ISO code, the phone prefix).
   - **Message semantic** (`semantic_search_messages`): only for
     what-was-discussed. Default `limit` 20; cap at `--limit N`.
   - **Telegram groups** (`search_telegram`): for what-was-discussed when
     the talk may be in Telegram groups. It takes words only, no sender; a
     hit carries `chat_id` and `chat_title`, and the sender still has to be
     resolved to a person with `search_people` / `resolve_person`.
   - **Notes / recordings semantic** (`semantic_search_notes`,
     `semantic_search_recordings`): for what-was-discussed topics that may
     live in your own notes or on recorded calls. Hits carry no
     `person_id` — read each snippet to identify the person, then resolve
     them with `search_people` / `resolve_person` before merging.
   - **Hybrid**: structured/profile set first, take the `person_id`s, run
     `semantic_search_messages` (+ notes/recordings as needed), intersect
     in-process.

5. **Rank and dedupe.** Merge every tool's hits by `id`. A contact that
   surfaces from more than one tool ranks higher. For message hits keep the
   best snippet per person; for structured hits order by
   `COALESCE(updated_at, created_at) DESC`.

6. **Render results** in a scannable table:

   ```
   | # | Name | Where | Why this hit |
   |---|---|---|---|
   | 1 | Michael Bychanok | MediaCube · CEO · PT | notes "Location: PT", +351 phone |
   | 2 | Jane Smith | Acme · CTO | "we shipped a rust compiler" — Telegram, 2026-04-12 |
   ```

   Cap at 10 unless `--limit` says otherwise. Offer "tell me more about #N"
   → defer to `/kola:meeting-brief`.

7. **When nothing matches**: say so, and name what you actually searched
   (which tools, which name/place variants) so the user can correct a wrong
   assumption — e.g. "searched Misha/Михаил/Michael × Portugal/PT/+351
   across profile, notes, and substring." Don't silently widen. Offer one
   concrete next query and wait.

## Examples

```
/kola:network-search find Misha from Lisbon
/kola:network-search founders I know in Portugal
/kola:network-search who's worked on database migrations
/kola:network-search everyone in my Investors list with a check size over 500k
/kola:network-search anyone at Anthropic doing developer relations
```

## Output

A ranked table plus one-line rationale per row. For message hits the
matched snippet is quoted with channel and date; for profile/structured
hits name the field that matched (location, notes, company) so the user can
verify recall without leaving the memo.
