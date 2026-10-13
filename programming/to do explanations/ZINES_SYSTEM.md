---
tags: [programming, zines, media, design-doc, scraping, art-viewer, affinity, new-tab]
created: 2026-10-08
updated: 2026-10-08
related: ["[[ART_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[SPECIAL_TODAY]]", "[[DOWNLOAD_MANAGER]]", "[[ANNAS_ARCHIVE_PIPELINE]]"]
---

# Zine library — design doc

Status: **built 2026-10-08** -- index (6 sources), Wikipedia canon, importer with un-imposition, library/taste/Zine of the Day, viewer hookup via the comic path, new-tab row, AHK entry points. **Voice phrases pending Jamie's pick** (functions exist: `ShowZine`, `NextZine`, `GrabZine`, `OpenZineOfTheDay`, `OpenZineByPath`).

What Jamie asked for: radical / queer / anarchist / Indigenous / race / poetry / art / **punk and DIY** zines — the stuff a leftist bookshop or infoshop library carries — read in the art viewer, favourited with affinity like everything else. **Not** a hoard of every zine ever made: prioritise the culturally important, insightful, impactful ones, and let her affinity steer what gets pulled next. Plus a **Zine of the Day** in the new tab's Special Today panel.

## The shape: three layers, same as art

| Layer | Question | Where |
|---|---|---|
| **Index** | What zines exist out there, and how much does the world care about each? | `E:\Media\Zines\_index\zine_index.db` (SQLite) |
| **Library** | What is downloaded, tagged, scored? | `E:\Media\Zines\<Source>\<Zine>\` — page images + `_comic.json` sidecar + `_source\<orig>.pdf` |
| **Viewer** | What is on screen? | `art_server.py` collection mode (:8288) |

### Reuse, not new machinery

- **Pages:** `comics.extract_pdf_pages` — lossless JPEG pull-out, 2x render fallback. A zine folder is laid out exactly like a comic chapter.
- **Viewer:** the art viewer's **collection mode** (`/api/collection?dir=`, `Gallery.load_collection`) already reads an ordered folder of pages, captions "page N of M", and keeps the affinity score **on the collection's `_comic.json`**. So `favorite` / `unfavorite` on a zine page scores the ZINE, through `affinity.py`, with favorite/loved/essential/buried bands for free. That is why the sidecar is called `_comic.json` and not `_zine.json` — the viewer looks for that name.
- **Zines are NOT a comics ROOT.** Adding `E:\Media\Zines` to `comics.ROOTS` would have put them in the comics Miller and the `porn <name>` grammar. Own store, borrowed functions.
- **Downloads** are recorded in `downloads.jsonl` (`source="zines"`) so `open downloads` shows failures.

## Sources (verified 2026-10-08)

| Key | Site | How | Impact signal |
|---|---|---|---|
| `ia` | Internet Archive — **allow-listed** sub-collections of `zines` | `advancedsearch.php` JSON; files via `/metadata/<id>` at download time | `downloads`, `num_favorites` per item |
| `qzap` | Queer Zine Archive Project | CollectiveAccess; object pages by sequential `object_id`; PDF at `/Detail/Object/DownloadRepresentation/representation_id/R/object_id/O/download/1/version/original` | none on-site → canon + cross-source |
| `sprout` | Sprout Distro | `/catalog/pdfs` lists ~416 PDFs; `-SCREEN` / `-IMPOSED` / `-print` filename suffixes | none → curated-distro prior |
| `crimethinc` | CrimethInc | `/zines?page=N` → `/zines/<slug>` → `cdn.crimethinc.com/assets/zines/<slug>/*.pdf` | none → curated prior |
| `indigenous_action` | Indigenous Action "Zineopedia" | `/zines/?query-70-page=N` → post → PDF links | none → curated prior |
| `tal` | The Anarchist Library | `/category/topic/zine` + `/stats/popular`; every text has screen PDF + imposed `.a4.pdf`/`.lt.pdf` | popular-texts rank |

**IA allow-list, not crawl.** The `zines` collection is ~100k items and includes UFO newsletters, Diplomacy games, a white-nationalist magazine (`american_renaissance`) and skinhead zines of mixed politics. Sub-collections are listed explicitly in `zine_sources.IA_COLLECTIONS` with topic tags (punk, anarchist, diy, queer, feminist, antifascist, socialist, graffiti, skate, rave). `zines_inbox` (38k unsorted) is included only through a subject-keyword filter. Deny-list is explicit too, so a mistake is visible in code.

**Language:** English (or unmarked) by default — setting `zines.languages`.

### Found but not automated

POC Zine Project (Issuu — no real download), Barnard / Furman / Cornell (physical collections), Brown Recluse distro, Black Zine Fest (an event). Black zines and poetry/art zines have **no big free archive** — they will come mostly through `grab zine` from itch.io and individual sites.

## Impact — "is this one important?"

No site publishes "culturally important". We combine what exists, each normalised **within its source** (a 90k-download IA item and a QZAP item with no counter must be comparable):

1. **Popularity** — IA `num_favorites` (weighted ×3, deliberate signal) and `downloads` (log-scaled, inflated on bundles); TAL popular rank. Percentile within source.
2. **Canon** — does the zine have a Wikipedia article (categories: Zines, Punk zines, Feminist zines, LGBT zines, Anarchist periodicals, Riot grrrl…), and in how many languages (sitelinks, as `art_canon.py` does for paintings)? Fuzzy title match against the index.
3. **Cross-source presence** — the same title held by two libraries means two sets of librarians picked it.
4. **Curated prior** — a distro's own catalogue (Sprout, CrimethInc, Indigenous Action) is already an editorial selection, so it starts above an unsorted IA inbox item.

`impact = 0.45·popularity + 0.30·canon + 0.10·cross_source + 0.15·prior`, 0..1. Stored on the index row and copied into the sidecar. Weights are named constants at the top of `zine_index.py` with their reasoning.

## Taste — "pull more from what I like"

`zine_library.py taste` aggregates sidecar affinity by **source**, **IA collection** and **tag** using **lift** (like `art_taste.py`), not raw count. `zine_import.py seed` multiplies each candidate's impact by its source/collection/tag lift, so favouriting three Slug & Lettuce issues makes the next seed lean punk. Buried zines push their collection down.

## Seed — the starter download

Jamie: "start by downloading a decent-sized chunk from all of these sources, then I'll look through them randomly." `zine_import.py seed --per-source N` takes the top-N by impact from each source with a **per-collection cap** (breadth is structural, like `art_plan.py`'s per-artist cap), prefers screen PDFs, and drips with polite delays. Resumable: an item already in the library is skipped.

## Imposed (print-and-fold) PDFs

Many distro PDFs are laid out for printing: two pages per landscape sheet in **saddle-stitch order**, which reads scrambled on screen. Rules, in order:
1. Prefer a source's screen version (Sprout `-SCREEN`, TAL plain `.pdf`).
2. Scans (QZAP, most IA) are one page per image — nothing to do.
3. Otherwise, if the sheets are landscape ~2:1 portrait halves AND the URL/filename says imposed/print, split each sheet and re-order by saddle-stitch (sheet side *i*: left = T−i, right = i+1 on even sides, mirrored on odd). Verified with printed page numbers via text extraction when present; when the numbers say sequential, it is a reading spread and stays in order.

## Unknown sites — the "make dump" pattern

`grab zine` on a host with no adapter falls back to "largest PDF on the page", **and records the host** in `INIDATA\zine_sites.json` (`unknown` with hit count, first/last URL, whether the fallback worked). `zine_import.py sites` lists them so a frequently-grabbed site gets promoted to a real adapter.

## Zine of the Day / Zine of the Week (2026-10-09)

- **Day = mostly SHORT.** `length_factor`: up to `zines.daily_target_pages` (15) counts fully; longer fades as (target/pages)^`zines.daily_length_strictness` (3.0) -- a 30-page zine is 1/8 as likely. 1-3 page flyers are held back a little. At the defaults ~71% of daily picks are <=15 pages (the library has 106 at 4-15pp, so ~3 months before the 90-day no-repeat starts forcing longer ones). Soft by design ("not a hard limit").
- **Week = any length, MONDAY to SUNDAY** (ISO week key `2026-W41`), never repeats within 52 weeks, never the same zine as today's daily (and vice versa).
- Both deterministic per key and remembered in `library_state.json` (`zotd` / `zotw`). New tab shows both rows; Miller root has both; AHK `OpenZineOfTheDay` / `OpenZineOfTheWeek`.

## Zine of the Day (original note)

`zine_library.py today --json` — deterministic per date (hash of the date), drawn from the library weighted by impact × taste, skipping buried and recently-shown zines. A row in `newtab_server.fetch_special`; clicking it opens that zine in the viewer.

## Voice (live 2026-10-08, generic store)

`show zine` / `show zine <tag>` -> `ShowZine` · `next zine` -> `NextZine` · `grab zine` -> `GrabZine` · `show daily zine` -> `OpenZineOfTheDay` · `show weekly zine` -> `OpenZineOfTheWeek` · `open zines` -> `OpenZines` (the Miller). Category: reading (`voice_categories.json` helper rule matches `Zine`).

## The Miller -- `open zines` (`Helpers\ZinesMenu.ahk`)

Browse: Zine of the day · Shuffle · Favourites · Recently added · By source · By tag · By language. A zine row shows its cover on the right; Enter reads it; `N.2` favourite · `N.3` bury · `N.4/5/6` mark English / Spanish / bilingual · `N.7` its page · `N.8` its folder.

**Get more** is the whole growth loop (Jamie: no weekly run, but "super easy, through the Miller, as automated as possible"): Get N more like my favourites · Get N more in Spanish · Get more from a library... · Get more of a tag... · Top not yet downloaded (Enter gets one) · Refresh the index · Sites grab zine found. Every row starts a DETACHED `pythonw ... --notify` and reports a tooltip when done. N = setting `zines.more_count`; also `zines.recent_count`, `zines.tag_min_count`, `zines.wanted_count`.

`seed --more N [--tag T] [--lang es] [--sources a,b]` pools sources and takes the best N by impact x taste with per-collection and per-series caps.

## Language -- English + Spanish, always marked

Only English and Spanish are harvested (IA `language` normalised by `lang_from_meta`; another language is skipped, unmarked is kept). Every zine's sidecar carries `language` (`["en"]`, `["es"]`, `["en","es"]`, `[]`) + `language_source`, plus an `english` / `spanish` tag so `show zine spanish` works. Evidence order: the library's metadata -> the PDF text layer -> **Windows OCR** (`winsdk`, pages 2-4; CrimethInc PDFs are outlined fonts with no text) -> the title (never "bilingual" from a title) -> unknown. `zine_import.py languages [--redo]` backfills. Spanish sources: the Biblioteca Anarquista (`tal_es`, es.anarchistlibraries.net -- same amusewiki adapter) + IA `acracia_zine`, `todo_por_hacer`, `rojo_y_negro_zine`, `cnt_zine`.

## Content guard

INFOWARS The Magazine (4 issues) was mis-filed in IA's "Miscellaneous Anarchism" and came in with the starter seed; moved to `_excluded\`. `zine_sources.is_conspiracy` blocks a named outlet in the title or 3+ conspiracy markers together -- NOT a single marker, because antifascist zines write about NWO / QAnon / Nazis and must stay (pinned in `test_zines.py`). Applied at harvest, seed, and the wanted list; `reconcile` marks matching index rows `excluded`.

## Tag hygiene

IA subjects are free text: `clean_tag` folds aliases (anarchism/anarchy -> anarchist, punk rock/punk band -> punk, lgbt* -> queer ...) and drops junk ("box", "send sase", "money order", scraped JS). Applied on READ, so sidecars keep the raw tags.

## Files

| File | Job |
|---|---|
| `Scripts\zines\zine_sources.py` | adapters: harvest (index rows) + resolve (download URLs) per source; host → adapter registry |
| `Scripts\zines\zine_index.py` | SQLite index; `harvest`, `score`, `top`, `stats` |
| `Scripts\zines\zine_canon.py` | Wikipedia zine canon → index boost |
| `Scripts\zines\zine_import.py` | `seed`, `fetch`, `grab <url>`, `sites`; un-imposition |
| `Scripts\zines\zine_library.py` | `iter_zines` (cached), `taste`, `today`, `random`, `list` |
| `Helpers\ZineFunctions.ahk` | `ShowZine [tag]`, `NextZine`, `OpenZineByPath`, `OpenZineOfTheDay`, `GrabZine` (detached `pythonw ... grab --notify`) |
| `Scripts\newtab\newtab_server.py` | `rows_from_zine_of_the_day` in `fetch_special`; `OpenZineByPath` in `ACTIONS` |
| `Scripts\codebase_tools\tests\test_zines.py` | un-imposition order, spread-vs-imposed, file choice, title keys, taste, ZotD stability |

## Traps

- **Test the viewer through `affinity.py`, not two `/api/favorite` calls.** While testing, the viewer's gallery changed between a favourite and its undo (something reloaded "everything"), so the undo landed on a random ARTWORK. Fixed by hand (`affinity.py bump <json> --by N`). Scoring is "whatever is on screen now" -- never script it blind.
- **Wikipedia 429s a burst.** `zine_canon.py` caches the category walk in `_index\wikipedia_canon.json`; re-walk only with `--refresh`.
- **A harvest holds the SQLite write lock.** WAL + commit-every-15s in `harvest`; other writers use a long timeout.
- **Never `os.walk` the library.** It lists every page image on the HDD: 330 zines took >2 min. `zine_library.iter_zines` lists exactly two levels (`<Source>\<Zine>`) and caches sidecars by (mtime, size) -- 0.3s. Read through it.
- **"magaZINE" contains "zine".** The one-word canon guard checks the Wikipedia disambiguator as a WHOLE WORD, or QZAP's "Out" matches *Out (magazine)*.
- **Two seeds at once duplicated zines** (each had its own snapshot of what was on disk). Seed now re-checks the disk before every download; the three duplicates from 2026-10-08 are in `_duplicates\` (excluded from the library, status `duplicate` in the index).
- **Commit before downloading.** `fetch_row` wrote the resolved file list and held that transaction through a 3-minute page-image download, locking every other writer.
- **Big IA scans** (Punk Planet, MRR: 350-450 MB) exceed the 250 MB cap; resolve always offers IA's page images (`page/n{i}_w1600.jpg`, ~0.4 MB each) LAST, so the fallback is automatic.
- **Issue spellings:** `#1`, `Issue 01`, `No. 1` normalise together.
- **Apostrophes:** `Master's` and `Master’s` must normalise identically (`norm_title` strips them first), or the same title from two sites never matches.

- **IA bundles:** some items hold hundreds of PDFs ("Anarchist Zine Archive"). Each PDF becomes its own zine; the item's popularity is shared, so it is divided by the PDF count before scoring or a bundle floods the top.
- **QZAP is one volunteer server.** Sequential, ≥2s between requests, User-Agent says who we are.
- **The art server holds code in memory** — restart it after editing `art_server.py`.
