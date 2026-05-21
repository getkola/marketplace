---
name: network-search
description: >
  Find people in the user's network by who-they-are or what-they-know.
  Combines Kola's structured filters (company / role / list / custom field)
  with semantic search over the user's message history. Use when the user
  asks "who do I know who…", "find someone at X", "anyone in my network
  doing Y", "who's worked on Z", "who has experience with W", or
  "introduce me to <kind of person>".
argument-hint: '<natural-language query> [--limit N]'
---

# /kola:network-search

Two-pass search: structured first (when the query has a structured shape
like "people at Acme"), semantic second (when it's about topic / expertise
that lives in message bodies).

## Instructions

1. **Decode the query.** Classify into one of three shapes:

   | Shape | Example | First tool |
   |---|---|---|
   | Structured filter | "people at Acme", "founders in NYC", "everyone in my Investors list" | `query_people` against `v_people_full` |
   | Topic / expertise | "who's worked on rust compilers", "anyone who's done a Series A recently" | `semantic_search` |
   | Hybrid | "founders I know who've talked about AI safety" | both, with the structured set used to filter semantic hits |

2. **Discover the schema before composing SQL.** For structured /
   hybrid queries, call `describe_people_schema` to learn the live
   `cf_<key>` columns and current lists. Don't guess column names.

3. **Run the search.**
   - **Structured**: compose a SELECT against `v_people_full` (also
     `v_lists`, `v_custom_field_defs`). The view exposes
     `email1..email4`, `email_count`, `telegram_dm_count`,
     `whatsapp_dm_count`, `calendar_event_count`, `list_ids_csv`,
     `list_names_csv` (both wrapped in commas — match
     `LIKE '%,Name,%'`), and per-install `cf_<key>` columns.
     Single SELECT only — the tool rejects multi-statement, writes,
     and tables outside the three views.
   - **Semantic**: pass the natural-language query straight through
     to `semantic_search`. Default `limit` is 20; cap at `--limit N`
     when provided. Each hit returns the snippet and the
     person it belongs to.
   - **Hybrid**: run the structured query first, take the resulting
     `person_id`s, then run `semantic_search` and filter to that set
     (in-process — the tool doesn't take a person_id filter).

4. **Rank and dedupe.** For semantic hits, group by person and keep
   the highest-scoring snippet per person. For structured hits, order
   by `COALESCE(updated_at, created_at) DESC`.

5. **Render results** in a table the user can scan:

   ```
   | # | Name | Where | Why this hit |
   |---|---|---|---|
   | 1 | Jane Smith | Acme · CTO | "we shipped a rust compiler last quarter" — Telegram, 2026-04-12 |
   ```

   Cap at 10 unless `--limit` says otherwise. Offer "tell me more about
   #N" → defer to `/kola:meeting-brief`.

6. **When nothing matches**: say so explicitly. Don't widen the query
   silently. Offer one alternative query (e.g. drop a filter, broaden
   the semantic phrasing) and wait.

## Examples

```
/kola:network-search founders I know in New York
/kola:network-search who's worked on database migrations
/kola:network-search everyone in my Investors list with a check size over 500k
/kola:network-search anyone at Anthropic doing developer relations
```

## Output

A ranked table plus one-line rationale per row. For semantic hits, the
matched snippet is quoted with its source channel and date so the user
can verify the recall without leaving the memo.
