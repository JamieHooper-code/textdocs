---
tags: [programming, poetry, writing, practice, object-writing, ahk, caster, tagging, design]
created: 2026-09-21
status: built
related: ["[[POETRY_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[JOURNAL_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[MEDIA_SYSTEM]]", "[[PEOPLE_SYSTEM]]", "[[SETTINGS_SYSTEM]]"]
---

# Practices System

Jamie's writing **practices** — object writing, haiku, the metaphor-hunt
collisions — as a first-class store, plus the pool of things to write about that
feeds them. The fourth parallel text system after [[QUOTES_SYSTEM]],
[[JOURNAL_SYSTEM]] and [[POETRY_SYSTEM]].

**Status (2026-09-21): BUILT and imported.** 50 sessions from the practice Doc
(2023-04-28 → 2026-03-03) are in `E:\Media\catalog\practice.json`, 4,987 prompts
from five real sources are in `E:\Media\catalog\prompt.json`, and three of her
sessions are linked to the poems that came out of them. Voice, picker, writing
box, Miller and the reading room are all live. Not yet: the `person`/`place`/`time`
axes are thin (see "Known limits").

---

## The shape of it

```
poetry object          ← voice
   ↓
StartPractice          ← launcher only (Helpers/PracticeFunctions.ahk)
   ↓
Scripts/PracticeHost.ahk   ← OWN PROCESS. Not optional. See "The ten-minute problem".
   ↓
prompt picker (9 + refresh on 11)  →  writing box + countdown  →  practices.py add
                ↑                                                        ↓
        prompt.json (4,987)                                     practice.json (50)
```

| Piece | Where |
|---|---|
| Practice store + CLI | `Scripts\practices\practices.py` → `E:\Media\catalog\practice.json` |
| Prompt pool + CLI + seeders | `Scripts\practices\prompts.py` → `E:\Media\catalog\prompt.json` |
| Type registry | `INIDATA\practice_types.json` |
| Importer | `Scripts\practices\practice_import.py` |
| AHK bridge (picker, box, countdown) | `Helpers\PracticeFunctions.ahk` |
| Session host (own process) | `Scripts\PracticeHost.ahk` |
| Miller | `Helpers\PracticesMenu.ahk` + `Scripts\PracticesViewer.ahk` |
| Reader collection | `Scripts\reader\reader_collection.py` (`resolve_practices`) |
| Reader openers + "show this" | `Helpers\ReaderFunctions.ahk` |
| Voice | `rules\practice_commands.py` |
| Tests | `Scripts\codebase_tools\tests\test_practices_store.py` (27), `test_practice_import.py` (20), `Helpers\Tests\test_practice_picker_rows.ahk` (7) |

---

## Why a sibling store and not a third `kind` in `poem.json`

A practice's identity is **(date, prompt, practice type, duration)**. None of
those exist on a poem. Going the other way, every field that makes a poem a poem
— the 1–10 rank, NC/C / NE / F status, `special`, its position in the Doc,
labelled versions, poem-poster export — is meaningless on a ten-minute
sense-dump. Ranking an object-writing session 1–10 is not a feature, it is a
category error, and 50 practices in a 972-record poem store would answer every
existing poems filter with noise.

The cost of the split is that a practice and a poem cannot be the same record.
That cost is paid by `grew_into`, below, because the link is the thing that
actually matters.

**The argument that nearly won the other way:** scraps live in `poem.json`
precisely so that flipping scrap↔poem is a field change and never a migration,
and `grow` / `grew_from` / `grew_into` was already built and tested. If practices
turn out to get reclassified as scraps routinely, revisit this.

---

## A practice is raw material, and sometimes a poem comes out of it

Not hypothetical, and not a future feature: her Doc already contains it three
times, and all three poems were **already in `poem.json`** from the poetry
import months earlier.

| Session | Poem |
|---|---|
| 2025-05-08 `trip with Ty to Missouri` | `poem:tie` |
| 2025-05-02 `Sunrise` | `poem:windless` |
| 2023-07-18 `Overpass` | `poem:overpass` |

So the importer **matches and links** rather than re-importing. Getting that
wrong would have put a second copy of "Tie" in a second system with no way to
tell which one was hers. Matching is by *containment* of the smaller word set
(≥ 0.75), not Jaccard — the Doc copy and the stored poem differ by a title line
or an edit made in one place and not the other, so a symmetric measure
understates a real match. All three matched at 100%.

A verse block matching nothing is **reported, never created** (`report`). One
did: the 2025-09-09 entry, best match 50%. That is hers to decide.

---

## The prompt is a record, not a string

The obvious shape is `prompt: "iceberg"` as a field on the practice. Every
question she actually asks then becomes a scan of every practice ever written:
*which objects have I used? don't offer me one I did last month? give me ten I
have NOT done.* And worst, a pool of **candidates she has not used yet cannot
exist at all**, because a string on a practice only comes into being once the
practice is written — the 742 prompts scraped from the community archive have to
live somewhere before they are used.

So a prompt is its own record, `used` is a field, and the picker is a query.
Her 40 historical prompts were marked used by the import, so the very first
`poetry object` already knew what she had done.

### Where the 4,987 prompts came from — all real, none generated

| Source | N | What it is |
|---|---|---|
| `jamie` | 27 (+25 from her own sessions) | Her own candidate list, from the head of the practice Doc |
| `pattison` | 39 | His 14-day challenge, with his What/Who/When/Where axes and durations |
| `dow` | 722 | **`dailyobjectwriting.wordpress.com`** — 781 posts, 2008-05-17 → 2010-08-25, free WordPress.com REST API, no key. A real object-writing community's daily prompts. Titles only: each post body is a stranger's writing. |
| `objectwriting` | 30 | `objectwriting.com`'s word-of-the-day. **A live feed, not a known cycle** — see below. Re-scraped by `seed-objectwriting`, which accumulates. |
| `concreteness` | 4,414 | Brysbaert et al. (2014) concreteness norms, 40k lemmas. Filtered to `Dom_Pos=Noun`, `Conc.M ≥ 4.5`, `Percent_known ≥ 0.95`, `SUBTLEX ≥ 25` → 2,320 objects; plus adjectives and verbs at `Conc.M ≥ 3.0` for the collisions. |

**There is no true "object of the day API."** The archive scrape plus a measured
concreteness backfill is the honest answer, and the backfill exists so the pool
can never run dry — it is the last tier the picker shows.

### objectwriting.com: a claim I got wrong

The first reading of that page was *"30 words, `dayDiff % 30`, so it repeats
monthly and there is no archive."* Jamie pushed back — *"are we certain, or did
they just upload a new one each month? You hit the same API, but in a different
month, and they will be different"* — and she is right that the evidence never
supported the claim. The telling detail is the one I had already read and not
weighed: **30 words with `startDate` pinned to the 1st of the current month.**
September has 30 days. A permanent loop has no reason to keep resetting its own
start date; a fresh monthly list does. One read of one month cannot tell the two
apart, and archive.org was offline when I tried to check.

So it is now treated as a **live feed**. `seed-objectwriting` re-parses the page
and stamps each word with the date it is shown on; `upsert` keeps the earliest
`first_seen`, so running it monthly quietly accumulates an archive nobody
publishes. The constants in `prompts.py` are only the 2026-09 snapshot, kept so
`today` works offline and a failed scrape degrades to something rather than
nothing. `--offline` skips the network deliberately.

**The general lesson**: a mechanism read once is a hypothesis, not a finding.
Where the cost of being wrong is silently missing data, accumulate rather than
assume.

### Where a prompt came from has to be legible

The picker printed the bare source keys, which told her nothing — `dow` and
`concreteness` do not say whether a word is one a real person chose to write
about or a row out of a word list, and that difference decides how much it is
worth reaching for. `SOURCE_LABELS` now prints *your own list · Pattison · Daily
Object Writing · objectwriting.com · generic word list*, leading with the BEST
source and appending `+N` when there are others (`jamie/concreteness` read as
one compound thing rather than "her own word, which also happens to be in the
big list"). The backfill says plainly that it is a generic list, because it is
one, and it always sorts last.

### Modifiers are held to a LOWER concreteness bar, deliberately

An adjective scoring 4.5 barely exists. What Pattison means by an "interesting
adjective" is a **sensory** one, and 3.0 is where those start — *muddy, slimy,
nutty, petrified, sappy, weary*. Raise it and the list empties; drop it to 0 and
it fills with *respective*, *electoral*, *specific*, which collide with nothing.
Her own Exercise 5 words (*understated, refried, smoky, hollow, decaffeinated*)
sit in this band.

### THE TRAP: a curated axis must survive the mechanical backfill

The modifier seed classifies by Brysbaert's dominant part of speech. `cold` and
`warm` are **on Jamie's own candidate list** and Brysbaert calls them
adjectives; `whisper`, `yawn`, `giggle`, `orange` and `eucalyptus` are real
prompts from the archive that it calls verbs. Without the guard in `upsert()`,
the backfill moved 23 of them out of `object` and into the modifier pool — so
the picker could **never offer them to write about again**, silently, with no
error and nothing in any log.

Dominant POS is a fact about the language. It is not a ruling on what she is
allowed to write about. Pinned by `test_a_curated_axis_survives_the_mechanical_backfill`.

---

## Refresh is on 11, and there is no 10

Jamie specified this by hand and by number. It is not decoration.

```
 1 .. 9   the prompts
 10       deliberately empty, rendered as —
 11       ⟳ Refresh the list
 12       ⇄ Switch axis
```

`0` is the universal cancel code across every one of her UIs
([[SETTINGS_SYSTEM]] and `docs/gui-conventions.md § Numpad navigation norms`),
so typing `1` `0` `Enter` is **one mistimed keystroke away from cancelling the
picker outright**. `11` is the same key twice and cannot be confused with
anything.

The picker template numbers *visible rows* positionally, 1..N. So refresh only
stays on 11 if the prompt block is always exactly nine tall — and the cases
where it would not be are exactly the cases where it matters most: a thin pool
late in her practice, an axis with only nine prompts total, or a lowered
`practices.picker_count`. In all of those the list is short, refresh would slide
up to 7 or 5, and the muscle memory would be wrong precisely when she is
reaching for "give me different ones". Hence the pad, and hence
`Helpers\Tests\test_practice_picker_rows.ahk`.

`practices.picker_count` therefore controls how many rows get **filled**, never
how tall the block is.

### Refresh means a new seed, not a new sort

Without a fresh seed the Python picker is deterministic and "refresh" would hand
back the same nine words forever. The picker is *weighted*, not sorted: a strict
tier sort would show her the same 27 of her own words for three refreshes
running, which is the opposite of "so I can always find ones that feel relevant
to me at that time".

---

## The ten-minute problem — why a session runs in its own process

`MAINFUNCTIONS.ahk` is `#SingleInstance Force`. Every `MAINFUN.bat` call starts a
fresh process, so Force means **any new command terminates whatever command is
already running.**

That is invisible for a command that finishes in milliseconds. A practice is the
opposite case: she sits and writes for ten minutes with the box open. In that
window she *will* fire another voice command, the mail watcher *will* raise a
toast, a finished download *will* announce itself with `MAINFUN.bat ShowTooltip`.
Any one of them would kill the writing box and take ten minutes of writing with
it — silently, with no log line, because the process dies before it can write
one.

`StartPractice` is therefore a **launcher only**: it spawns
`Scripts\PracticeHost.ahk` and returns. Same trap and same fix as
`Scripts\AnnaGrabHost.ahk` and `Scripts\KindleGrabHost.ahk`.

`#SingleInstance Off` in the host is deliberate: two sessions at once are
harmless, and Force would mean starting a second one silently destroys the
first one's writing.

The countdown is a bare `Gui` + `SetTimer`, which is normally wrong
(`references/process-traps.md`) and is fine *only* here, because the host
process stays alive for the whole session. It must not be lifted into a Helpers
function called from the dispatcher.

---

## The writing box is the poetry writing box

Not a variant of it. Jamie flagged this directly: a practice is writing, so it
gets the surface she already knows — `line_numbers` **always** (numbered lines
are how you see at a glance that an object-writing session ran to 20 images),
the same 700×480 field, and the same `key: value` header lines, which is the
journal's inline-markup convention shared by `AddPoem`, `EditPoem` and the quote
review editor.

```
tags: grief, cold
senses: sight, touch, organic

a blue shelf of it above the waterline
the crack that carries for miles
```

`senses:` appears only for types that declare `senses: true` — haiku does not.
Keep `_PracticeWrite` in step with `AddPoem` in `PoemsFunctions.ahk`.

---

## Reading them — the Doc, done properly

Her practice Google Doc was one long scroll of dated sessions and she read it by
scrolling. `read practice` is that, better.

**The generic "readerify" layer already existed**, which was the right guess:
the reading room resolves a collection through `resolve_<type>(spec)` returning
`{title, subtitle, items, flow}`, and every type — books, quotes, poems, smut —
is one such resolver plus a registration. Practices are `resolve_practices` in
`Scripts\reader\reader_collection.py`. Nothing generic had to be built.

**`continuous`, not `paged`, and that is the whole feature.** `PAGED_TYPES`
gives each item a fresh page, which is right for an exercise or a chapter. A
practice is short — median 159 words, and the collisions are a couple of lines —
so a page break per session would be a slideshow of mostly-blank pages. Left out
of `PAGED_TYPES`, short ones share a page exactly as she asked. Measured on the
real store: **50 practices across 44 spreads, with three sharing a single page in
several places.**

Spec grammar is deliberately the same `;`-joined one `poems:` uses, so there is
one to learn: `practices:type=haiku`, `practices:year=2023`, `practices:tag=grief`,
`practices:prompt=candle`, `practices:grew`, `practices:order=random`. The URL is
the same thing written down — `/practices`, `/practices/haiku`, `/practices/2023`,
`/practices/grew`, `/practices/candle` — and unlike `/read` and `/smut`, a bare
`/practices` names something real, because "all of them" is the common case.

---

## "Edit this" on a page holding several things

In a paged flow "this" is unambiguous. In a continuous one a spread can carry
three practices, and the server would hand back whichever `currentIndex()`
settled on — a guess, and a wrong one often enough to be worse than asking.

So `ReaderShowThis` checks `visible_ids`. One item: edit it, no number to press.
Several: numbers go up on the page, she presses one. The same move the Kindle
grab overlay already makes for sentences, one level up — which is where the idea
came from.

```
"show this"  →  /api/numbers?mode=item  →  badges 1..N on the headings
                                        →  she presses 2
                                        →  /api/edititem?id=…
                                        →  MAINFUN.bat ReaderEditItem <id>
                                        →  the house writing box
```

`ReaderEditItem` dispatches on the **id prefix**, not the open type, because a
collection can mix stores (`allpoems:` carries both her poems and quote-store
poems), so the type does not reliably say which editor an item wants. Escape
takes the numbers down, server-side as well as on screen.

### Two traps in `visibleIndices()`, both already solved elsewhere in that file

Worth knowing because both produce a *plausible* wrong answer rather than an
error, and I hit both:

1. **`getBoundingClientRect()` on a multi-column flow spans every fragment.** A
   practice running over five columns has one bounding rect 3000px wide that
   overlaps the page on *every* spread, so the first version reported item 0 as
   visible everywhere. `getClientRects()` returns the per-column pieces; only
   those locate it. `drawUnits` already carried this lesson.
2. **The page moves by a transitioned transform on `#flow`**, so immediately
   after `paint()` the inline style says `translateX(-5024px)` while the
   computed style is still identity and the rects have not moved — measuring
   against the viewport reads the *previous* page. Subtracting `#flow`'s own
   rect cancels the transform mid-transition, which is exactly why `drawUnits`
   positions badges that way and why `spreadOf` takes a `base`.

Inferring from `starts` instead does not work either: two practices sharing a
page have the **same** start, so `starts[i] <= spread && nextStart(i) > spread`
drops the first one — precisely the case the feature exists for — and loosening
it to `>=` over-includes an item that ended on the previous page. `starts` cannot
tell "shares page 6" from "finished on 5". The rects can.

**There is no JS test harness for the reader page**, so these are pinned by this
document and by the Python-side collection tests, not by a unit test. Verified
by hand against the real store: three badges on a three-practice spread, pressing
`2` opened the second one.

---

## Practice types are a registry, not code

`INIDATA\practice_types.json`, one row per practice — same rule as
`INIDATA/Contexts`, `lockout_profiles.json`, `screenshot_buckets.json` and
`meditation_styles.json`. Adding a practice is one JSON object plus an alias;
nothing in Python or AHK changes. Each type declares what kind of prompt it
needs (`single` / `pair` / `none`), which is why the collision exercises get a
pair picker for free.

All eight types were found in her own Doc. None are invented:

| Type | Where it came from |
|---|---|
| `object_writing` | The bulk of the Doc, July 2023 → Sept 2025 |
| `haiku` | April 28 2023 — three 3-line stanzas |
| `adjective_noun` | July 13 2023, "Exercise 5" — *understated railroad*, *refried conversation* |
| `noun_verb` | November 13 2023 — *Woodstove vomits*, *Surfboard cancels* |
| `noun_noun` | November 15 2023 — *Summer mattress*, *Ocean paintbrush* (Exercise 8 step five) |
| `expressed_identity` | July 6 2023 — *water = life*, *books = food* |
| `metaphor_hunt` | Exercise 8, worked across Jan 31 / Feb 2 / Feb 12 / Feb 14 2024 |
| `word_cloud` | March 3 2026 — the Chrysalis entry |

Spoken aliases must be unique across types: a duplicate Choice key is a Dragon
`GrammarError` that kills the whole rule silently. Pinned by
`test_spoken_aliases_are_unique_across_types`.

---

## What the real Doc forced on the importer

Each of these cost a parse pass and produced a wrong import before it existed.
All are pinned in `test_practice_import.py`.

1. **The unit boundary is a bold DATE, not a heading.** The whole file has
   exactly two `<h2>`s (both in the March 3 2026 entry). Everything else — every
   date, every object — is a plain `<p>` carrying a bold span. Bold is measured
   as a *ratio*; `≥ 0.99` means the whole line, so a date inside a sentence is
   not a header. `parse_paragraphs` is imported from `poem_import.py` rather than
   re-implemented, because both Docs come out of the same exporter.

2. **One era writes no year at all.** June–August 2023 is `July 23` / `Candle`.
   Doc order is **newest-first**, so a yearless date takes the year of the
   nearest dated session and is then stepped back until it is genuinely older
   than its predecessor. A date that cannot be made to fit is reported, never
   guessed at. Abbreviated months (`Jun 22`) are real here — a full-month-name
   regex silently dropped four sessions.

3. **Pattison's own exercise text sits under the last session.** 14 blocks of
   his book, undated, so they were absorbed into her three haiku and made a
   30-word entry into a 568-word "object writing". Cut by a bare `Exercise N`
   head **plus** an instruction-shaped body — the head alone is not enough,
   because `Exercise 5` and `Exercise 9:` are real session labels elsewhere.
   Filed as the store's `guide`, along with the Doc's header.

4. **Line length cannot separate her object writing from a poem.** Her 2025-era
   object writing is written one image per line, so it is exactly as short-lined
   as verse. Testing shape alone cut at block 0 and reported whole sessions as
   *0 words + verse*, throwing the writing away. What actually separates them is
   **her stanza mark**: a lone `-` line, which she uses inside poems and never
   inside object writing.

5. **Haiku is three-line stanzas, not short lines** — same reason. `Homelessness`
   and the Missouri session both read as haiku on length alone.

6. **A `=` line only types a session as `expressed_identity` if the session OPENS
   with one.** July 6 2023 is object writing on `Pepper` that happens to also
   carry a later `water = life` run; scanning the first six blocks for an `=`
   typed the whole day as identity and threw the real prompt away.

7. **Section labels are not prompts.** `object writing:`, `Exercise 9:`,
   `Word Cloud:` are stripped; `trip with Ty to Missouri:` (five words, colon) is
   a prompt and a three-word cap missed it.

8. **Her poem header codes are exempt from the prompt word-cap.**
   `(NE) (?) The Light Always Leads Me Astray` is eight words, so the cap sent it
   into the body where it read as the opening line of the object writing.

**The coverage guarantee**: every non-empty block lands in exactly one session
(0 orphans, 0 doubles against the real file). That is the check that catches a
parser change quietly eating a session.

---

## The guarantees (pinned by `test_practices_store.py`)

1. **Nothing she wrote can be lost.** Any change to a record's text first pushes
   the previous version onto `revisions`. No command deletes: `trash` sets
   `state`, `restore` undoes it.
2. **Written is not edited.** `written` is the date of the session and no edit
   touches it.
3. **Tag provenance.** Tags are `{tag, src}` rows with src `doc | manual | auto`;
   a writer rewrites only its own src, so a re-import cannot strip a tag she
   added.
4. **The prompt is a link.** `prompt_key` points into `prompts.py`; `prompt` is
   display text kept alongside, the same way the media catalog keeps
   `recommended_by` beside `recommended_by_ids`.

Ids are date-first (`practice:20230723_candle`) so they sort chronologically and
are readable in a log line.

---

## Voice

The `poetry` namespace is deliberate and **not verb-first**. House style is
verb-first and `voice_index check` duly lints these as WARN; Jamie overrode that
— practices are part of the poetry system and she wants them said that way.
Domain-first is the documented exception at real scale, and eight practice types
behind one Choice is that case. No phrase collides (`poetry` was a free
namespace, matching only Choice *values* inside `open <directory>` and friends).

| Phrase | Does |
|---|---|
| `poetry <practice_type>` | Picker → writing box. Choice over the registry's aliases. |
| `poetry practices` | The Miller — browse everything, curate the pool |
| `poetry streak` | How many consecutive days |

`poetry <practice_type>` is a **Choice, not a Dictation**: a wildcard behind
`poetry` would claim the whole namespace, and the poetry system will want more
of it later.

---

## Settings

Per the standing rule, every tunable is declared rather than left a constant:
`practices.picker_count`, `practices.reoffer_after_days`,
`practices.default_axis`, `practices.pool_browse_limit`. `reoffer_after_days`
defaults to 0 = never re-offer something she has written about; set it to e.g.
365 to let old objects come back around.

`PRACTICE_PICKER_ROWS = 9` is **not** a setting and must not become one — see
"Refresh is on 11".

---

## Known limits

- **The `person` / `place` / `time` axes are thin** (9 / 14 / 10), because the
  only sources that classify by axis are Pattison's 39 and her own 27. The
  concreteness backfill has no axis information and no WordNet is installed to
  derive one, so machine-classifying the other 4,900 is not currently possible.
  The picker returns fewer rows for those axes and says so. Adding more is
  `prompts.py add <word> --axis place`.
- **Verb forms in the collisions are inflected as the corpus had them**
  (*bumblebee battled*, *forest beating*) rather than normalised to her
  *Woodstove vomits* third-person present. Fixing that needs a lemmatiser.
- **One of Pattison's 14 days is missing** (39 of 42 prompts). Two published
  transcriptions overlap on a day they number differently, so one triple never
  surfaced cleanly. Deliberately not filled in with invented prompts — a
  fabricated prompt attributed to Pattison would be indistinguishable from a
  real one forever after.
- **The Miller's practice list loads full text for every row** to build the
  3-line preview. Fine at 50 records; if it grows, switch the list feed to
  `--brief` and fetch text per-row.

---

## Traps worth remembering

- **AHK v2 does not parse `\uXXXX` in a string literal.** Written that way, the
  spacer and refresh arrows rendered as the literal text `\u2014` and `\u21bb`.
  Use `Chr(0x2014)`.
- **`numpad_mode` prepends its own `#` column.** Declaring another one in
  `colHeaders` renders the row number twice.
- **`numpad_mode: true` is load-bearing for `numpad_dispatch`**, not just
  cosmetic: `_refreshValid` only suppresses text-filtering for a digit-only
  filter when `numpad_mode` is set, so without it typing `11` filters the
  catalog instead of picking row 11.
- **`_Pm` is PoemsMenu's prefix.** The practices Miller uses `_Prc`.
- **PrintWindow captures a Miller/picker mid-rebuild.** Two screenshots showed
  rows 9–12 missing after a refresh; the rows were there. Verify behaviour from
  the log, not from one screenshot.
- **AHK hotkeys ignore synthetic input from another AHK's `Send`** unless
  `SendLevel` is raised. Ctrl+Enter looked broken under test and was not — check
  `SendLevel 1` before believing a hotkey is dead.
