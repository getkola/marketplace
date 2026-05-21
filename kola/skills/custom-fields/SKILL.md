---
name: custom-fields
description: >
  Manage Kola's per-install custom-field schema — the user-defined
  extension columns on every person. Add a new field (text / number /
  boolean / datetime / file), rename or describe an existing one,
  re-order, or delete one. Use when the user says "add a custom field
  for X", "track Y on every person", "change the label for Z field",
  "what custom fields do I have", or "drop the W field".
argument-hint: '<add|edit|list|delete> [args]'
---

# /kola:custom-fields

Schema-level workflow over Kola's `custom_field_defs`. Per-person
values are set by `/kola:save-contact` and other skills — this skill
only touches the schema.

## Sub-commands

### `list`
Call `list_custom_fields`. Render: `key`, `label`, `type`, `position`,
`description` (truncated to one line).

### `add <key> <label> [<type>] [-- <description>]`
Defaults: `type=text`, no description. Allowed types: `text`,
`number`, `boolean`, `datetime`, `file`.

Before creating:
- `key` must match `^[a-z][a-z0-9_]*$`. Pitch a fixed key if the user
  proposed something else.
- `key` must not collide with a `v_people_full` reserved column
  (`display_name`, `first_name`, `last_name`, `email1`..`email4`,
  `company`, `position`, `linkedin_url`, `telegram_handle`,
  `whatsapp_jid`, `whatsapp_phone`, `phone`, `birthday`, `location`,
  `notes`, `archived`, `email_count`, `telegram_dm_count`,
  `whatsapp_dm_count`, `calendar_event_count`, `linkedin_dm_count`,
  `list_ids_csv`, `list_names_csv`, `created_at`, `updated_at`,
  `archived_at`, and the existing `cf_<key>` set). Kola's repo also
  enforces this — if `create_custom_field` raises on a reserved key,
  echo the error and ask for a different key.
- `key` and `type` are **immutable** once set. Warn the user before
  creating that they cannot rename the key or change the type later
  (only `label`, `description`, and `position` are editable).

Then call `create_custom_field(key, label, type, description, position)`.

### `edit <key> [label="…"] [description="…"] [position=N]`
Call `update_custom_field`. `key` and `type` are not editable here —
if the user asks to change them, explain that they need to add a new
field and migrate values manually (`set_custom_field_value` per
person), then `delete <old-key>` once the new field is populated.

### `delete <key>`
Confirm with population stats first:

```
# Count current values
SELECT COUNT(*) FROM v_people_full WHERE cf_<key> IS NOT NULL;
```
(use `query_people`).

Then prompt:

```
Drop custom field 'foo' (N people have a value)?
The def is removed; existing values stay in custom_fields_json but
become invisible — they're garbage-collected on the next write to
each person. yes/no
```

If yes, call `delete_custom_field`.

## Examples

```
/kola:custom-fields list
/kola:custom-fields add check_size "Check size" number -- "Investor check size in USD"
/kola:custom-fields add met_at "First met" datetime
/kola:custom-fields edit check_size label="Typical check"
/kola:custom-fields delete met_at
```

## Output

One-line confirmation per write. `list` renders a table.
