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
