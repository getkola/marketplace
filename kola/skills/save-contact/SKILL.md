---
name: save-contact
description: >
  Capture or update a person in Kola — from a name, business card paste,
  email signature, LinkedIn URL, or freeform "I just met X at Y, they do Z."
  Dedupes against existing rows before creating, and updates them when the
  user means update. The user is multilingual; trigger on Russian as
  readily as English. Use when the user says "save this contact" /
  "сохрани контакт", "add X to Kola" / "добавь X в Kola", "I just met …" /
  "я только что познакомился с …", "remember this person" / "запомни этого
  человека", or pastes contact details.
argument-hint: '<freeform contact details, paste, or "update <person> <field>=<value>">'
---

# /kola:save-contact

Single ingest path for adding a person or patching an existing one.

## Instructions

1. **Parse the input.** Pull out as many of these as you can find:
   - `display_name` (required if creating; for updates, used to locate the row)
   - `first_name`, `last_name`
   - `email1` (and `email2..4` if multiple)
   - `linkedin_url`
   - `telegram_handle`, `whatsapp_phone`
   - `phone`
   - `company`, `position`
   - `location`
   - `birthday` (ISO `YYYY-MM-DD`)
   - `notes` — anything that didn't fit a structured slot, including
     "where / how did I meet them" context

   If the input is a paste with both fielded data and prose, treat the
   prose as `notes`.

2. **Check for an existing match.** In order (first hit wins — mirrors
   Kola's own upsert ladder):
   1. `query_people` exact match on `linkedin_url` if present
   2. `query_people` exact match on `email1..email4` if present
   3. `search_people` substring on `display_name`

   If one match found → confirm with the user before updating ("I see
   Jane Smith @ Acme already — update them, or create a new row?").
   If multiple → list the top 5 and ask which.
   If none → create.

3. **Create.** Call `create_person` with the parsed fields. If the user
   explicitly said "add to list X" or named a list inline, follow with
   `add_to_list` after the row is written.

4. **Update.** Call `update_person` for the matched row. Be conservative:
   - Don't overwrite a non-empty field with new data unless the user
     said "replace" or the new value is strictly longer (e.g. a fuller
     name "Jane Smith" replacing "Jane").
   - For `notes`, append (with a leading blank line) rather than
     overwrite.
   - For custom fields, route to `set_custom_field_value` — one call per
     `(person_id, key, value)`.

5. **Confirm.** Echo back the resolved row:

   ```
   ✅ Saved: <display_name> (#<id>)
      <position> @ <company>
      <email1>, <linkedin_url>
   ```

   If the user said "add to <list>" but the list doesn't exist yet,
   offer to create it (defer to `/kola:lists`).

## Custom-field handling

If the input mentions a field that maps to an existing custom-field
def (call `list_custom_fields` first to learn the schema), set it
via `set_custom_field_value`. If the input mentions a field that
doesn't exist yet, **don't silently create the def** — surface
the gap and offer `/kola:custom-fields` to add it. Schema changes
are deliberate; ingest paths shouldn't sprout new keys.

## Examples

```
/kola:save-contact
Jane Smith, CTO at Acme. jane@acme.com. https://linkedin.com/in/janesmith
Met at React Conf 2026 — working on edge-rendering, hiring for staff eng.
```

```
/kola:save-contact update Jane Smith company=Acme Robotics position=VP Eng
```

```
/kola:save-contact
[paste of email signature block]
```

## Output

One-line confirmation with the resolved `person_id` and the canonical
fields, so the next skill in the chain (e.g. `/kola:meeting-brief`,
`/kola:lists add`) can target the row by id without re-resolving.
