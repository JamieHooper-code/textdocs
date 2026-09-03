---
tags: [programming, reading, books, ahk, caster, epub, media-system, quotes, design-doc]
created: 2026-09-01
related: ["[[QUOTES_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[COMPLETION_LOG]]", "[[ART_SYSTEM]]"]
---

# Reading whole books in the reading room

The reading room started as a viewer for what was **grabbed out of** books —
practices, poems, quotes, every one of them a record in the quote store
([[QUOTES_SYSTEM]]). It now also reads **the books themselves**, off the EPUBs
already sitting in the library, so moving reading off Kindle is a per-book
decision rather than an all-or-nothing switch.

Built 2026-09-01. Kindle is still the default and still fully supported —
nothing was taken away.

## The one idea

**A chapter is an item.**

The reader's whole model is an ordered list of items, each with an id, a title
and text. Pagination, the per-collection bookmark, Left/Right, the numbered
grab, highlight painting and the affinity keys are all written against that
shape. So a book was made readable by producing *items of that shape from an
EPUB* — and nothing downstream had to learn what a book is.

That is why this was a small change to the reader and a new module beside it,
rather than a second viewer.

| Layer | File | Job |
|---|---|---|
| The file | `Scripts/reader/book_text.py` | EPUB → ordered chapters (`id`, `title`, `section`, `text`) |
| The set | `Scripts/reader/reader_collection.py` | the `read:<book id>` spec → items, same shape as a practice |
| The state | `Scripts/reader/read_server.py` | unchanged, plus three guards (below) |
| The doors | `Helpers/ReaderFunctions.ahk` · `Helpers/ReadingMenu.ahk` | `OpenBookInReader`, the Miller's "Open in the reader" |
| The choice | `Helpers/KindleReading.ahk` | `ReadBookHere` — which surface a book opens on |

## `book:` and `read:` are not the same thing

Read the specs carefully, because the names are one letter apart in meaning:

- `book:<title>` — everything she **grabbed out of** that book, from the quote store.
- `read:<book id>` — **the book**, from its file on disk.

They are keyed differently on purpose. A title identifies a book well enough to
group quotes under it; only the catalog id can find the file.

## Where a book is read is a property OF THE BOOK

`read_surface` in `library.json` — `"reader"`, or absent for Kindle.

**The transition is the open.** There is no toggle to flip and remember:
opening a book in the reader makes the reader its surface, opening it in Kindle
makes Kindle its surface, and `read <name>` follows. Every way of opening a book
already says which surface she meant, so asking again would be asking a question
she just answered.

```
"read midnight"  ->  ReadBookHere(id)
                       read_surface == "reader"  ->  OpenBookInReader   (falls back to Kindle if unreadable)
                       else                      ->  ReadBook           (Kindle)
```

`ReadBook` and `OpenBookInReader` each stay a **single-surface** opener — that
is exactly what the explicit "Open in Kindle" / "Open in the reader" buttons
need. Only the *choosing* is new, and it lives in one function so the two
surfaces cannot grow rival versions of "open a book".

The fallback matters: a reader open that fails is always "this book has no
readable file" (a PDF, a MOBI, a Kindle-only purchase), and in every one of
those cases Kindle is the right answer, not an error message.

`clog book-opened <id> --surface kindle|reader` is the one writer, and it stamps
`last_opened` in the same call — they are one event ("this book was just opened,
here, now"), and a caller that recorded the surface without the time would leave
Current reads sorted by a history the surface no longer agrees with.
`book-get` col 8 and `book-current` col 7 report the surface, and Current reads
marks the moved books with `· in the reader`. Only the moved ones — Kindle is
most of the shelf, so marking those too would be noise on every row and would
hide the handful the mark exists to show.

## Current reads is ordered by recency

Twenty current reads is a shelf, not a list. Alphabetical put *A Psalm for the
Wild-Built* at the top every single time and buried the two or three books she is
actually in the middle of — the only question that level answers.

`book-current` now sorts newest-opened first (never-opened last, alphabetically
among themselves) and ships `last_opened` as col 8. The **recent / older split**
is drawn in the Miller, at `reading.recent_days` (default 7), with the house
divider between the groups — and only when both groups have something in them,
because a lone `— OLDER —` above a shelf that is entirely older says nothing.
The recent group has no header of its own: the line marks where recency ends,
which is the whole information.

**The age leads the detail column.** It is ~320px and truncates, and appended
the date was the first thing lost — every row past the third showed an author
and no date at all, in a list whose whole point is the order.

`last_opened` only exists from the moment the field did, so it falls back to the
**completion log**: every `read <name>` writes a `reading` entry, and the newest
one per title seeds the order. Without that fallback every book would have sorted
into "Older" until she happened to open it again — a recency list that starts out
saying nothing. It keys on the title (that is what the entry records), so a
renamed book simply misses; a miss costs a position in a list, never a wrong
book.

## What survives the move, and what doesn't

**Grabbing works, and it is better than Kindle's.** The Kindle path OCRs the
screen and rebuilds line breaks from word geometry because Kindle's clipboard is
one flat line ([[QUOTES_SYSTEM]] § the reflow). Here the structure *is* the
document, so the text arrives exact — same `quotes.py add`, same book settings,
same capture modes, same tagger, plus an anchor that paints the highlight back
onto the page. A quote grabbed from the reader and one grabbed in Kindle are the
same kind of record.

**Three things need a store record and say so** rather than failing somewhere
deep: affinity (a chapter has nothing to score), the Miller hand-off (it browses
practices), and the Kindle location jump. Each is guarded in `read_server.py`
*and* on the AHK side, because the page button and the voice command are
different callers.

**A chapter's `ref` is `ch <n>`, not a Kindle location.** It orders reader grabs
among themselves correctly, and the `ch` is the tell that stops
`OpenReaderInKindle` handing `12` to Kindle's Go-to dialog as a location.

## What the EPUB reader does to the text

Formatting is **flattened on purpose** — the reader has her fonts, her measure,
her page colour, and re-applying a publisher's stylesheet would fight it. What
survives is the structure that changes how it *reads*:

- one line per paragraph (the reader's own convention)
- `<br/>` → a line of its own, so **verse stays verse**
- headings → `[[h2 ...]]` / `[[h3 ...]]`, rules → `[[hr]]` — the block vocabulary
  the page already draws
- images, classes, drop caps → gone

**Chapter names come from the contents page**, not the document: inside a
chapter the heading is often "I" or an ornament, while the TOC is where "Chapter
One" lives. Both EPUB 3 nav documents and EPUB 2 NCX files are read.

**A part title page is usually an image with no text at all.** Dropped silently
it took the book's structure with it — *Bury Our Bones in the Midnight Soil* has
three interleaved threads whose chapters all restart at I, so nine items in a
row read "Chapter I" with nothing to say which thread they were. The dropped
page's own name is now carried onto the chapters under it: `MARÍA: (D. 1532) ·
Chapter I`.

**A named document is kept however short.** A length-only floor swallows a
three-line poem while keeping the copyright page — exactly backwards. Named
front-matter boilerplate (cover, contents, copyright, "also by") is dropped
outright so a novel opens on its first real page.

## Traps worth knowing

- **The percent-escape one, which cost a whole book.** Zip entries carry real
  spaces; manifest hrefs carry `%20`. Joining the raw form `KeyError`s on every
  document, and the book comes out with **zero chapters and no error anywhere** —
  it just looks like a book that doesn't work. Every href goes through `_href()`.
  Pinned in `Scripts/codebase_tools/tests/test_book_text.py`.
- **`read_server.py` holds its imports from when it started.** Editing
  `book_text.py` or `reader_collection.py` changes nothing until the `pythonw`
  on port 8289 is killed — silently, because the old code keeps answering. Same
  shape as the mailwatch daemon trap ([[COMPLETION_LOG]]) and the art server's
  ([[ART_SYSTEM]]).
- **Chapter ids must stay stable.** A highlight is a quote anchored to an item
  id, so an id that changed between sessions would silently stop painting every
  mark taken in that chapter. The id is `<book id>#<spine href>` — both fixed
  properties of the file, neither dependent on how many chapters the parser
  chose to keep.
- **Parsing is cached on `(path, mtime, size)`.** Every grab calls
  `State.reload()`, which re-resolves the whole spec; without the cache,
  highlighting one sentence would re-open the zip and re-parse four hundred
  pages.

## Coverage

105 EPUBs in the library parse with zero failures and zero empties. PDF, MOBI
and AZW3 are refused by name (`"MOBI can't be read here — only EPUB"`) rather
than half-read: MOBI/AZW are Amazon's own containers, which the Kindle path
already handles properly, and a PDF has no reflowable text to give a reader
whose whole point is her own measure.

Ambient soundtracks already carried over: a reader open plays the book's music,
*before* raising the tab rather than after, so the music tab never steals the
page it just opened. (`ReadBook` has the same ordering problem and solves it by
re-raising Kindle afterwards; here the fix was free.)

## Possible next

- A `read <book>` spoken word *inside* the reading room (today the reader's own
  spoken resolution covers practices only).
- Reading progress: the reader knows the chapter and page; the completion log
  currently only knows that a session happened ([[COMPLETION_LOG]]).
