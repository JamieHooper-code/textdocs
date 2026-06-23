---
tags: [programming, media, recommendations, design-doc, books, voice-commands, qmd]
---

# Unified Media Recommendations System — design doc

Status: **design / phased build.** Books are already live (see [COMPLETION_LOG.md](COMPLETION_LOG.md) "Book / reading integration"). This doc generalizes that book catalog into one system that holds **every** media type Jamie collects — books, movies, TV, anime, music (artists/albums), podcasts, YouTube channels, websites, "places to find theory," games — and makes the whole thing **searchable, organized, and smartly queryable** without smashing the types into mush.

## The vision (Jamie's words)

> "So many things all floating around in my system… I'm interested in some form of unification that does not just smash them all together but makes it all searchable and organized in the best way possible."

Two concrete behaviors define success:

1. **Generic vs scoped search.** "anarchist" returns anarchist *books AND movies AND podcasts AND theory-sources*. "anarchist books" returns just the books. One shared tag vocabulary makes the cross-type net work; a `--type` filter narrows it.
2. **Smart natural-language recommendation** ✅ (built 2026-06). Drop a sentence — *"a poetic book about the queer Black experience in America"* — and get ranked fits. **Lives inside the unified manager** (no separate voice command): the "✨ Recommend" row at the top of `open media` prompts for a wish, results show with a one-line *why* per pick, `1`/Enter marks a pick in-progress.
   - Backend: **`clog media-recommend "<wish>"`** → `_llm()` → `llm_gateway.py` (`claude -p`, Max sub, $0). Sends the catalog (id/type/title/creator/tags + description if present) + the wish; the model **reasons over its own knowledge of these works plus the tags**, returns a ranked JSON array of `{id, why}`. Honors the hub's `--type` scope; skips song/album subtypes (recommends artists).
   - **Descriptions are optional** — the model already knows most catalogued works, so recommendations are strong even before any `description` is backfilled (that backfill only sharpens edge cases; it's a TODO).
   - QMD `media` collection remains a possible future alternative/augment, not needed for v1.

## Architecture: parallel catalogs, one unified query, one shared tag vocabulary

```
E:\Media\
  Books\<Author>\library.json      ← BOOKS partition (unchanged; Kindle-managed)
  catalog\                         ← one flat <type>.json per non-book type
    movie.json                       (filenames are the SINGULAR type name,
    tv.json                           matching the `type` field: movie/tv/
    anime.json                        anime/music/podcast/youtube/web/game)
    music.json                       ← artists + albums + songs (subtype + parent)
    podcast.json
    youtube.json
    web.json                         ← websites / blogs / "places to find theory"
    game.json
```

- **Books are just another type.** They keep `type: "book"` and join the unified query + tag + umbrella layer. Physically they stay in the Kindle per-author `library.json` files so `kindle_import.py` re-scan keeps working — "unified conceptually, minimal change physically." Every other type is a flat per-type JSON under `catalog\`.
- **One query layer spans all catalogs.** `media-query --tags anarchism` reads books (per-author loader) + every `catalog\*.json`, tags each row with its `type`. `--type movie` scopes it. Same umbrella expansion + `--group` the book system already has.
- **One shared tag taxonomy.** `book_tag_aliases.json` → `media_tag_aliases.json` (aliases + stopwords + **parents** + **umbrellas**), shared by every type and by `kindle_import`. **Thematic** umbrellas (`theory`, `identity`, `history`, `philosophy`, `spirituality`…) apply to ANY medium — a movie tagged `anarchism` rolls up to `theory` exactly like a book. **Genre** tags are added per-medium (film genres, music genres) with their own umbrellas, in the same file. This is the "reformat to media tags" idea: shared thematic facets, medium-specific genre facets, one file.

### Item schema (shared core)

```json
{
  "id": "movie:no_other_land",
  "type": "movie",
  "title": "No Other Land",
  "creator": "Basel Adra & Yuval Abraham",
  "tags": [{"tag": "palestine", "src": "manual"}, {"tag": "documentary", "src": "manual"}],
  "status": "queued",
  "recommended_by": "Caroline",
  "rating": null,
  "url": "https://...",
  "description": "A Palestinian-Israeli documentary on the destruction of Masafer Yatta...",
  "notes": ""
}
```

- **`type`** — book · movie · tv · anime · music · podcast · youtube · web · game.
- **`creator`** — author / director / artist / host / channel (one field, read per type).
- **`status`** — one internal enum **`queued` / `active` / `done`**, shown with type-aware labels: to-read/reading/read, to-watch/watching/watched, to-listen/listening/heard. (Books keep their existing `to_read`/`reading`/`finished` values; the query layer maps them.)
- **`description`** — short blurb powering smart search. Populated by: online lookup (OpenLibrary/TMDB/etc. where available) → fallback to a local-LLM/Haiku one-liner via `llm_gateway`. Optional at import; backfillable in a batch pass.
- **`url`** — where to read/watch/buy. For web/youtube it's the resource link; for books it's the "where to read" page. `add read` auto-captures the active browser URL (via `ChromeCurrentUrl`) when a book is added from a website; `book-add --url` / `media-add --url` / staging-TSV 6th column also set it.
- **`tags`** — leaf tags only; umbrellas derived at query/render time (see book taxonomy in COMPLETION_LOG.md).

## Migration plan (per-type, staged + reviewed — same as books)

Each scattered source → a reviewable **staging TSV** → `import-staged` (dry-run default, `--commit` to write, dedupes vs catalog + within file) → Jamie reviews → commit. Recommender attribution (`(hannah)`, `(rigo)`, "from Kayla") is parsed into `recommended_by`. The mixed dump `MEDIACONTENTANIMESHOWSMOVIES.txt` is split by its section headers into per-type TSVs.

Source inventory (from the 2026-06 vault survey):

| Type | Sources |
|---|---|
| Books | TOCONSUMEBOOKS.txt ✅(111 imported) · TOCONSUMEBOOKSGENDER/RACE.txt ✅(21 imported) · COSMERE NOTES.txt · TOLKIENREADINGORDER.txt · EDUBOOKS.txt · TOCONSUMESTOICISM.txt · SavedLinks/{Books,PoliticalBooks}.md |
| Movies | MOVIES.txt (clean, tone-grouped, "Olive" recs) · PoliticalMovies.md · SavedLinks/SpanishMovies.md |
| TV / Anime | ANIMES.txt ⚠️(creds — see below) · MEDIACONTENT…txt (SHOWS + ANIME sections) |
| Music | MUSIC.txt · MEDIACONTENT…txt (MUSIC section) · Songs to Sing.txt |
| Podcasts | MEDIACONTENT…txt (PODCASTS) · Psychology.md |
| YouTube | SPANISH YOUTUBE.txt (curated, topic-grouped) |
| Web / theory sources | PoliticalLinks.md · REDREADINGCIRCLE.txt · Poetry.md · TUTORING LINKS.txt |
| Games | MEDIACONTENT…txt (LET'S PLAYS/VIDJA) |

**Duplicates to collapse on import:** `BOOKS.txt` == `TOCONSUMEBOOKS.txt`; `TOCONSUMEMEDIACONTENT…` == `MEDIACONTENT…`. Empty/broken: `TOSHOWWINTER.txt`, `TOWATCHLCK.txt`.

⚠️ **`ANIMES.txt` contains plaintext passwords/credentials** mixed with the anime list. Move those into the encrypted slot system (CopyPasteManager) and out of the vault BEFORE importing that file. Not touched by this system.

## Voice / UI

**Live (2026-06):**
- **`open media`** → `OpenMedia` — unified two-pane hub (`CompletionLogFunctions.ahk`), grouped by thematic umbrella so cross-type themes cluster (anarchist books + films together under THEORY). Cols Type/Title/Creator/Status/Tags; filter box narrows by any column. Per-item actions: `1`=mark in-progress · `N.2`=done · `N.3`=edit tags · `N.4`=recommender · `N.5`=remove. Works on books (delegates to the library) AND catalog items transparently via `clog _find_media`.
- **`add media`** → `AddMedia` — type picker (movie/tv/anime/music/podcast/youtube/web/game/book) + title/creator/tags/recommender form. `book` routes to `AddRead` (richer online lookup); everything else → `media-add` (status `queued`).
- clog backend: `media-query [--type --tags --status --recommender --group umbrella|type]` · `media-add` · `media-set-status` · `media-set-tags` · `media-remove` · `media-set-recommender`. `import-staged --type <t>` bulk-imports any type.
- **Note:** the old `open media` directory shortcut (opened `E:\Media` in Explorer) was renamed to **`open media direct`** (key in `directories.json`) to free the phrase for this hub.

**Later:**
- `add movie` / `add album` / `add podcast` — type-specific, with browser auto-detect (Letterboxd/TMDB/Spotify in a `media_sites.json` sibling of `book_sites.json`).
- `open movies` / `open music` — type-scoped (`OpenMedia` already takes a `filterType` arg; just needs the phrases).
- `recommend <sentence>` — QMD `media` semantic search or Claude-farming.

## Build order (phased)

1. ✅ **Engine generalize** (2026-06) — `media-query` / `media-add` / `media-set-*` / `media-remove` / `import-staged --type`; books fold in via a read-only adapter (`_book_as_media`) + `_find_media` id routing. Tag taxonomy still in `book_tag_aliases.json` (shared; rename to `media_tag_aliases.json` deferred — cosmetic).
2. ✅ **Movies** (2026-06) — 36 from `MOVIES.txt` + `PoliticalMovies.md`; cross-type query proven ("queer" → 14 books + 5 movies).
3. ✅ **Voice + hub** (2026-06) — `open media` / `add media` (see Voice / UI). [needs a `reboot caster` to go live]
4. ✅ **Mixed dump** (2026-06) — `MEDIACONTENT…txt` split by ACTUAL type (not its header) into per-type TSVs: anime 17, tv 22, movie 5, game 5, podcast 1, book 4, music 8 (nested). Reclassified misfiled items (Akira→anime, Parasite→Parasyte, I May Destroy You/Kipo→tv). Catalog now book 153 / movie 41 / tv 22 / anime 17 / music 8 / game 5 / podcast 1.
5. ✅ **Book sources + web/youtube** (2026-06) — Cosmere 28 (series tag `cosmere`), Tolkien 7 (`tolkien`), reading-circle 3, Settlers/FNFI as books-with-URLs; SPANISH YOUTUBE 35 (`type=youtube`), PoliticalLinks/Poetry/zine links 9 (`type=web`). Books gained a `url` field; `add read` now captures the source page. Music nesting renders in the hub; `open media <type>` scoped phrases live; orphans folded. **Catalog ≈331 items, 9 types.**
6. ✅ **Smart recommendation** (2026-06) — `media-recommend` via `llm_gateway`, surfaced as the "✨ Recommend" row in the unified `open media` manager. The manager now does everything in one screen: browse (umbrella-grouped) · add (`+ Add media`) · recommend · per-item status/tags/recommender/remove.
7. Remaining (TODOs in `Lists/TODO/Macros.md`, #media): backfill `description`; per-type browser auto-detect (`media_sites.json`); `add media` URL capture for non-book types; `media_tag_aliases.json` rename. Dropped: `TUTORING LINKS.txt` (Jamie doesn't want it). Left as-is: the two deprecated-credential files.

## Resolved schema decisions (2026-06)

- **Music = nested.** `type: music` entries carry `subtype` (artist / album / song) and a **`parent`** id. Artist is top-level (`parent: null`); albums post **under** their artist (`parent: <artist id>`); songs under their album (or artist). The hub renders them nested. So a bare "get into Nujabes" rec is one artist entry; specific albums/songs hang off it.
- **Websites / theory-sources:** `queued`/`done` status only (no middle "active" state — you don't "finish-reading" a site the same way).
- **Series/reading-order** (Cosmere, Tolkien, Mistborn): kept in `notes` for now; no modeled link relationship yet. **Superseded for books by [[MEDIA_ENRICHMENT_SYSTEM]] — books now carry a structured `series {name, position}`, hub grouped + sorted by position.**

## Media enrichment — "keep the best data possible"

**Full design: [[MEDIA_ENRICHMENT_SYSTEM]]** (locked 2026-06-19; build deferred). One-line: for every book (and later every media type), auto-collect the richest data possible — description, genre tags, cover, series, bibliographic — behind a Spotify-style confirmation step, on a generic provider seam so TV/movies reuse the core. Recommendations are out of scope for now; we bank genres/series/subjects as the future engine's fuel. See that doc for the API research, source strategy, data-model, genre-conforming, series handling, confirmation UX, and the deferred Anna's-Archive-download + StoryGraph-scraper seams.
