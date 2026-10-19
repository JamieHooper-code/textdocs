---
tags: [programming, design-doc, stacks, projects, miller, ahk, caster]
created: 2026-10-15
updated: 2026-10-15
status: rebuilt 2026-10-15 — Stacks menu, triage (arrows + side list), whisper, Recently removed
related: ["[[SETTINGS_SYSTEM]]", "[[MILLER_GUI_PERFORMANCE]]", "[[REMINDERS_SYSTEM]]"]
---

# Stacks

A stack is somewhere to throw a thought and come back to it: "stack that" cuts
the field she is in onto the current stack, "pop it" pastes the top one back.
Fourteen stacks, ~70 items, `macros` the big one.

The 2026-10-15 rebuild was about **getting things back OUT**. Stacks had been
folded into the projects store months earlier, and in the process each item
became a *todo*: in the Projects menu a stack was one band, an item showed a
title, and reading it meant drilling in and choosing Open. The right pane showed
the item's todo actions (done / current / favorite), never its text. Removing,
reordering, moving and cleaning up were all a pain, so she stopped using them.

## What she can do now

| Say / press | Does |
|---|---|
| `open stacks` | The Stacks menu (own window). Every stack with its count, the current one first (●). |
| arrow through a stack | The item's **full text** on the right. Reading a stack is Down Down Down. |
| Enter on an item | Its page: **the options, then the full text underneath** (her layout: *Option / Option / Option / Full text*). Copy · Take it · Delete · Move to top · Move to bottom · Move to another stack · Make it a project todo · Add to TODO list · Edit the text. |
| `N.1`–`N.4` on an item row | Copy / Delete / To top / To bottom, without drilling. |
| typing in the filter | Searches **every word of every item** in every stack (one spawn — `stacks --items`). |
| `triage stack [<name>]` | One item at a time, full text on screen, the whole stack listed on the right (current row selected, a mark per decision, click/tap a row to jump). **→ / ←** move without deciding (wraps) · **↓** delete, **↓ again** puts it back in the same place · **0** delete · **1** keep (on a deleted one: put back + keep) · **2** another stack · **3** project todo · **4** TODO list · **9** undo the last. Bare spoken "zero".."nine" work in that window. |
| `stack whisper [<name>]` | A typing box with dictation started; Ctrl+Enter files it on top. Nothing is grabbed or cut. |
| Recently removed | Every pop, delete, dump and triage removal, newest first, with the stack it came from. "Put it back on <stack>" restores it. |

Unchanged on purpose: `stack that` (Ctrl+A, grab, cut, push), `stack that <name>`,
`stack <name>` (copy, not cut), `pop it [n]`, `stack push <n>`, `dump stack`,
`switch stack`, `add stack`, `save stack`, `show stack`, `list stack`.

## How it is built

```
voice / SD ──> StackFunctions.ahk (verbs) ─┐
open stacks ─> StacksViewer.ahk ─> StacksMenu.ahk ─┤── StackStore.ahk ──> projects.py stack-*
triage / whisper / edit ─> StackTriage.ahk (GuiHost) ─┘        (one command per op, under a lock)
Projects menu ─> a stack drills into StacksMenu's page
```

- **Data** stays in `INIDATA/Projects/projects.json`: a stack is a project node
  flagged `stack: true`, its items are child nodes, the LAST child is the top
  (= `pop it`, = #1). History is `INIDATA/Projects/history.json`, each row now
  carrying `stack` and `how` (pop/delete/dump/triage/todo-list) — that is what
  makes restore go back to the right stack.
- **One engine command per operation.** The AHK side used to read every item,
  edit an array and write it all back: two Python spawns per op, and a voice
  command landing between them was silently lost. Now `stack-take`,
  `stack-move`, `stack-edit`, `stack-to-todo`, `stack-dump`, `stack-restore`
  each do the whole thing in one process and print JSON (or `ERROR <why>`,
  which the tooltip shows). Items are addressed by `--id` (the menu) or `--n`
  (the pop number, voice).
- **A store lock.** Every projects.py command now holds an exclusive lock
  (`projects.json.lock`) from load to save, so two overlapping commands can't
  drop each other's write. The title worker takes it only around its own
  reload+save (it spends seconds in the local LLM). A lock that can't be had in
  8s runs unlocked rather than hang a voice command.
- **Items stay `todo` nodes.** The plan was a new `stack_item` type; it turned
  out unnecessary — the problem was the UI, and "make it a project todo" is then
  a plain reparent that keeps the id and title.
- **Process split.** The Miller viewer includes only StackStore + StacksMenu
  (+ JSON, settings). Anything that needs the wider world (writing box,
  pickers, the TODO-list writer, add-stack) is handed to MAINFUN. Triage,
  whisper and edit are windows, so they live in StackTriage.ahk and run in the
  GuiHost. `OpenStacks` lives with the verbs in StackFunctions, because the
  viewer must not include the launcher.

## Decisions worth knowing

- **Triage is not a Miller.** In a Miller a bare `0` closes the menu; in triage
  `0` (delete) is the most common answer. So it is its own window, digit +
  Enter like every numpad UI, and its own voice rule
  (`stack_triage_commands.py`, title context "Stack triage") that turns a bare
  spoken digit into exactly that.
- **No "(n)" prefix on item rows.** Items are the first rows of a stack page,
  so the Miller row number already *is* the `pop it` number.
- **A triage delete is real at once, and reversible in place.** Every item stays
  in the triage list whatever happens to it. Delete = `stack-take` (into
  Recently removed; it returns `history_index`). Putting it back =
  `stack-restore --at N`, where N is how many items still on the stack sit after
  it in the list — so it lands in its old slot, not on top (new node id). Chosen
  over "mark now, delete on close" so closing, a crash, or re-saying "triage
  stack" can never lose or half-apply her decisions.
- **Triage never closes itself** — she can go back to anything; after the last
  item it says so and waits. Esc closes and reports the counts.
- A TODO-list send can be undone on the stack side only — the list line stays
  and the footer says so.
- **The triage window is a fixed size.** Moving static rows in a Size handler
  left them unpainted; long items scroll inside the text box (PgUp/PgDn/Up/Down).

## Settings (`open settings` → stacks, or ⚙ in the menu)

- `stacks.wrap_chars` — line length of the full text under an item's options.
- `stacks.history_limit` — how many rows Recently removed shows.
- `stacks.todo_list` — which TODO list "Add to TODO list" / triage 4 writes to (default Macros).

## Files

`Scripts/projects/projects.py` (stack-* commands) · `Helpers/StackStore.ahk` ·
`Helpers/StackFunctions.ahk` · `Helpers/StacksMenu.ahk` · `Helpers/StackTriage.ahk` ·
`Scripts/StacksViewer.ahk` · caster `rules/stack_commands.py` +
`rules/stack_triage_commands.py` · tests `Scripts/codebase_tools/tests/test_stack_ops.py`,
`Helpers/Tests/Gui/test_stacks_menu.ahk`.

## Not done

- **The `created` dates are all 2026-06-15** — the migration stamp. The old
  `.txt` files never stored dates, so real ages are gone; "oldest first" means
  nothing for anything older than June.
- `stack whisper` was not exercised live (it starts Wispr dictation).
- Stack rename / delete-stack only exist in the Projects menu's own functions.
