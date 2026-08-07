---
name: custom-fields
description: >
  Manage Kola's per-install custom-field schema — the user-defined
  extension columns on every person. Add a new field (text / number /
  boolean / datetime / file), rename or describe an existing one,
  re-order, or delete one. The user is multilingual; trigger on Russian as
  readily as English. Use when the user says "add a custom field for X" /
  "добавь кастомное поле для X", "track Y on every person" / "храни Y у
  каждого человека", "change the label for Z field" / "переименуй поле Z",
  "what custom fields do I have" / "какие у меня кастомные поля", or
  "drop the W field" / "удали поле W".
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
- `key` must not collide with a built-in `v_people_full` column.
  **Read the live list — never a list written here.** Call
  `describe_people_schema` and treat every column it returns as taken,
  including the existing `cf_<key>` set.

  There are around sixty built-in columns and Kola adds more as it
  learns new channels, so a copy of the list in this file goes stale
  and then misleads in both directions: it warns about names that are
  free and stays silent about names that are not. `country`, `city`,
  `headline`, `slack_dm_count` and `last_interaction_at` are all
  built-in, and none of them looks reserved.

  Kola's repository enforces this too. If `create_custom_field` raises
  on a reserved key, echo the error and ask for a different key.
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
