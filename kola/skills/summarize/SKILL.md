---
name: summarize
description: >
  Write a professional summary of a recorded call, its transcript, or a
  note — TL;DR, key points, decisions, and action items, ready to read or
  send. Source recordings and notes are often multilingual (Russian,
  Ukrainian, mixed); summarize in the source language by default and trigger
  on Russian as readily as English. Use when the user says "summarize my
  last call" / "сделай саммари последнего звонка", "what was decided on the
  call with X" / "что решили на звонке с X", "recap the meeting" /
  "перескажи встречу", "summarize this transcript" / "суммируй этот
  транскрипт", "summarize my note about Y" / "сделай выжимку из заметки про
  Y", or "write a recap email for [call]" / "напиши письмо с итогами [звонка]".
argument-hint: '<recording | note | "topic" | id | "last call"> [--lang <language>] [--style brief|detailed|email] [--save]'
---

# /kola:summarize

Turns a call transcript or note into a clean, professional summary. The
hard parts are **finding the right source**, **handling the source
language**, and **not summarizing transcription noise as if it were
content**. Get those right and the write-up is easy.

## The sources — what you're summarizing

| Tool | Returns | Use it for |
|---|---|---|
| `list_recordings` | recordings newest-first (no transcript body) | "my last call", "the call from Tuesday" |
| `search_recordings` | full rows incl. transcript, FTS over title+transcript | a name, project, or exact phrase from the call |
| `semantic_search_recordings` | closest-chunk hits, recording inlined | "the call about pricing" — gist, not exact words |
| `get_recording_transcript([ids])` | full transcript + `transcription_language`/`_prob` | hydrate once you know the recording id(s) |
| `search_notes` / `semantic_search_notes` | note rows / chunk hits | summarizing a typed or AI note instead of a call |
| `get_note(id)` | one note with body | hydrate a note by id |

## Instructions

1. **Find the source.** Route on what the user gave you:
   - **"last call" / "latest recording" / "most recent"** → `list_recordings(limit=10)`, take the newest with `status == 'recorded'`.
   - **A topic or person** ("the call about pricing", "my call with Anna") →
     run `search_recordings` (exact words) **and** `semantic_search_recordings`
     (gist) in one batch; merge by `id`.
   - **A bare number** → ambiguous between a recording and a note. Try
     `get_recording_transcript([id])` first; if it errors with "recording
     not found", fall back to `get_note(id)`.
   - **"my note about Y"** → `search_notes` + `semantic_search_notes`.
   - **A pasted transcript or block of text in the prompt** → summarize it
     directly, no lookup.
   - **Multiple plausible matches** → list the top 5 (title + date) and ask
     which one. Don't guess when more than one call fits.
   - **Zero matches** → say so and name what you searched.

2. **Hydrate the full text.**
   - Recordings: `get_recording_transcript([ids])`. The transcript is only
     present when `transcription_status == 'done'`. Other states land in
     `errors` — handle them honestly:
     - `pending` / `transcribing` → "Kola hasn't transcribed this call yet
       (the on-device transcription agent runs on a schedule, often
       overnight). Nothing to summarize until it finishes." Stop.
     - `failed` → report `transcription_error`; offer no summary.
   - Notes: use the `body` from `get_note` / the search hit.
   - **Multiple recordings** (e.g. "summarize all my calls with Acme this
     week"): fetch each transcript, then summarize them together with a
     per-call sub-section.

3. **Read the transcript format.** Call transcripts are speaker-diarized as
   **`Me:`** (the user's microphone) and **`Them:`** (everything the call
   played back). Diarization is mic-vs-system only — it **cannot separate
   two remote speakers**, so on a 3+-party call everyone but the user is
   lumped under "Them". Attribute decisions and asks via these labels; if
   the recording title or the user tells you who "Them" is, you may name
   them, otherwise write "the other party". Never invent attendee names.

4. **Handle language — this is multilingual data.**
   - Read `transcription_language` and `transcription_language_prob` from the
     transcript payload. **Default: write the summary in the source
     language.** A Russian call gets a Russian summary; an English call an
     English one.
   - **Override** when the user asks ("summarize in English", "по-русски",
     `--lang <language>`) — then write in that language regardless of source.
   - **Mixed / code-switched** transcripts: pick the dominant language for
     the prose; keep proper nouns, product names, and direct quotes verbatim.
   - **Low confidence** (`transcription_language_prob` low, or it's missing):
     add one line noting the transcript may be imperfect, and lean on the
     content you're confident about.

5. **Ignore transcription noise — do not summarize it.** Whisper
   hallucinates filler over silent stretches: subtitle artifacts like
   "Продолжение следует…", "Субтитры сделал DimaTorzok", "Thank you.",
   "Субтитры…", repeated single phrases. These are **not** things anyone
   said — drop them silently. Likewise ignore obvious mis-transcription
   gibberish; if a key passage is garbled, say "(unclear in the recording)"
   rather than guessing at content.

6. **Extract the important information first — this is a people-memory
   tool, not just a recap.** Before writing prose, pull the durable signals
   out of the transcript or note. Skip anything not actually present; never
   invent. Capture:
   - **People mentioned** — names of anyone referenced (not just the
     speakers): "Anna said her CTO Pavel will…". These are who-met-whom
     signals worth remembering.
   - **Companies / projects** named.
   - **Signals** — facts worth keeping: someone changed jobs, is raising, is
     hiring, moved cities, a relationship/intent ("they want to invest",
     "they're unhappy with vendor X").
   - **Events / dates** — meetings agreed, deadlines, milestones, birthdays,
     trips ("demo on the 14th", "they're in Lisbon next month").
   - **Action items** — who owes what, by when. Attribute via Me/Them.
   - **Open questions** — unresolved threads, things to follow up on.
   - **Numbers / commitments** — amounts, terms, dates — verbatim.

7. **Enrich from Kola — look up what you extracted before writing.** Kola is
   a memory graph; a name on a call is often someone you already know. For
   the entities from step 6, run lookups (batch them) to add context:
   - **People mentioned** → `resolve_person` (handles nicknames /
     cross-script / phonetic — Миша = Misha = Michael) and `search_people`
     for each name. A hit means you can attribute the right person and note
     their role/company from the row; a miss is itself useful (a *new*
     person worth capturing — flag it).
   - **Companies / projects** → `search_people` / `query_people` over
     `v_people_full`, or `semantic_search_companies`, to tie the call to
     people and firms already on file.
   - **Topics discussed** → `semantic_search_messages` and
     `semantic_search_notes` / `semantic_search_recordings` on the key
     topics, to surface prior threads or earlier calls on the same subject —
     so the summary can say "this continues the pricing thread from the
     March call" instead of treating the call as standalone.
   - Use enrichment to **disambiguate "Them"**, correct name spellings, and
     add a one-line "who this is" for mentioned people — but keep it
     grounded: only fold in a Kola fact when the match is confident, and
     never let looked-up data override what was actually said on the call.

8. **Write the summary — professional, faithful, no padding.** Tone:
   neutral, concise, executive. Every claim must trace to the transcript (or
   a confident Kola match, clearly attributed) — **never invent decisions,
   numbers, dates, or commitments.** Build it from the extracted +
   enriched info above. Structure by `--style` (default `detailed`):

   **`detailed`** (default):
   ```
   # <title or topic> — <date>
   <who was on the call, if known> · <duration if useful>

   ## TL;DR
   2–3 sentences: what the call was about and the single most important outcome.

   ## People & companies mentioned
   - <name> — <role / why they came up> <(known in Kola — one-line who, or "new — not in Kola yet")>

   ## Key points
   - <substantive point, attributed to Me/Them where it matters>

   ## Signals worth remembering
   - <job change / intent / hiring / fundraising / location / relationship>

   ## Decisions
   - <what was decided> — <by whom>

   ## Action items
   - [ ] <owner> — <task> — <deadline if stated, else "no date">

   ## Events & dates
   - <date / milestone> — <what>

   ## Open questions / follow-ups
   - <unresolved thread>
   ```
   Omit any section that's genuinely empty rather than writing "None".

   **`brief`**: just **TL;DR** + **Action items**. A few lines total.

   **`email`**: a ready-to-send recap email — greeting, 2–4 sentence recap,
   a bulleted "next steps / action items" list, a sign-off. Written in the
   source language (or `--lang`). Leave `[recipient]` / `[your name]` as
   placeholders only if you can't infer them.

9. **Save it back — read first, append, never overwrite.**
   - **Link the people first.** In the body you're about to save, write
     every person you confidently resolved in step 7 as a mention link:
     `[Display Name](https://getkola.app/o/person/<id>)` (the person's
     `link` field from `resolve_person` / `get_person` is exactly this URL).
     First occurrence in the body is enough; later occurrences stay plain.
     Kola turns these into person chips and back-references: the note shows
     up in each person's `mentioned_in_notes` and feeds their engagement
     signal. A name you could NOT resolve stays plain text — never guess an
     id, and never link a low-confidence match. (This is for the **saved
     note only** — chat output and `--style email` keep plain names.)
   - **Find any existing note.** A recording usually already has a note
     attached (`recording_id` on the note, surfaced as `rec_*`); when you
     summarized a note directly you already have its id. Otherwise scan
     `list_notes` for one whose `recording_id` matches this recording.
   - **Always `get_note(id)` first** to read the current `body` — never
     write blind.
   - **If a note exists, append.** Call `update_note(note_id, body=<existing
     body> + "\n\n---\n\n" + <summary>)` — preserve everything that's
     already there (the user's own jottings, prior AI passes, and their
     existing mention links) and add the new summary at the **end**. Do
     **not** replace the body or touch the title.
   - **If no note exists**, create one: `create_note(body=<summary>,
     title="Summary: <call title> · <date>")`.
   - Either way the result is recallable by `search_notes` /
     `semantic_search_notes` and feeds future `/kola:meeting-brief` passes
     (mention links are exactly what its `mentioned_in_notes` reads).
     Confirm the note id and whether you appended or created. Without
     `--save`, end by asking whether to save.

## Examples

```
/kola:summarize last call
/kola:summarize the call about Q3 pricing --style brief
/kola:summarize my call with Anna --lang English
/kola:summarize 482 --style email --save
/kola:summarize сделай саммари последнего звонка
```

## Output

A single self-contained summary in the source language (or the requested
one), structured per `--style`. It reflects only what's actually in the
transcript or note — gaps are flagged, not filled. Transcription artifacts
are stripped. When the source isn't transcribed yet or transcription
failed, the skill says so plainly instead of producing an empty summary.
