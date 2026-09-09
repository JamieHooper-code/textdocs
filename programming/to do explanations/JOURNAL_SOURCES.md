---
tags: [programming, journal, caster, ahk, reading, media-system, writing]
created: 2026-09-04
related: ["[[JOURNAL_SYSTEM]]", "[[JOURNAL_QUESTION_LOOP]]", "[[MEDIA_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[COMPLETION_LOG]]", "[[PEOPLE_SYSTEM]]"]
---

# "journal this" — writing about the thing you're looking at

Jamie says **"journal this"** while reading a book, sitting with an exercise,
watching a video or talking to an AI, and the writing box opens already knowing
what the entry is about — numbered into that thing's series, linked to it in the
media library, and reachable from both ends.

Built 2026-09-04 on top of [[JOURNAL_SYSTEM]].

## The one idea: `source`

Every journal entry has carried an empty `source: {}` since the store was built.
It is now the answer to *what was I responding to?*

```json
"source": {
  "kind": "book",
  "book_id": "isbn:9781035064656",
  "title": "Bury Our Bones in the Midnight Soil",
  "creator": "V. E. Schwab",
  "ref": "loc 1420",
  "seq": 2
}
```

**The shape is the quote store's shape** — `kind` / `book_id` / `title` /
`creator` / `ref`. A journal entry about a book and a highlight from that book
then describe themselves identically, which is what lets both hang off the same
catalog id with no translation layer. Adding a parallel vocabulary here would
have been the easy mistake.

| Kind | Carries |
|---|---|
| `book` | `book_id`, `title`, `creator`, `ref` (Kindle location or `ch 12`), `chapter`, `seq` |
| `exercise` | `quote_id`, `title`, `book_id`, `ref`, `seq` |
| `youtube` | `url`, `title`, `creator` |
| `reddit` | `url`, `title`, `creator` |
| `chat` | `url`, `title`, `creator` (the model) |
| `letter` | `person_id`, `title` (their name), `seq` — numbered PER PERSON |

### Provenance is not subject

`source` now carries **two different relations**, and only one of them is
browsable:

- **Provenance** — where the text came from. The importer stamps all 116 entries
  it pulled out of the Google Doc with `kind: "google-doc"`. True, worth keeping,
  and useless as a way to browse: it would put a single row holding two thirds of
  the journal at the top of a menu whose whole point is *what was I writing
  about?*
- **Subject** — what the entry is about.

`SUBJECT_KINDS` in journal.py draws that line. Keeping both in one field is
deliberate — a second field would have to be kept in step for no gain — but the
distinction has to live somewhere, and that is where.

### `seq` is stored, never counted

"Bury Our Bones in the Midnight Soil **#2**" is numbered by what the source has
already produced. Recounting live means deleting #2 makes the next entry #3 a
second time. It is written once, at add time, **under the same lock as the
write** — two entries started seconds apart would otherwise both come out as #3.

An **exercise numbers per exercise**, not per book: a practice you return to is
its own thread, while the book it came from is a shelf.

## Detection is data, not a chain of ifs

A context declares how to journal it, in `INIDATA\Contexts\<token>.json`:

```json
"journal": { "source_fn": "JournalSourceKindle" }
```

`JournalSourceForContext` walks `DetectContextChain()` — which comes back
`[deepest..root]` — and calls the first `source_fn` it finds. So `youtube_watch`
beats `youtube` beats `chrome` for free, and teaching this command about
Letterboxd is one JSON line plus one function, with no edit to `journal this`.
Exactly the escape hatch `post_click_function` / `nth_click_function` / `open_fn`
already are elsewhere in the context registry.

**No block in the chain means a plain `add journal`.** A command that guessed at
an unknown window would be worse than one that quietly does the ordinary thing.
A resolver that *declines* (wrong page, nothing open) is not an error either —
the walk continues outward, which is what lets `youtube` sit under `chrome`.

Wired today: `kindle`, `reading_room`, `youtube_watch`, `reddit`, `chatgpt`,
`claude_web`.

## What each resolver stands on

Nothing here is new machinery — every one of these already existed:

- **Kindle** — `_KindleOpenBookId(hwnd)` resolves the open book to a catalog id
  (pinned ASIN → title cascade).
- **Reading room** — `/api/state` returns `item: {id, title, ref, book_id}`, and
  that one endpoint covers **both** books and exercises. The discriminator is
  `ref`: a chapter records `ch 12` (`reader_collection.chapter_ref`), anything
  else is an item out of the quote store. Same test `OpenReaderInKindle` already
  makes, so the two cannot drift. Reading a book, `item.title` is the **chapter's
  label** — which is the only place the chapter exists, and why it is recorded.
- **YouTube / Reddit / chat** — the tab URL plus the window title.

### The Kindle location — measured, not guessed

Tested against a live book on 2026-09-04. Four routes, three of them dead:

| Route | Result |
|---|---|
| **UIA, windowed** | **WORKS.** The footer is a real Text element: `23%       Location 1822 of 8347` at `[1, 1368]`. Free, invisible, touches nothing. |
| UIA, fullscreen | **Nothing.** The footer is not hidden — it is not in the tree at all. The whole page is one `[document]` of pixels. |
| Select and copy | **Does not work, and costs her place.** `Shift+Right` in the Kindle reader does not extend a selection, it **turns pages**: `ClipWait` timed out with an empty clipboard, and the book moved from location 1822 to 1866. This is the route that looked obvious on paper — a quote gets its location from the citation Kindle appends on copy — and it is exactly why it was worth testing rather than writing. |
| `Ctrl+G` "Go to" | Gives the book's **size** (`Location [ ] of 8347`), never the current position. |

So: read it directly, and when that comes back empty **because the window is
fullscreen**, drop out of fullscreen for a beat, read, put it back. That peek is
the only thing that works there, and it is honest about its cost — ~2.5 s and two
window resizes. It changes nothing else: no clipboard, no selection, no page turn
(verified — 1822 before and after).

The peek is gated on the window **actually being fullscreen**
(`ForegroundFullscreenWindow`), so a read that failed for some other reason can
never resize a window she is reading in, and it is switchable off via
`journal.kindle_location_peek`.

A missing location is not a failure — the entry still knows the book.

### Reading a window title is the part most likely to be wrong

Splitting on the last `" - "` is right for `<video> - <channel>` and wrong for a
title that merely contains a dash. Two guards, and the second is the one that
matters:

- short enough to be a name (≤ 40 chars), and
- **starts with a capital.** A channel name is a name; the second half of a
  sentence continues a sentence, and continues it in lower case.

Measured against *"Why Rest Is Hard - and What Nobody Tells You About It"*, whose
tail is only 33 characters and sails straight past a length check alone. The cost
is a genuinely lowercase channel name staying in the title — a harmless miss,
where the other direction files an entry under half a sentence.

Pinned in `Helpers\Tests\test_journal_sources.ahk`.

## The writing box

- The **title bar** names the source — "Journal entry — Bury Our Bones in the
  Midnight Soil" — so a box opened on the wrong book is obvious before she writes
  into it.
- The **header** is the resolver's line, placed in the body with two blank lines
  under it. It is *her text*: visible, editable, deletable. The structured
  `source` block is the separate thing the journal browses and counts by.
- The **title prompt prefills with the series stem** — `Bury Our Bones in the
  Midnight Soil #2: ` — caret after it, so she only names this one. The trailing
  space is load-bearing; trimming it makes her type a space every time.

The source crosses into the writing box as a **JSON file path**, because that box
runs in GuiHost and a process boundary only passes strings — the same reason
journal.py takes `--text-file`. It is deleted on read: a stale hand-off left on
disk is a journal entry filed under last week's book.

## Voice

| Phrase | Does |
|---|---|
| `journal this` | contextual — the book, exercise, video or chat in front |
| `journal <book>` | a new numbered entry for that book, from anywhere |

`journal <book>` shares `reading_books.json` with `read <book>`, so
"read midnight" and "journal midnight" can never mean different books.

### The GrammarError hazard, and the guard

`journal <journal_type>` already existed, so `journal <book>` puts a **second
Choice behind the same word, in one rule**. Dragon raises a `GrammarError` on a
key present in both — and that does not break the phrase, it takes down the
**whole rule**, silently, so "open journal" and "add journal" stop working too.

`_book_choice(taken)` drops any book word the type Choice already claims.
`parts` (No Bad Parts) against `parts work` (IFS) is the near-miss that makes
this real rather than theoretical. Overlap is currently zero and is asserted, not
assumed.

## Browsing, both directions

- **Journal ▸ By source** → Books / Exercises / Videos / Reddit / AI
  conversations → the thing → its entries, **oldest first**: a numbered series
  reads the way it was written.
- **Reading ▸ a book ▸ Journal entries (3)** → the newest entry in that series.
  Count read live, because a stale "(3)" beside a series she just added to is the
  kind of quiet wrongness that makes her stop trusting the number. No entries yet
  → the row starts one instead.

Both resolve the **same catalog id**, so a book and what she has written about it
cannot drift apart.

## The conversion

Her existing *"Bury Our Bones in the Midnight Soil #1: Maria and Sabine"* — she
was already using the `#N` convention by hand — kept its title verbatim and
gained `source = {kind: book, book_id: isbn:9781035064656, seq: 1}` plus the
`response` entry type. The next one comes out `#2` on its own.

## Entry type

One new type, **`response`** ("me responding to something I read, watched or
did"), spoken as `response` / `reading log` / `about`, with no title prefix — the
title already carries the book. `source.kind` is the finer distinction, and the
"By source" branch is what actually browses it.

## Files

| Layer | File |
|---|---|
| Store | `Scripts\journal\journal.py` — `source_key`, `next_seq`, `set-source`, `source-next`, `sources`, `viewer --source-key` |
| Resolvers | `Helpers\JournalSources.ahk` |
| The box | `Helpers\JournalCapture.ahk` — `AddJournalEntryForSource`, `_JSourceArgs`, `_JSourceTitleStem` |
| Browse | `Helpers\JournalMenu.ahk` — "By source", `OpenJournalForBook` |
| Book side | `Helpers\ReadingMenu.ahk` — `_ReadingJournalNode` |
| Contexts | `INIDATA\Contexts\{kindle,reading_room,youtube_watch,reddit,chatgpt,claude_web}.json` |
| Voice | `rules\journal_commands.py` |
| Tests | `Helpers\Tests\test_journal_sources.ahk` (pure) |
