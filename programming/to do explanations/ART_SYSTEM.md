---
tags: [programming, art, media, design-doc, voice-commands, ahk, miller, settings, performance]
---

# Art library + viewer — design doc

Status: **live.** 9,294 works on disk at `E:\Media\Art`, browsable by six facets, viewable fullscreen in Chrome, favouritable by voice.

This doc is the **map**. Every module here carries a long, genuinely good `WHY THIS EXISTS` docstring — read the module for the reasoning, read this for *which module* and *how they connect*. The connective tissue is the part that lived nowhere.

Related: [MEDIA_SYSTEM.md](MEDIA_SYSTEM.md) (the catalog this deliberately is **not** part of), [COMPLETION_LOG.md](COMPLETION_LOG.md), [QUOTES_SYSTEM.md](QUOTES_SYSTEM.md).

## The split that explains everything

Three layers, and confusing them is the main way to get lost:

| Layer | Question it answers | Where it lives |
|---|---|---|
| **Index** | What artwork exists *in the world*? | `art_index.db` (SQLite, ~200 MB, `_index/`) |
| **Library** | What is *downloaded*, and how is it tagged? | ~9,300 JSON sidecars beside their images |
| **Viewer state** | What is *on screen right now*? | `art_server.py` in memory |

That third layer is why the viewer is a **local web app** rather than a desktop image viewer. Because Chrome renders a page that `art_server.py` owns, "what is she looking at" is a plain HTTP question — which is the whole reason `show artist` can pivot the gallery to whoever painted the current image. A desktop viewer would give us a window title and nothing else.

The library is **not** in the unified media catalog (`E:\Media\catalog\*.json`). It is a sibling, like the quotes store: its own storage, its own query layer, deliberately absent from `media-query`.

## Pipeline

```
art_index.py     what exists          → art_index.db          (museum APIs, one-time-ish)
      ↓
art_plan.py      what to download     → capped PER ARTIST, so breadth is structural
      ↓
art_import.py    fetch + tag          → E:\Media\Art\<Theme>\<work>.{jpg,json}
      ↓
  enrichers, each rewriting sidecars in place:
    art_canon.py    notability (Wikipedia sitelink count → "greatest of all time")
    art_medium.py   73,600 raw medium strings → a 3-level browsable tree
    art_series.py   recover print series from titles (Los Caprichos, Liber Studiorum)
      ↓
art_gallery.py   ← THE CHOKEPOINT: iter_works() is the one door onto the library
      ↓                                    ↓
art_server.py (viewing)            ArtBrowserMenu.ahk (browsing)
  HTTP :8288 + Chrome                Miller, own process
      ↓                                    ↓
ArtGallery.ahk  ←──── voice / Stream Deck ────┘
```

`art_taste.py` reads the library sideways — lift, not raw counts — to answer *what does Jamie actually like*. `art_books_fetch.py` is a separate errand entirely (free art **books** → `E:\Media\Books\Art`), sharing only the subject.

Library provenance today: met 3,894 · aic 3,336 · wikidata 1,029 · si 769 · cma 141 · nga 125.

## `iter_works()` is the chokepoint — and it is cached

**Every** module above and every voice command reads the library through `art_gallery.iter_works()` — 18 call sites. That is the single most useful fact about this system: a change there reaches everything, and a *fault* there breaks everything at once, in ways that look unrelated.

### The 2026-08-24 bug (read this before touching the index)

"show art" and "open art" both hung ~30s then failed. Neither had a bug of its own. `iter_works()` opened all ~9,300 sidecars off the external HDD **on every call** — 10.7 MB spread across files averaging 1.2 KB, which is the worst possible shape for a spinning external drive. Cold, that took **48 seconds** against a 20s timeout in `ArtGallery.ahk`, so on a cold file cache the call could never finish. `open art` failed identically because the Miller shells `art_gallery.py facets` — same function, no timeout at all, so it just hung.

The tell in `ahk_event.log` is a `DISPATCH/in` and a failure tooltip ~24s apart, with `/api/state` still answering instantly (the server is `ThreadingHTTPServer`, so a hung `/api/gallery` does not block state).

**The fix:** consolidate the sidecars into one cache file, `_index\works_cache.json` (~10 MB), which reads back sequentially in well under a second.

| | before | after |
|---|---|---|
| `/api/gallery` — "show art" | hung >30s, failed | 0.8s |
| `art_gallery.py facets` — "open art" | hung >40s | 0.6s |
| full `iter_works()` pass | 48s cold | 1.6s build / 0.24s cached |

### How the cache stays honest

Validation is **per file** against `(mtime, size)`, taken from the directory scan itself — `os.scandir` hands back the stat it already had to fetch, so validation costs nothing extra. Only new or edited sidecars are re-read: adding 20 works costs 20 reads, not 9,300. The cache is rewritten only when something actually changed, so a no-op pass does not push 10 MB at the drive.

This trade matters, and it is worth being explicit about: it swaps a **slow, loud** failure for a **silent, permanent** one if invalidation is ever wrong. A stale cache shows outdated tags and titles forever, with no error anywhere and no timeout to signal it. Retagging is routine here (`art_series.py tag`, `art_canon.py`), so the contract is pinned in tests rather than left to code review — `Scripts/codebase_tools/tests/test_art_index_cache.py`: new / edited / deleted sidecars, cache actually used, only-the-changed-file re-read, and corrupt / stale-version / unwritable degradation.

The cache is derived data and **safe to delete** — the next call rebuilds it. There is deliberately no `reindex` command; a rebuild is just a missing file. Only that first rebuild pays the full cold cost, and it never happens twice.

Two scan-scope rules: `_galleries` (generated hardlinks) and `_index` (derived data, including the cache itself) are both skipped. `_index` skipping is also a small win — it holds a 434 KB `wikidata_canon.json` that was being parsed as a "work" on every pass and then discarded for having no image.

A 5-second in-process memo collapses the burst of `iter_works()` calls a single `art_server` request makes (six call sites) into one scan. Short on purpose: the scan is cheap once warm, staleness is not. Each writer subcommand is its own process, so the memo can never serve stale data across a write; `art_gallery.invalidate()` exists for a future writer that needs its own next pass to see what it just wrote.

## Downloading a subject is a drip, not a bulk run

A subject collection is hours of wall clock — 49 occult terms against up to 400 Met object lookups each — and the museum APIs rate-limit under it. So `cmd_subject` works the way `med_download.py` does for the meditation packs, and for the same reasons.

- **Resume.** Progress is checkpointed per term into `_index/_download_state.json`, so a kill, a reboot or a rate-limit stop costs at most the term in flight. Re-running the same command continues; `--restart` forces a clean pass. Without this a restart silently redid hours of *scanning*, because the only thing that skipped was work already on disk.
- **Cooldown.** A 403/429 stops the **whole** run and stamps `cooldown_until`; the next run refuses until it expires. Pushing through a rate limit is what turns a 3-hour block into a much longer one, so the rate limit is deliberately re-raised out of `met_subject_search` rather than swallowed per-term. `--force` overrides.
- **Variety.** `--max-per-artist` (default 6) spans the whole collection, not one term — museum relevance ranking clusters hard, and the clustering that hurts is the same name recurring *across* terms. `Unknown` is exempt: it is a bucket, not a person, and capping it would discard most anonymous Renaissance woodcuts.
- **Per-term quota.** `TERM_LIMITS` overrides `--limit-per-term` for the veins worth going deep on. A term that **fills** its quota is telling you there was more behind it — the museum ranks by relevance, so hitting the cap means good matches were still coming. `witches` came back 30/30, so it gets 150; `demons` (28) and `devil` (27) stopped short and were therefore nearly exhausted at 30, so raising them would buy nothing. This is the answer to "I want more witches" — a bigger number, not more witch-adjacent search terms.

  Raising a number makes that term pending again **automatically**, and only that term: `_term_pending` re-runs a term only if it filled its quota *and* the quota has since risen. An exhausted term stays done. Works already on disk return `skip` and do not consume the new quota, so a raise fetches only what is genuinely new. It lives in the data, not as a CLI flag, so the scheduled task picks it up with no arguments to remember.
- **Deepening ad hoc.** `--terms "witches,witch"` re-runs named terms without redoing the collection. Artist counts survive, so deepening never costs the spread.
- `art_import.py subject-status` reports done/remaining terms and any active cooldown. The durable log is `_index/_download.log`.

Unattended runs go through the **`ArtOccultDrip`** scheduled task, hourly, `MultipleInstances=IgnoreNew`. Hourly is safe precisely because the cooldown guard decides whether a given firing does anything; a blocked hour costs one log line.

### Three bugs this cost, all of which looked like nothing was wrong

1. **The throttle only fired on a hit.** `time.sleep(0.15)` sat after the `yield`, so every `continue` path — not public domain, no image, failed `subject_hit` — skipped it entirely. A term matching *nothing* hammered all 400 ids back-to-back with zero delay; six such terms earned the 403. The sleep is now in a `finally` so no early `continue` can ever skip it, and `MET_SUBJECT_BAIL_AFTER` stops a term that has found nothing by object 100.
2. **`sys.stdout` is None under `pythonw`.** The module-level `sys.stdout.reconfigure(...)` raised on **import**, so the scheduled task died before its first line of work. The only symptom was `LastTaskResult=1` and a log that never gained a line — the task looked like it was running hourly while doing nothing at all. Guarded now, with `_attach_console_fallback()` pointing the streams at the log so ordinary `print()` calls survive too. `med_download.py` already had the guard; `art_import.py` did not.
3. **A concurrent probe earned the block in the first place.** Bypassing the module's sequential, backed-off `_get` with a thread pool got the Met to 403 everything for ~20 minutes. Use `_get`.

All three are pinned in `Scripts/codebase_tools/tests/test_art_subject_resume.py`.

## The surface

### Viewing — `Helpers\ArtGallery.ahk` → HTTP → `art_server.py`

Thin by design: resolve intent → one API call → tooltip. Any real logic (tag resolution, ordering, what counts as "the artist") lives in Python where it is testable.

| Voice | Function |
|---|---|
| `show art` | `OpenArtRandom` — shuffle the whole library |
| `show art <tag>` | `OpenArtTag` — one tag, or `a,b` for works carrying both |
| `show artist` / `show genre` / `show medium` | `ArtShowArtist` / `ArtShowGenre` / `ArtShowMedium` — pivot to whatever is on screen |
| `favorite` / `unfavorite` | `ArtFavorite` / `ArtUnfavorite` — affinity ±1 (`affinity.py`) |
| `show scroll` / `show scroll <n>` | `ArtToggleScroll` / `ArtScrollSeconds` — auto-advance |
| `show back` / `show forward` | `ArtHistoryBack` / `ArtHistoryForward` |
| `show overlay` / `show commands` | `ArtToggleOverlay` / `ArtToggleCommands` |

Rules: `art_commands.py` (global) and `art_viewer_commands.py` (scoped to the viewer). `_ArtEnsureServer` launches the server with `pythonw` via `Run` so it outlives the MAINFUN dispatcher, then polls `/api/state` for ~5s.

### Browsing — `Helpers\ArtBrowserMenu.ahk` (voice `open art`)

A Miller, in its own process per the enforced convention (`LaunchMillerViewer` → `Scripts\ArtBrowserViewer.ahk` → `_MillerViewerShow`). Roots: Favorites · Catalogue · Genres · Artists · Eras · Subjects · Regions · Mediums · Museum terms · Everything · Shuffle all · ⚙ Settings.

Right-pane **image preview** is the whole point — arrowing a list of works shows each *picture*, not a description of it. For an art library, seeing beats reading.

Two rules that are easy to break:

- Every leaf action **dispatches** through `_ArtMainfun` rather than calling `ArtGallery.ahk` directly. The viewer is its own process with a light include closure, and `ahk_include_closure.py` skips `Scripts/` — so a direct `OpenArtFile()` call would pass validate *and* the closure check and then throw at runtime.
- Tunables are **declared settings** (`artbrowser.*`), never constants — `works_per_facet`, `min_facet_count`, `preview_images`, `autoplay_seconds`, `hide_buried`, `pictures_only`, `painter_first`, `painting_bias`, `caption_flash_seconds`. The `⚙ Settings` row is already wired; see [SETTINGS_SYSTEM.md](SETTINGS_SYSTEM.md).

## Ordering rules worth knowing

`Gallery.load()` is shuffled **by default** — sorted order groups every work by one artist before moving on, which makes a gallery feel like a filing cabinet rather than a wander. Four exceptions, in precedence order:

1. **Favourites** sort best-first by affinity, shuffled *within* each score so one artist cannot clump at the top of a tier.
2. **A series** never shuffles, whatever the caller asked. Los Caprichos runs 1–80 and was made to be read in order.
3. **One artist** opens with what the *world* knows (`notability`), because a museum holds what it happens to own — Jamie's Blake is 89 works, nearly all Virgil pastorals and Job plates, and almost none of the pictures Blake is known for.
4. **Buried** works (affinity < 0) are held back from every gallery except one that asks for `buried` by name.

An explicit request beats a default: `pictures_only` is not applied when the query already names a form, or asking for `sculpture` would silently intersect with `pictures` and come back empty.

## Traps

- **A hang in one art command means all of them are broken.** They share `iter_works()`. Do not debug the command; check the chokepoint.
- **`art_server.py` holds `art_gallery` in memory from when it started.** Editing the module changes nothing until the server restarts — silently, since the old code keeps answering. Kill the `pythonw` on port 8288 and let `_ArtEnsureServer` bring it back.
- **The library is small files, not big ones.** Any new per-work disk pass re-creates the 2026-08-24 bug. Read through `iter_works()`.
- **`_index` holds a 200 MB SQLite and two large CSVs.** It is not a place to iterate casually.
