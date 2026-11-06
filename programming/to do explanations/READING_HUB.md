---
tags: [programming, reading, miller, local-viewer, books, feed-kit]
created: 2026-11-03
updated: 2026-11-03
related: ["[[POEM_PRINT_AND_DESIGN]]", "[[READING_ROOM_BOOKS]]", "[[UNIFIED_MILLERS]]", "[[NEW_TAB_PAGE]]", "[[COMPLETION_LOG]]"]
---

# Reading Hub: one spec, a Miller and a page

The Reading hub (Current reads, Genres, Recently added, Recommended by, To read,
Finished, Add, More, All books) is the **third system on the shared feed kit**,
after Poems and Stories (kit design: [[POEM_PRINT_AND_DESIGN]] § "The shared kit").
It's defined once and drawn twice:

| | |
|---|---|
| **spec** | `Scripts/reading/reading_library.py`: every shelf, a book's panel, every pick-list |
| **writer** | `Scripts/reading/reading.py`: positional verbs, each one a clog.py command |
| **Miller** | "open read" → `OpenReading` → `Scripts/ReadingViewer.ahk` (MillerFeed) |
| **page** | `http://127.0.0.1:8302/reading/` → `OpenReadingPage` (feed_server.py) |
| **actions** | `Helpers/ReadingFunctions.ahk`: string-only AHK wrappers, run via MAINFUN |
| **setting** | `reading.open_with` (miller · page): what "open read" opens. Default: **the page**. "open read miller" always opens the Miller |

Jamie's ask (2026-11-03): *"they both shell out to the same Python … to where
neither will end up being outdated"*, with the page eventually able to replace
the Miller.

## History

The same morning, this was first built as a separate hub: `Scripts/reader/reading_hub.py`,
a hand-written `/library` page on the reading room server, and its own Miller
renderer. Meanwhile the Poetry Studio chat built the generic kit. That made the
hub a parallel copy of the kit, so it was ported onto the kit the same day and
the copy was deleted. Nothing outside Reading depended on it.

## How Reading uses the kit

- **Books are items.** A book's panel has the same actions wherever it was
  reached from. Its status decides the toggles: "Take out of / Make it a current
  read", "Move to To read", "Remove from To read".
- **Every sub-page is a pick-list:** tags, recommended by, genre, shelf,
  meditation style, grabs (translate), a person's groups, and merge. Each choice
  carries its own write (`tag-on` / `tag-off`, `credit` / `uncredit`…). Download
  sessions put batch pick-lists (`grouptags`, `grouprec`) in the shelf itself.
- **Text fields:** spoken name, a new tag, someone new. On the page they're text
  boxes. In the Miller, `_ReadingAskText` (the feed's `title` hook) asks the
  spec's own question (`prompt` on the action).
- **Dividers** group Current reads by shelf. This row type was added to the kit
  for Reading: unnumbered in the Miller, skipped by the page's arrow keys.
- **QUICK row actions** (the N.M fast path) are the same for every book row:
  1 Read it · 2 Log a session · 3 Mark finished · 4 Open in Kindle · 5 Open in the reader.
- **Reads** run clog in-process, under one lock. **Writes** go through
  `reading.py`, which only calls clog (and `quotes.py vocab-add` for a new tag).
- **The page redraws** when any `library.json`, the reading log, or
  `INIDATA/reading_library.stamp` (touched by every write) changes.

## Changing it

1. Edit `reading_library.py`. A new AHK action goes in `ReadingFunctions.ahk`
   (string args only) and in the spec's `MAINFUN_FNS`. A new write goes in
   `reading.py` and in `SET_VERBS`.
2. Restart the page server: `py Scripts/viewers/feed_server.py restart`. The Miller
   picks the change up on its next open.

Tests: `Scripts/codebase_tools/tests/test_reading_library.py`, plus the GUI test
`reading_menu`.

## Gaps compared with the old Miller

- **Row actions** used to differ per list (To read had "Start reading" and
  "Remove"). The kit has one set for every book row; those actions are now on the
  book's panel.
- **Ctrl+R / Del layouts on Genres** (GuiLayout) are gone: the kit's Miller
  doesn't carry `layout_id`.
- **Failed downloads** is a row that opens the download manager, rather than the
  list mounted inline.
- **The Say column** is folded into the detail ("say “earthsea”").
- **Speed:** each Miller level is a fresh Python, about 0.3 s, the same as Poems
  and Stories.
