---
tags: [programming, books, kindle, automation, ahk, caster, listnav, uia, design-doc, tagging, media]
---

# Anna's Archive pipeline — one `jump N`, book on the Kindle

**Authoritative doc.** Read before touching `Helpers/AnnasArchiveFunctions.ahk`,
`Scripts/anna_download_watch.py`, `Scripts/anna_metadata.py`,
`INIDATA/Contexts/anna_s_archive_results.json`, or the tag-provenance rules below.

Related: [[MEDIA_SYSTEM]] · [[COMPLETION_LOG]] · [[QUOTES_SYSTEM]] ·
`INIDATA/Contexts/README.md` (the ListNav profile system) ·
`skills/ahk-functions/references/uia-clicking.md`

---

## What it does

Say a search (`book <title>` → the existing `searches.json` entry), then **`jump N`**
on the results page. That single command runs the whole chain:

```
jump N ──▶ click result N in a NEW TAB          (search page survives)
       ──▶ wait for the detail page              (slow-server link = ready signal)
       ──▶ "Read more…" → capture ~28 metadata fields → sidecar (md5-keyed)
       ──▶ arm the detached watcher              (pythonw, outlives this process)
       ──▶ Slow Partner Server #1
       ──▶ wait out the site's countdown         (~17s, NOT circumvented)
       ──▶ "Download now" → confirm Save As
       ──▶ MAINFUN process EXITS (~40s total)
              ⋮ watcher runs on alone
       ──▶ file lands in E:\Downloads            (waits up to 5 min)
       ──▶ kindle_import.py   → library + auto genre tags
       ──▶ anna_metadata.py   → merge the captured record + promote tags
       ──▶ send_books_to_kindle.py → Kindle
       ──▶ tooltips at each stage
```

Verified end to end: ~16s from `jump 1` to "imported and sent" on a fast server.

## The pieces

| Layer | File | Owns |
|---|---|---|
| Entry / row click | `INIDATA/Contexts/anna_s_archive_results.json` | ListNav profile: which links are rows, new-tab click, the post-click hook. **No code.** |
| Click-through | `Helpers/AnnasArchiveFunctions.ahk` | `AnnaGrabBook` (the whole flow), `AnnaDownloadBook` (server → countdown → download → Save As), `_AA_CaptureMetadata` (the sidecar) |
| Detached tail | `Scripts/anna_download_watch.py` | wait-for-file → import → metadata → send, + on-screen tooltips |
| Metadata | `Scripts/anna_metadata.py` | parse the captured record, merge into `library.json`, promote tags, `--survey` |
| Existing, reused | `kindle_import.py` · `send_books_to_kindle.py` · `BookManagerMenu.ahk` | ingest, delivery, review queue — **the back half already existed** |

## Design decisions (the why)

**The profile carries the flow, not the engine.** `jump N` is the generic ListNav
verb shared with Instagram/Teams/Gmail/Spotify. Nothing site-specific went into
`ListNavFunctions.ahk`: the profile names `post_click_function: "AnnaGrabBook"`
and the existing hook invokes it. Blast radius zero — a profile without the field
no-ops. (The hook previously fired only on the grid path; it now fires on the
enumeration path too, which is a fix, not a special case.)

**Height alone identifies a row.** Every result is one tall link (809×111 compact,
96×165 covers) and every other link on the page is ≤37px. So
`row_selector: {Type: Hyperlink, min_height: 100}` — no width filter, which is why
the SAME profile works on the compact search view AND the cover-grid list view, and
survives window resizing. Don't add a width filter.

**Capture BEFORE download.** The metadata panel only exists on the detail page;
clicking a download link navigates away and it's gone forever. So capture runs
first, writes a sidecar, and the watcher joins on it later. **The md5 is the join
key** — it appears both in the panel (`MD5 f9dc…`) and inside AA's filename
(`-- f9dc579c… --`). No guessing.

**The tail is detached.** A 5-minute download must not hold an AHK dispatcher open.
`pythonw` + `Run(..., "Hide")`, mirroring `_BMArmWatcher`. The watcher is armed
BEFORE the download starts: its "new file" cutoff is set when it launches, so
arming afterwards would race a fast download and miss it.

**Readiness is polled, never slept.** The slow-server link's existence IS "the
detail page loaded". The download link's appearance IS "the countdown is served".
No fixed `Sleep` anywhere. The site's ~17s wait is served in full — nothing here
circumvents a rate limit.

**Completion is a real test, not a timer.** A file counts as arrived only once it
carries a book extension AND its size has been stable for 3 consecutive polls
(Chrome's in-flight `.tmp`/`.crdownload` never carry one). Three, not two: a
stalled transfer can look stable for a moment — the same easing-plateau lesson
`_LN_FollowAutoScroll` learned.

## Gotchas that cost real time

**Tag provenance is a CORRECTNESS rule, not bookkeeping.** Every writer owns exactly
one `src` and rewrites only its own:

| `src` | Writer | Rewritten by |
|---|---|---|
| `auto` | `kindle_import` genre lookups | `--retag` regenerates these |
| `annas` | `anna_metadata.py` | nobody else |
| `manual` | Jamie | `book-set-tags` |

These tags were originally written as `auto` — which collided, because
`retag_books` keeps only what it doesn't own and regenerates the rest. **A routine
`--retag` silently deleted every Anna's-Archive tag in the library.** Both filters
now keep everything they don't own (`!= "auto"` / `!= "manual"`).

**`--retag --no-net` strips tags.** `genre_tags` skips the Google Books/OpenLibrary
lookup entirely without the network, so a no-net retag blanks every book whose tags
came from there. Always run `--retag` with the network.

**Key parsing: longest match first.** Panel entries are ONE flat string with no
delimiter (`"Filesize 305904"`), and several keys are prefixes of others:
`Z-Library` vs `Z-Library Source Date`, `Nexus/STC` vs `Nexus/STC Source Updated
Date`, `Libgen.li File` vs `Libgen.li fiction_id`. Splitting on the first space
writes `key="Z-Library", value="Source Date 2023-03-09"` **and reports success**.
Match against `KNOWN_KEYS` longest-first. Values are always LISTS — a record with
three source collections genuinely has three `Filepath`s and three `IPFS CID`s.

**Never invent vocabulary.** `canon_tag()` returns unknown tags *unchanged* (so they
stay addressable), so its output is NOT proof a tag exists — gate on
`get_record()`. Without that, `thrillers` gets auto-created next to `thriller` and
quietly fragments a curated vocabulary. Unmapped categories are preserved as
`annas.unmapped_tags` and surface as **suggested** in the tag picker.

**Unknown keys are preserved, never dropped.** AA pages differ by source collection,
so an unknown key is expected. Everything unparsed lands verbatim in
`annas.unparsed`. `py Scripts/anna_metadata.py --survey` reads across every stored
record and ranks the missing keys by evidence — that's how `KNOWN_KEYS` grows
(from 2 records it already ranked `OCLC` ×61, `Open Library` ×9, `EBSCOhost` ×5).

**AHK→Python handoff must be `UTF-8-RAW`.** A BOM breaks `json.load` on the far side.

**`&` in a URL.** Only the *query* crosses the MAINFUN bridge — the template stays
in `searches.json` on the AHK side. Don't pass a full `&`-laden URL through a
`.bat` argument.

**Source metadata is often junk.** Search result #1 is not the best-catalogued
record. One had the title repeated in the author slot and no year/publisher, which
filed it under a title-named folder. The parser was correct; the record wasn't.
`clean_author` was also fixed here: `"Doyle, Sir Arthur Conan"` fell through both
branches and returned a bare `"Doyle"` (only 1 of 158 author folders had ever hit
this).

## Where the data lives

- Sidecars: `%TEMP%\anna_meta\<md5>.json` (deleted on successful merge)
- Books: `E:\Media\Books\<author>\library.json` → the book's `annas` block
- Downloads: `E:\Downloads` (what `kindle_import.py` scans — Chrome already
  defaults there, so no path plumbing)
- Alias backlog: `INIDATA/tag_alias_backlog.json` (see below)

## Reading the logs

The whole pipeline is one greppable story:

```bash
grep "Anna/" ahk_event.log          # grab · meta · download · saveas · watch
```

## The tagging backlog (`Scripts/tag_backlog.py`)

Tagging splits into two problems that look alike and are not. **Mechanical**
(`thrillers`→`thriller`) a rule derives — already handled by the alias index.
**Semantic** (`political`→`theory`) NO rule derives; it's a fact about Jamie's
taxonomy. Every failure here came from asking the deterministic layer to answer
semantic questions.

So: collect unknown tags, let the local model PROPOSE from the existing vocabulary,
Jamie decides in batches. `scan` / `propose` / `list` / `accept` / `accept-all` /
`promote` / `reject`. Writes go through `clog book-tag-alias` (aliases AND folds
across the library) or `quotes.py vocab-add` — **never hand-edit `media_tags.json`**.

**The guard that makes the LLM safe:** it picks from a CLOSED LIST and any answer
not in it is discarded, not trusted. It cannot hallucinate `subjects` into the
taxonomy because `subjects` isn't on the menu. Task `tag_alias` in
`local_llm/tasks.json`, same path as `quote_tag`/`quote_facet`. On the first real
run it declined 9/10 and every decline was correct — conservative by design
("prefer none over a weak guess"), and a `none` is the signal to `promote` a tag
rather than alias it (that's how `cosmere` (28 books) and `tolkien` (7) became
vocabulary instead of being folded into `fantasy`).

Aliases are permanent, so the backlog SHRINKS as the vocabulary grows —
amortisation, not a treadmill. Trigger a review on **backlog size** (~15 pending),
not book count.

## Known gaps / next

- **Row-action `N.M` → Edit tags** still opens the old two-pane picker; drill-ins
  use the new control. Real fix = `layout: "checklist"` opt on
  `_PersistentLoopPickGui` (see [[COMPLETION_LOG]] polish items).
- **`KNOWN_KEYS` is short.** `--survey` already ranks the candidates: `OCLC`,
  `Open Library`, `EBSCOhost … Subject`, `ISBN-10/13`, `LCC`, `ASIN`. Note
  `Open Library Subject` and `EBSCOhost … Subject` are better tag sources than the
  EPUB subjects the importer currently uses.
- **9 backlog items** pending Jamie's call (`body image`, `chinese`, …).
- **The standalone Book Manager** still exists under More; the Reading node
  duplicates its day job. Retire once the node has proven out.
- The pipeline takes the FIRST slow server; no fallback if it's down.
