---
tags: [programming, exports, journal, quotes, caster, ahk, markdown, obsidian, writing]
created: 2026-08-16
related: ["[[JOURNAL_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[SETTINGS_SYSTEM]]"]
---

# Export system — JSON stores back into readable documents

A generic layer that turns any of the text stores into one big markdown file in
the Obsidian vault, driven by **named presets stored as data**.

See also [[JOURNAL_SYSTEM]] (the first consumer) and [[QUOTES_SYSTEM]] (the
second, and the one that proved the split was real).

## Why it exists

Jamie stopped using the journal. Not because the store was wrong — the tagging,
the people links and the depth model all worked — but because of this:

> it feels so weird to me to have my journal broken up into little sections
> instead of being something that I can scroll through

A JSON file with 96 entries in it is not a journal you read. The Google Doc it
replaced *was*. So the system's job is to rebuild that Doc on demand: year
headings, long dates, titles, the whole thing scrollable in one buffer.

The second requirement was that this not be a journal feature:

> this system needs to be generic so that I can use it for my quotes, or I can
> use it for my poetry, or I can use it for my journal. Each different variation
> will have different options.

## What is shared and what is not

The three stores have genuinely different item shapes. A single
schema-agnostic renderer would be fake generality — so the split is:

| Layer | Owns |
|---|---|
| `Scripts\exports\exports.py` | the preset registry, the Miller feed, the voice cache, the auto-regenerate sweep |
| `Scripts\exports\adapters\<system>.py` | which items a filter set SELECTS, and what they look like as markdown |
| `Helpers\ExportsMenu.ahk` | the shared Miller node, mounted by both stores |

**The asymmetry between the two adapters is the proof the split is right.**
`journal.py` already had a renderer, so its adapter is a thin translation layer
and rendering stays in journal.py (which owns the segment/role/depth schema and
would drift instantly if a renderer lived elsewhere). `quotes.py` had **no
markdown renderer at all**, so its adapter supplies one — grouped by provenance
(book → author → the gatha/collected buckets), because that is how a quote is
found again: "the thing Lorde said", never "the thing I saved on a Tuesday".

## A preset is DATA, not code

`INIDATA\export_presets.json`. A new document costs a row in a JSON file:

```json
"journal-essential": {
  "system": "journal",
  "label": "Essential",
  "spoken": ["essential"],
  "filters": {"min_importance": 10},
  "options": {"depth": "summary"},
  "out": "Journal — essential.md",
  "auto": false
}
```

Unknown keys **round-trip untouched** — same north star as `sends.py` and
`dump_presets.py`, so a preset written by a future Miller is not lopped off by
an older CLI.

Seeded set: `journal` (the default), `journal-dream`, `journal-classic`,
`journal-ifs`, `journal-essential` (★10), `journal-notable` (★7+),
`journal-everything` (full depth, transcripts), `quotes-all`, `quotes-poems`.

### Newest first

Every document renders newest-first (`oldest_first: false`), year sections and
contents included — so `Journal.md` opens on today and `## 2026` is the first
heading. With 116 entries and ~694k characters, oldest-first meant every open
landed in 2020 and needed a scroll to the bottom. The quotes documents order
their groups by most-recently-added quote for the same reason, but order WITHIN
a book is left alone: that is capture order, which for a book is reading order.

### "Has a chat" is not "is a chat" — `exclude_chat_only`

The default document originally excluded `entry_types: ai-conversation`, and
that was wrong twice over.

First, `classify_entry` derived `ai-conversation` from a bare chat **link**.
Jamie's IFS meditations often end with the url of the conversation she had
afterwards, so five of her own entries were filed as captured conversations and
**silently vanished from the journal she reads**. `ai-conversation` now requires
an actual `ai`/`prompt` SEGMENT — the conversation has to be *in* the entry. A
link is a reference.

Second, once a conversation is correctly filed *under* an entry, that entry
legitimately becomes `ai-conversation` — and excluding on the type would drop it
again. But her rule is that such an entry appears as "the original and then a
link to that chat", so it belongs in the document. What does not belong is a
bare ChatGPT dump with no writing of her own.

So the filter is `exclude_chat_only`, backed by `is_chat_only()`: an entry whose
segment roles contain no `body` or `commentary`. `ensure_core` guarantees
*something* sits at core depth, so the test has to be the ROLE, not the depth.

### The default document is SUBTRACTIVE, deliberately

Jamie described the default as "Dream, classic, and IFS, but not the ChatGPT
ones". It is stored as `exclude_types: [ai-conversation]` rather than
`types: [dream, classic, ifs]`.

The difference matters the day she invents a new form. She said she would:

> If I come up with a new journaling type, like Exercises or something … then I
> can very easily add the voice command and add the type

An include-list would silently omit every entry of that new type until someone
remembered to edit the preset. The subtractive form picks it up for free.

`query()` also applies **exclusion before inclusion**: an entry that is both
`classic` and `ai-conversation` is a captured chat, and the default document is
meant not to contain it. The other ordering would let every such entry back in
through the `classic` door.

## The document

Rendering lives in `journal.render_document`. Shape:

```markdown
# Journal
*96 entries · 2020-09-26 – 2026-08-15 · everything you actually wrote*

> [!abstract]- Contents          <- collapsed, at the TOP
> **2026**
> - [[#Wednesday, July 29, 2026 — aggressive therapies for chronic pain]]

## 2026                          <- year section
### Wednesday, July 29, 2026 — aggressive therapies for chronic pain
*[classic]  #grief #parts-work  ★8*

…her writing…

> [!summary]- Summary            <- collapsed, at the BOTTOM
> …the condensed takeaway…

🔗 [ChatGPT conversation](https://chatgpt.com/…) · 8 turns
```

Heading levels nest `#` document → `##` year → `###` entry, so Obsidian's
outline pane gives a real tree.

### Her writing on top, summaries at the bottom — a RENDER order, not a storage one

Jamie's rule: *"I want the summaries to always be at the bottom and the real
content always at the top."*

`render_entry` splits the segments into writing (`body`, `commentary`,
`quoted`, `prompt`) → summaries → transcript, regardless of stored order.

Storage is deliberately **not** changed to match. `set_summary` still inserts
the summary segment directly above the first `ai` segment, because `chat-log`
reads the conversation in stored order and a summary hoisted to the end of the
array would read out of sequence there. **Storage keeps chronology; the document
keeps readability.**

### Folding

Obsidian callouts, where `-` after the type means collapsed by default:
`> [!abstract]- Contents`, `> [!summary]- Summary`. They degrade to plain
blockquote lines in Notepad++ — the trade Jamie chose, since these files live in
the vault and folding there beats portability nowhere.

**The chat link is deliberately NOT folded.** It is one line; folding it buys
nothing. Her rule for a chat-bearing entry in a document is "the original, and
then a link to that chat" — never the transcript, which is mostly filler.
`chat_link_lines` dedupes on url, because every segment of one conversation
carries the same share link and printing it per segment put it on screen three
times in a four-segment entry.

## Auto-regeneration

A document she has to remember to regenerate is out of date every time she opens
it. So `journal.save_store` sets a dirty flag and `main()` fires
`exports.py generate-auto --system journal` **detached** on the way out.

Three properties, each load-bearing:

- **One spawn per CLI invocation, not per write.** A command that saves in a
  loop (`people-scan`, `retype`, `tag-batch`) regenerates exactly once.
- **Detached and non-blocking.** Rendering 96 entries takes about a second; that
  must never sit in front of a tag toggle in the Miller.
- **Never fatal.** A broken preset or a locked file must not turn a successful
  `add journal` into an error — the entry is already safely on disk.

`JOURNAL_NO_REGEN=1` suppresses it (and guards the child against recursion).

## The Miller node is system-agnostic

`Helpers\ExportsMenu.ahk` contains **no journal vocabulary and no quote
vocabulary**. Every filter row, its widget and its option list come from
`exports.py axes --system X`, which each adapter derives from its live store —
so a type or tag invented at runtime is pickable immediately, and adding a
filter axis to an adapter makes it editable with no AHK change at all.

Enter on a preset **generates** (the common case); drilling gives Generate ·
Which entries · How it looks · Regenerate automatically · Output file · Delete.

Generated files inside the vault open via `obsidian://open?vault=…`, not the
default `.md` handler — the whole point of generating them is the app with
folding callouts, backlinks and search.

### STYLE TRAP: one-line IIFEs

Every closure in ExportsMenu is a single-line `((x) => (c, k, close) => …)(v)`,
matching the other Miller menus. Splitting one across lines is **not** cosmetic:
a continuation line beginning with `(` is read by AHK as the opening of a
continuation SECTION. It passes `validate` and then misbehaves at runtime. That
cost this menu one silently empty level during the build.

(The empty level had a *second*, unrelated cause worth knowing: each level fills
via `RunWait` shell-outs to Python, so a screenshot taken 6s after launch caught
a half-filled pane. `ahk.py show --no-launch` on the settled window is the
check.)

## Deleting an entry

`🗑 Delete this entry` on the entry page, always confirmed, default button
**Keep**. The modal names the entry (title + date) rather than saying "this
entry" — the row it fired from is off screen once the confirmation has focus,
and "are you sure?" with nothing to check against is how the wrong thing gets
deleted.

It is a **soft** delete: `status=deleted`, which `query` filters out, and the
words stay in `journal.json`. Nothing in the Miller can hard-delete; that needs
`journal.py remove <id> --hard` at a prompt, deliberately.

## Voice

`rules\export_commands.py` — `"journal generate <journal_doc>"` /
`"quotes generate <quote_doc>"` → `GenerateDocument(preset)`. Domain word first,
per the house convention.

The Choices are built from `~/.claude/context/export_presets_dump.json`, which
every mutating `exports.py` command rewrites — so a document created in the
Miller is speakable without a manual refresh. Same contract `sends.py` has with
`sends_dump.json`, and the same silent failure if it is skipped: Dragon just
types the words.

Choice keys are **sanitised** (`IFS / parts work` → `ifs parts work`): punctuation
in a key raises `GrammarError`, which kills the entire rule, not just that row.
Each preset contributes its label plus its `spoken` aliases, so the IFS document
answers to `ifs`, `internal` and `parts work`.

## CLI

```bash
py Scripts\exports\exports.py list [--system journal]
py Scripts\exports\exports.py axes --system journal      # the whole editor vocabulary
py Scripts\exports\exports.py generate journal --open
py Scripts\exports\exports.py generate-auto --system journal
py Scripts\exports\exports.py add "Poetry exercises" --system journal
py Scripts\exports\exports.py set-filter <name> types poetry-exercise
py Scripts\exports\exports.py set-option <name> depth core
py Scripts\exports\exports.py seed [--force]
```

Output location is the declared setting `journal.export_dir` (default
`ObsidianVault\Journal`), per the standing rule that a Miller's tunables are
settings rather than constants — see [[SETTINGS_SYSTEM]].

## Open threads

- **Poems** are still notionally the third store. Right now poems live in the
  quote store as `entry_types: [poem]` and the `quotes-poems` preset covers
  them; a dedicated store would just need a third adapter.
- **Per-year split documents** (`split: year`) are designed for in the preset
  schema but not implemented — one file is currently always one document.
- **`journal generate` does not yet regenerate everything** — it runs one
  preset. `generate-auto` is the sweep, and only `auto` presets are in it.
- **Bad provenance data shows up in the quotes document**: one book renders as
  `Schwartz. No Bad Parts by Richard C. Schwartz — Richard C`, a title/creator
  split error in `library.json` rather than a renderer bug.
