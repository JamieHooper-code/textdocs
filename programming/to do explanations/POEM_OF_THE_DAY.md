---
tags: [programming, poetry, new-tab, reader, scraping, design]
created: 2026-10-14
status: built-v1
related: ["[[SPECIAL_TODAY]]", "[[POETRY_SYSTEM]]", "[[NEW_TAB_PAGE]]", "[[QUOTES_SYSTEM]]", "[[READING_ROOM_BOOKS]]", "[[SETTINGS_SYSTEM]]"]
---

# Poem of the Day

Two rows in the new tab's **Special Today** panel ([[SPECIAL_TODAY]]), the same
two all day:

| Row (pill) | What | Opens |
|---|---|---|
| **poem of the day** | a poem that is NOT hers | the reading room, on that poem |
| **one of yours** | one of her own poems ([[POETRY_SYSTEM]]) | the poetry reader (`poems:all`), so **"edit this"** edits it |

Purpose (Jamie, 2026-10-14): reading is hard with her hand disability, so poems
come to her one at a time in the reader; and her own poems get re-read (and
edited) instead of sitting in the Doc.

## Where "not mine" comes from

Three pools, one picked per day by weight (settings, `open settings` → Poem of the day).
**Weights as of 2026-10-14 (Jamie): web 5; books ramp to 3 at 20 poetry books (5
books = 0.75); favourites ramp to 2 at 250.** Simulated 60 days: A Poem A Day 27,
Poetry Daily 24, books 8, poets.org 1. Inside the web the SITE is picked first,
then a poem (only sites with something unseen take part) — by count Poetry Daily
would bury A Poem A Day, her favourite.

| Pool | Source | Default weight |
|---|---|---|
| web | **A Poem A Day** (apoemaday.tumblr.com — her favourite; the whole archive, 1,179 poems) + **poets.org Poem-a-Day** (today's poem) + **Poetry Daily** (poems.com, the whole archive, 3,143 poems — added 2026-10-14) | 3 |
| books (ramping) | her poetry books, **one chapter = one poem**, exactly as the reading room already splits them. 5 books / 509 poems on 2026-10-14 (Andrea Gibson ×3, Audre Lorde ×2). Picks a BOOK first, then a poem, or the Collected Lorde (320) would be two picks in three. | 3 |
| favorites | poems she has favourited (Up) into the quote store | 0 → 2, **ramping**: weight = 2 × favourites / 250 (`favorites_full_at`). 3 favourites ≈ 0.024, 125 = 1, 250+ = 2. Jamie, 2026-10-14: a handful of favourites at full weight would come round every few days. |

When the web pool is chosen, a post from the last 2 days she hasn't seen wins
(`fresh_web_first`) — that one really is "the poem of the day". Nothing repeats
within `repeat_days` (365; her own: `mine_repeat_days` 180); a pool that runs dry
falls back to the poem shown longest ago rather than going blank.

Her own: `mine_pool` = all (default) / favourites / rank 7+ / needs editing / in
progress. Higher ranks come up more often (rank 10 ≈ 3.5× an unranked poem;
favourites ×1.5).

**A poetry book** = the book (or its author) declares `poem` as its quote type
(Reading Miller "Captures as"), or is tagged `poetry`. Front matter (title,
dedication, "A Note on Poetry E-Books", contents…), anything in a "Prose" section,
and chapters under 80 / over 9000 chars are skipped. To add a book: mark it as
poetry — nothing else.

## Kept, not hidden (Jamie, 2026-10-14)

She wanted every web poem kept and every poem-of-the-day remembered, and both
visible rather than buried:

- **Every scraped poem stays on disk for good**, favourited or not:
  `E:\Media\Poetryeeds\poem_feed.json` (4,323 on 2026-10-14). Nothing removes
  from it; a favourite only ADDS a quote-store copy.
- **Every poem shown, and every one passed over with ↻, is logged for good**:
  `history.jsonl` (append-only, one line per event) + per day in
  `poem_of_day.json` (never trimmed — the first build pruned at 60 days; that is
  gone). `py poem_of_day.py history` prints it.
- **Readable in the reading room:** `OpenPoemOfTheDayHistory()` = every poem of the
  day, newest first, hers and the strangers', passed-over ones marked
  (`potd:history`, address `/poems-of-the-day/`); `OpenWebPoems([source])` = the
  whole web archive newest first (`webpoem:all` / `tumblr` / `poetsorg` /
  `poetrydaily` / `fav`). Voice phrases for these: offered, not chosen yet.

## Poetry Daily: the poet comes from the page

The WordPress API (`/wp-json/wp/v2/daily_poem`) gives the text — each line an
`excerpt_line` span, an empty span = stanza break — but NOT the poet, who is only
on the poem's own page (`daily_poem_author`, first and last name in separate
spans). So a new poem's poet is read at sync time; the back catalogue was filled
by a one-off run at one page a second (`sync --source poetrydaily --authors
4000`, ~52 min). Each sync after that reads at most `PD_AUTHOR_BATCH` (40). **A
poem whose poet is not known yet is never offered.** The credit line ("from
<BOOK>: by … Published by …") is kept as `credit`.

## The panel and voice

- Special Today is drawn in **sections** (birthdays · zines · poems · journal
  question · music) with a thin grey rule between sections, and the Journal
  Prompt panel moved into it — see [[NEW_TAB_PAGE]] § Special Today.
- Each poem row (and the journal question) has an inline **↻**: a different one
  today (`reroll` write → `poem_of_day.py reroll other|mine`).
- Voice (generic store): **"daily poem"** → `ReadDailyPoem`, **"daily poem
  mine"** → `ReadDailyPoemMine` (no-parameter wrappers so the store doesn't add a
  spoken slot). Voice-browser category: reading.

## How it works

| Piece | Where |
|---|---|
| Engine + CLI | `Scripts\poems\poem_of_day.py` (`sync [--full] [--source] [--authors N]` / `today` / `reroll` / `open-target` / `show` / `set-quote` / `status` / `history`) |
| Feed store | `E:\Media\Poetry\feeds\poem_feed.json` (outside the media repo's git allowlist — strangers' poems are not pushed to GitHub) |
| Day picks + shown history | `E:\Media\Poetry\feeds\poem_of_day.json`; sync log `sync.log` beside it |
| Reader | `reader_collection.py` `webpoem:` spec (`all` / `tumblr` / `poetsorg` / `fav`), type `webpoem`, address `/web-poems/…`; `read_server.py` `favorite_webpoem` |
| AHK | `ReadPoemOfTheDay(which)` / `RerollPoemOfTheDay(which)` / `ReadDailyPoem` / `ReadDailyPoemMine` / `OpenPoemOfTheDayHistory` / `OpenWebPoems(source)` in `Helpers\ReaderFunctions.ahk` |
| Panel | `newtab_server.py` `rows_from_poem_of_day`, score 1.9 / 1.85 (under the zines, above music) |
| Settings | `INIDATA\Settings\schema\poem_of_day.json` |
| Tests | `Scripts\codebase_tools\tests\test_poem_of_day.py` |

- **The feed is not the quote store.** 1,000+ strangers' poems in quote.json would
  flood "All poems", the display pools and the auto-tagger. Same rule as book
  chapters: a web poem becomes a quote-store record **when she presses Up** on it
  (saved whole, `source.kind = web`, poet as creator, post URL as ref), and the feed
  remembers its `quote_id`. Down to zero removes it again. From then on it is in
  the favorites pool.
- **Book poems open as a VISIT** — `read:<book id>;visit`. The reader skips saving a
  bookmark for a visit spec (as for a shuffled one), so the poem of the day never
  moves her place in a book she is reading. Left/Right still walks the book.
- `today` is on the new tab's path, so it never touches the network: when the feed
  is >12 h old it starts a windowless detached `sync` and answers from the cache.
  First call of a day ~1.4 s (reads the books once), cached ~0.2 s.
- A web poem opened on its own lands inside `webpoem:all`, newest first.
- **Credited, always (Jamie, 2026-10-14):** a stranger's poem shows its **byline
  under the title** ("by Rumi, translated by Coleman Barks") and a **credit under
  the poem**: the poet, the book it first appeared in (Poetry Daily's credit line,
  title de-capitalised), and "via <site> (<host>), <date>". Item fields `byline` /
  `credit` (`webpoem_byline` / `webpoem_credit` in reader_collection.py), drawn by
  `bylineHtml` / `creditHtml` in read_server.py — divs without `data-b`, so never
  numbered or grabbed. A book poem opened as a visit gets "by <poet>" + "from
  <book>"; on a book's own page the byline goes after the chapter's first heading
  and is pinned to the reader's size (the publisher's heading styles would make
  it huge). A favourite in Poems of the Day gets the same pair. Her own poems:
  none — she wrote them.

## Sources checked 2026-10-14

| Site | Result |
|---|---|
| apoemaday.tumblr.com | ✅ Tumblr v1 read API (`/api/read/json`), full archive + text, no key |
| poets.org Poem-a-Day | ✅ listing → poem page parse (`field--body`, `data-byline-author-name`); today only, so the archive grows a poem a day |
| poems.com (Poetry Daily) | ✅ WordPress REST `daily_poem` (whole archive) + the poem page for the poet |
| poetryfoundation.org | ❌ Cloudflare bot challenge — not bypassed, on purpose |

## Open threads

- **Learn the source weights from her favourites** (Jamie, 2026-10-14, "at some
  point"): per source, the share of poems shown that she favourited could raise
  or lower that source's weight. Not now — she is happy with the fixed weights.
  The data is already there: history.jsonl (shown) + feed `quote_id` (favourited).

- Voice phrases for the history and the archive; Stream Deck buttons.
- A book's poem inside Poems of the Day can't be favourited there (it is a
  chapter, not a record) — "open it in its book to favourite it".
- Refreshing a visited book page reloads it as the plain book (the address bar has
  no `;visit`), so page turns after a refresh DO move the bookmark.
- Breaking down more books: any EPUB marked poetry joins automatically; PDFs/MOBIs
  can't be split by the reader.
