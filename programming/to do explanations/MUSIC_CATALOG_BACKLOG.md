---
tags: [music, catalog, spotify, todo, planning]
---
# Music catalog — feature backlog + delivery plan

Living plan for the unified music catalog (`E:/Media/catalog/music.json`).
Companion design doc: [[MEDIA_SYSTEM]]. AHK orchestrator:
`Helpers/SpotifyAddFunctions.ahk`. Python ingest: `Scripts/MediaCatalog/`.
**Read [[SPOTIFY_SCRAPING_INTERNALS]] before touching the scrape pipeline** —
covers the /related dump format, wrong-URL detection layers, Spotify
broken-state signatures, and the off-by-one parser bug that polluted every
placeholder pre-2026-06-10.

Each item below is sized into **wave 1** (quick wins, shippable in one
session), **wave 2** (medium — needs new GUI / new schema field), or
**wave 3** (big — needs new architecture, e.g. background workers,
relatedness graph).

Add to this file when new asks surface; tick items as they ship.

## Wave 1 — quick wins (shippable now)

- [x] Scrape pipeline correctness pass (2026-06-10)
  - `_related_from_dump` rewritten: coord-pair `[link]`↔`[group] AID` lines
    instead of FIFO. Old version was off-by-one for every card and poisoned
    every placeholder. Background + signatures: [[SPOTIFY_SCRAPING_INTERNALS]].
  - All 1280 stale placeholders nuked + 65 source-artist `related` arrays
    cleared. Backups: `E:/Media/catalog/music.json.bak-*`.
  - 7 catalog artists had wrong Spotify URLs (Elliott Smith → Newfound
    Interest in Connecticut, Built To Spill → Sparklehorse, etc.) — fixed
    from the queue's `mb_url_rel` URLs which were mostly correct. Bad
    placeholder refs they generated were scrubbed.
  - New CLI: `verify-spotify-url` (HTTP og:title fuzzy-match, ~500ms,
    retries once). Wired into `BackfillPlaceholdersForAllSavedArtists` as
    a Chrome-skip pre-flight.
  - `save-placeholder-artists` got a window-title `WRONG_PAGE` guard
    (defense-in-depth against Chrome diverging).
  - `BackfillPlaceholdersForAllSavedArtists` now: takes a `maxCount` arg,
    adaptive page-ready poll (was hardcoded `Sleep(2500)`), detects three
    distinct Spotify-broken signatures + silent never-rendered, aborts
    after 3 consecutive broken pages, sanitizes catalog IDs to avoid the
    NTFS ADS gotcha.

- [x] Right-side number entry in genre picker (legacy `_pickByNumber`)
  - Was: typing "54" for a right-row pick said "no entry numbered 54".
  - Fix: `_pickByNumber` now falls through to `_tryPickRightByNumber`
    when the left catalog miss happens AND a right pane is wired.

- [x] Hardcode/softcode default = softcode
  - Was: Enter = hardcode (modal's default button #1 = hardcode).
  - Fix: swap order so Softcode is button #1; Hardcode is button #2.

- [x] `/related` zero-results race
  - Was: first dump after nav often parsed 0 (Spotify lazy-render).
  - Fix: 2s wait → 3.5s + retry-once-on-zero (extra 2.5s + re-dump).

- [x] Progress tooltips during enrichment fetch + nav
  - "Fetching enrichment...", "Loading related artists...", "Reading
    page...", "Found N related — saving placeholders...", etc.

- [x] Diagnostic LogEvent calls around AlbumPhrase, ExploreRelated,
  RelatedPlaceholder.

## Wave 2 — medium (next session candidates)

### Favorite / to-listen status for artists + albums

- New `personal_status` field on every catalog row: `""`, `"favorite"`,
  `"to_listen"`. Set on either artists OR albums independently.
- AHK: in `_ManageExistingArtist`, add "Mark favorite / to-listen" rows.
  In `_DoEditAlbumPhrases`, add per-row action 2 = mark favorite, 3 =
  to_listen.
- Voice commands:
  - `spot favorite` → random favorite-tagged item URL → SpotOpenInChrome
  - `spot listen` → random to_listen-tagged item URL
  - `spot all` → random across the full library (skip placeholders)
- Python: `media_ingest update-personal-status <id> <value>` +
  `pick-random --personal-status favorite|to_listen|any`.

### Genre of the week

- Manual cycle: an INIDATA `current_week_genre.txt` (or a field in
  `INIDATA/catalog_meta.json`) tracks the currently-active genre.
- Voice: `spot week` → random item under that genre. `set week genre <X>`
  picks one manually.
- Auto-rotation rule for the week genre (LATER, after relatedness
  module exists): on rotate, pick a genre whose item count > 5 AND whose
  taxonomy distance from the previous N weeks' picks is high (avoid
  "indie folk" → "folk" → "indie" rotation).

### Manual "to scrape" feed (queue / log)

- New file: `INIDATA/scrape_queue.json` — array of `{name, spotify_url,
  source, added_at, status: "queued"|"in_progress"|"done"|"skipped"}`.
- AHK: voice `add to log <X>` / `scrape feed` / `feed next`.
- After every `add artist`, ALSO offer "save for later" alongside
  "explore now / done". Each "save for later" entry pushes to the queue.
- `feed next` pops the next queued entry, navigates, runs the add flow.
- When the queue empties AND the user opts in, auto-suggest from
  placeholders ranked by `ref_count` desc.

### Miller-everywhere in the library manager (2026-06-09)

- [x] Browse Library (Genre → Artist → Album) now displays friendly
  names in the breadcrumb (e.g. "Genres: Indie > Artists: Adrianne
  Lenker > [Albums]") instead of slug IDs.
- [x] Listen "Pick a genre" and "Pick an artist" converted to Miller
  drills with leaf = play.
- [x] Manage artists collapses into the same Miller as "Pick an
  artist" (single shared flow). N.4 on an artist row opens the
  Manage menu.
- [x] Favorites / To-listen browse: two new Listen submenu entries
  open a Miller view filtered to that pool. Leaf = play; row actions
  toggle/swap the status.
- [x] Miller picker now supports per-level `row_actions` (N.M syntax):
  type `5.2` Enter to fire action "2" on row 5. Visible per-level
  hint surfaces the action map.
  - Genre level: N.2 = Set as week
  - Artist level: N.2 = Favorite, N.3 = To-listen, N.4 = Manage
  - Album level: N.2 = Favorite, N.3 = To-listen
- [x] Auto-suggest top placeholders when "Start processing" runs
  against an empty queue. Multi-pick over top-10 by ref_count; picks
  are queued + processed.

## Wave 3 — bigger structural work

### Background scraping while she's in other GUIs

- Goal: enrichment + /related dump run AHEAD of the modal so the user
  never waits for a network call.
- Approach A: when an artist is queued to the scrape feed, kick off a
  background `enrichment.py prepare-artist-add` immediately + save the
  JSON to a per-artist cache file. When she later runs the add flow,
  AHK reads the cache (instant).
- Approach B: pre-fetch the moment the URL appears on screen (any
  Spotify artist page = run prepare in the background; if she then
  decides to add, the preview shows instantly).
- Architecture: a single long-running Python worker (in `AlwaysOn/`?)
  that watches a queue file and processes jobs. NOT lightweight; needs
  IPC + cancellation + status reporting.

### Relatedness graph (powers cycling + smart suggestion)

- Sources we already capture: `related[]` on each artist row (Spotify
  /related), `lastfm_similar` in enrichment, MB relationships.
- Build: a graph where nodes are artist names, edges are co-occurrences
  weighted by source + match score.
- Uses:
  - "spot week" rotation that avoids genre clusters.
  - Placeholder ranking (which placeholder is most central in her
    library?).
  - Visualization (eventual GUI).
- Implementation: derived offline from `music.json`; cached in
  `INIDATA/relatedness_graph.json`; refreshed on every artist save.

### Main library manager GUI

- One-stop hub: browse by genre, by artist, by status. Set week genre.
  Promote placeholders. Review the scrape queue. See orphan rows.
- Likely a dedicated AHK Gui with multiple tabs OR (better) a web app
  driven by a Python data layer + a local browser.
- Owns: every "I'd like a UI for this eventually" item from Jamie's
  asks.

## Cross-cutting / nice-to-have

- Atomic write everywhere (done — `mc.write_media_type` uses
  `os.replace`).
- Placeholder promotion auto-merges `related_from` back-refs on the
  promoted row (currently dropped — preserve as `was_placeholder_for`?).
- `spot related` voice command works standalone (done).
- "add to log" / "add to feed" voice command (wave 2 queue feature).

## Process notes

- **Bug priority**: any time Jamie reports a regression or a "feels
  broken" UX issue, treat it as wave 1 even if the underlying fix is
  bigger. Stability of the surface > new features.
- **Schema migrations**: when a new field is added to `music.json`,
  every read path needs to handle its absence (`it.get("personal_status")
  or ""`). No bumping a "schema_version" — old rows just lack the field.
- **No premature abstraction**: implement each feature directly. If
  wave 2 features turn out to share infrastructure, refactor THEN.
