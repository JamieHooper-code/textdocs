---
tags: [programming, journal, letters, writing, caster, ahk]
created: 2026-11-12
related: ["[[JOURNAL_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[PEOPLE_SYSTEM]]"]
---

# Letters — writing to people, and keeping what comes back

Jamie writes letters to people (Charli, Cedar…) and wants them kept with their
replies, as **chains**: I write Charli, Charli writes back, I answer that. Some
letters stand alone. Built 2026-11-12 (systems item t-807a2484).

## The one design decision: a letter is a journal entry

Not a new store. A letter is a journal entry whose `source` is kind `letter`
with a `person_id`, which already existed (`journal letter <name>`, "Letter to
Cedar #3"). So numbering, the journal Miller, exports, the question loop and
"ask an AI about this" all work on letters without any extra code. Three things
were added:

| Field | Where | Meaning |
|---|---|---|
| `source.dir` | source | `""` = she wrote it (TO the person), `"from"` = one she received |
| `reply_to` | entry | the entry id of the letter this one answers. Following these links gives the chain |
| `special` / `favorite` | entry | from `{special}` / `{favorite}` markers ("mark special"), same words poems use |

Received letters are **numbered apart**: "Letter from Charli #2" doesn't take
the number of her next "Letter to Charli". `source_key` returns `letter-from:<id>`
for those.

## Saying it

| Say | Does |
|---|---|
| `add letter` | blank letter box. Say **`to Charli`** inside it |
| `add letter to <name>` / `journal letter <name>` | letter box already addressed |
| `add reply` | save a letter you got: paste it in, say **`from Charli`** |
| `add letter from <name>` | the same, already knowing who |
| `open letters` / `open letters <name>` | the Letters Miller |
| `mark Charli` (in the box) | who it's with, like `tag`: to, or from in an `add reply` box (`{with Charli}`) |
| `mark special` (in the box) | goes on the ★ Special letters list |
| `put poem` (in the box) | pick one of your poems; it goes in between `{poem}` / `{end poem}` with its title |
| `start poem` / `end poem` | write a poem straight into the letter |

The letter box has its own left panel (`INIDATA/WritingPanels/letter.json`,
which includes the journal's panel).

## On save (Ctrl+Enter)

`LetterFinishSource` (`Helpers/JournalSources.ahk`):

1. Takes `{to X}` / `{from X}` out of the text and resolves the name
   (`journal.py letter-person`: exact name/alias, else a unique prefix).
2. Nobody named, or a name that matches several people → a **pick list**:
   people she writes with first, then everyone.
3. If that person has letters going the other way → **"which letter is this
   answering?"** Enter = the newest, or "Not a reply". Backing out = unlinked,
   so a letter is never lost.
4. Then the normal journal title prompt, prefilled "Letter to Charli #3: ".

Backing out of step 2 keeps the box's draft; `add letter` offers it back.

## Open letters (the Miller)

`Helpers/LettersMenu.ahk` + `Scripts/LettersViewer.ahk`. Root: Write a letter ·
Save a letter you got · ★ Special letters · one row per person (most recent
first, "3 to · 2 from"). A person's level shows their letters **as threads**:
each chain's first letter, then its replies indented under what they answer
(→ = hers, ← = received). On a letter: Read · Reply (opposite direction,
already linked) · Edit · Mark special · Link it to the letter it answers
(pick, or "not a reply") · Open in the journal (every journal action).

## Engine (`Scripts/journal/journal.py`)

`letters [--person ID | --special]`, `letter-info --id`, `letter-person --name`,
`letter-reply-options --person --dir`, `set-reply --id (--to ID | --clear)`,
`set-flag --id --flag special|favorite [--off|--toggle]`, `add --reply-to`,
`--source-dir from`. Tests: `Scripts/codebase_tools/tests/test_journal_letters.py`.

## Printing and images: the poem studio (2026-11-12)

A letter gets everything a poem has in the studio: print (size, font, margins,
pages, parchment), the Instagram shape (paper, PNG slides), pictures (clipart
picker, archive, online), the folder, versions and the finished/posted marks.
Letters menu → a letter → **Print studio… · Print now · Instagram images…**

- **How:** the letter stays in the journal. `poems.py letter-sync <journal id>`
  makes or refreshes a **print copy** in the poem store, kind `letter`, kept in
  `data["letters"]`, **not** `data["items"]`. So the poetry library, the Google
  Doc sync, poem of the day, tags and series never see letters (they all walk
  `items()`); only `poems.find()` looks there, which is all the studio needs.
  The Doc push refuses a letter twice over (`_schedule_doc_push` and `push`).
- **The text is copied, never edited there:** every studio load re-copies it from
  the journal (`_fresh_letter`), through `set_text`, so each change is a numbered
  version (v1, v2…). The studio's Edit button opens the journal's edit box.
- **Folder:** `E:\Media\Poetry\letters\<slug>\` (beside `poems\`), via
  `poem_assets.folder()`.
- **Starting point:** a new letter's studio settings are seeded once: no title,
  no byline, print max 14pt (a short letter otherwise auto-grows to 36pt). All of it
  changeable in the studio.
- **A poem inside a letter:** `poems.letter_stanzas` turns `{poem}`…`{end poem}`
  into stanzas flagged as verse; `poem_print.poem_payload` sends `verse` +
  `wrapDefault: 'prose'` for letters only; the engine (`poem_layout.js`) wraps the
  letter's paragraphs as prose and keeps the poem's lines as verse (a too-long
  verse line turns, like in a poem). The poem sits **indented 2em** (engine
  option `verseIndentEm`, 0 = flush; none when centred), done through the same
  per-row room a wrap-image gives, so the preview, PDF, PNGs and click boxes agree. Poems carry neither field, so they lay out
  exactly as before. Tests: `test_letter_print.py`.

## Not built yet / ideas

- Letters' images in the "Poem images" gallery (`flyers.entries` walks poems only).
- The edit box shows the "who it's with" rows, but only a new letter reads them.

- Exporting a whole chain as one document (export presets could take a person).
- Letters that went to several people at once.
