---
name: lists
description: >
  Manage Kola lists — create a new list, rename one, delete one, add or
  remove people, and show what's on a list. The user is multilingual;
  trigger on Russian as readily as English. Use when the user says
  "create a list called X" / "создай список X", "add Y to my Z list" /
  "добавь Y в список Z", "remove Y from Z" / "убери Y из Z",
  "rename Z to W" / "переименуй Z в W", "show my Investors list" /
  "покажи список инвесторов", "what lists do I have" / "какие у меня
  списки", or "list everyone in <list>" / "покажи всех из <списка>".
argument-hint: '<create|add|remove|show|rename|delete|ls> [args]'
---

# /kola:lists

Thin command surface over Kola's list CRUD.

## Sub-commands

### `create <name>`
Call `create_list(name)`. Names may not contain commas (Kola enforces
this — `v_people_full` joins list names with commas). Echo the new
list's id.

### `add <list> <person> [<person>...]`
Resolve each `<person>` like `/kola:save-contact` does (LinkedIn URL →
email → name substring; one match → use it, many → ask, none → offer to
capture via `/kola:save-contact` and resume). Resolve `<list>` by name
via `list_lists`. Call `add_to_list` per person. Report `n added,
n already on list, n skipped (not resolved)`.

### `remove <list> <person> [<person>...]`
Mirror of `add` — calls `remove_from_list`. Removing someone from a
user list does **not** archive them; that's a separate
`archive_person` call.

### `show <list>` (alias: `ls <list>`)
Resolve the list by name via `list_lists`, then `get_list(list_id)` to
fetch members. Render as a table: name, position @ company, last
interaction (max of email / telegram / whatsapp / linkedin / calendar
timestamps).

### `rename <list> <new-name>`
Call `rename_list`. Same comma rule as create.

### `delete <list>`
Confirm first ("Delete list 'X' (N members)? Members are not archived,
just unlisted. yes/no") then call `delete_list`.

### `ls` (no args)
Call `list_lists`. Render as: name · member count · created date.

## Behaviour rules

- **Never create a list silently.** If `add` references a list that
  doesn't exist, ask before creating.
- **Suggest, don't decide, when names collide.** Kola list names aren't
  unique by default — if `list_lists` returns two lists with the same
  name, ask which one.
- **Batch where natural.** "add Jane, John, Sam to Investors" is one
  resolve loop + one `add_to_list` call per person, not three trips
  through this skill.

## Examples

```
/kola:lists create Investors
/kola:lists add Investors jane@acme.com John Smith https://linkedin.com/in/sam
/kola:lists show Investors
/kola:lists ls
/kola:lists rename Investors "Seed Investors"
/kola:lists remove Investors John Smith
/kola:lists delete "Seed Investors"
```

## Output

Per sub-command: one-line confirmation (`create`, `add`, `remove`,
`rename`, `delete`) or a rendered table (`show`, `ls`).
