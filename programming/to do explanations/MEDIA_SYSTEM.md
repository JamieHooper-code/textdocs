---
tags: [programming, media, recommendations, design-doc, books, voice-commands, qmd, people, tags, plex, tv, episodes, favorites, uia, automation]
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
- **`open media`** → `OpenMedia` — the unified hub, **rebuilt as a Miller (`Helpers\MediaHubMenu.ahk`, 2026-07)**, replacing the old impregnable flat two-pane list. It is a **thin aggregator that owns no data**: every top node is either a live DELEGATE of an independently-working system (mounted via that system's `_XAsNode()`) or a thin `media-query` branch, so updating a component / the catalog / a tag updates the hub automatically — no parallel drift. Root: **Books** (`_BookManagerAsNode` + `_ReadingAsNode`, Reading folded under Books) · **Quotes** (`_QuotesAsNode`) · **Music** sub-hub (Spotify library `_LibAsNode` + Ambient + catalogued) · **Movies / TV / Anime / Podcasts / YouTube / Web / Games** (`media-query --type X`) · **Photos / Videos** (personal media — **`photo`/`video` are now catalog types**, in `MEDIA_TYPES` + `FS_MEDIA_TYPES`; a scanner **`Scripts\MediaCatalog\media_fs_import.py`** — the disk→catalog bridge, like `kindle_import` for books — walks `E:\Media\{Photos,Videos}`, upserts one path-keyed entry per file with **folder-derived tags** (`Personal\Kink & Nudity` → `personal, kink, nudity`, `src="folder"` so hand-tags survive a rescan) and **drops entries whose file is gone** (disk = truth for existence, catalog overlays metadata). The hub's Photos/Videos nodes **serve from the catalog** grouped by folder (`media_fs_import.py folders`/`list`, shelled from AHK via `_MediaFsRun`), each with a **Rescan** action; files open in their default app on Enter (`N.2` reveal · `N.3` edit tags · `N.4` remove-from-catalog). Because they're catalog items they're tag/theme/searchable like any type — **but excluded from the LLM recommender** (`cmd_media_recommend` skips `FS_MEDIA_TYPES`)) · lenses **By Theme** (umbrella, cross-type) / **By Status** / **By Recommender** · Recommend + Add-media leaves. Item leaves reuse the existing actions (`_OpenMediaPerform`, kept in `CompletionLogFunctions.ahk`): Enter = mark in-progress · `N.2` done · `N.3` tags · `N.4` recommender · `N.5` remove. Built Method B (GuiHost, own-process, no-float) so every mounted `AsNode` resolves; a hub `search_index` over non-music catalog items keeps title search instant; exposes `_MediaHubAsNode()` for the future master menu. Deep-links preserved: `open media <type>` drills straight into that type (`music`→sub-hub), and a packed `tag:`/`rec:` token drills into a lens. The old flat GUI (`_OpenMediaBuildCatalog` / `_OpenMediaRightPane` / `_OpenMediaRightAction`) was removed.
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

## People — the recommender dimension (built 2026-07-19)

`recommended_by` was a free-text string per item. It is now a link to a **person record**, making "who put me onto this" a browsable dimension of the whole catalog rather than a note on each row.

**Why records, not strings:** renames stop orphaning things (items link by id); people carry **tags**, so *story and steep* becomes a real grouping you can browse; and spelling/casing drift collapses through aliases. The migration found 25 distinct recommenders across books + catalog, spread over 66 items.

### Storage

`E:\Media\catalog\person.json`, shape `{"items": [...]}` — deliberately the same shape and directory as a media type, so `MediaCatalog/people.py` reuses `load_media_type` / `write_media_type` / `catalog_lock` verbatim (atomic writes + cross-process locking for free). But **`person` is not in `MEDIA_TYPES`**, so `media-query` / `iter_catalog` / the recommend engine never surface people as things to read or watch. It is a dimension of the catalog, not a member of it.

```json
{ "id": "person:lina", "name": "Lina", "aliases": ["lina h"],
  "tags": ["story and steep"], "notes": "", "added": "2026-07-19T..." }
```

Person tags share the one `media_tags.json` vocabulary via `applies_to: ["person"]` (added to `EXTRA_APPLIES_TYPES` alongside `quote`/`article`). Because `person` is absent from `MEDIA_TYPES`, a book picker is never offered "book club" and a person picker is never offered "anarchism" — the same `applies_to` gate that already separates quote tags.

### Linking — and why nothing downstream broke

Items gain `recommended_by_ids: ["person:lina", ...]` (**multiple** recommenders per item — two people recommending one book is real). Every write re-derives the legacy `recommended_by` string as the comma-joined display names. That one line is load-bearing: `book-query` col 5, `media-query` col 7, the `--recommender` substring filters, the hub filter box and `_book_as_media` all kept working with **zero** changes. Read paths lazily resolve a legacy string through name/alias matching, so un-migrated items still report ids without a write.

**Author level:** `author_recommended_by_ids` sits beside `author_tags` in `library.json` — "she put me onto this *writer*", so the next book by that author arrives already attributed. `book-promote-recommender` is additive, mirroring `book-promote-tags`.

### Commands (clog)

`person-list [--tag]` · `person-add` · `person-set-tags` · `person-rename` · `person-merge --into` · `person-remove` · `person-items` · `person-migrate [--commit]` · `rec-rank --id|--ids` · `rec-set --ids --add|--remove` · `book-promote-recommender` · `author-recommenders` · `people-sync`.

`rec-rank` is the picker feed and mirrors `tag-rank`'s contract: bands `current` / `partial` (group mode, "k/n" so one toggle applies to a whole batch) / `group:<tag>` / `other`. `tag-rank --id person:x --for-type person` ranks a person's own tags against the *people* corpus (a person's groups co-occur with other people's groups, never with a book's themes) — the emit half was factored into `_tag_rank_emit` so both corpora share one output format.

### UI

A **recommender is a tag from the UI's point of view**, so it plugs into the existing `MillerTags` control rather than a second picker — banding, group mode, add-new and N.M row actions all come for free. `MillerTags` gained `add_new_label` / `add_new_hint` / `add_new_title` / `add_new_body` / `detail` opts so the control can name its own noun (defaults preserve every existing backend verbatim).

- **Per item:** "Recommended by ▸" on every book node (Current reads, To read, Recently added). Replaces the old free-text `_SetBookRecommender` prompt — the valid answers are a known finite set, so they're **picked, never typed** (gui-conventions).
- **Per batch:** "Recommended by (all N)" on a download session, next to the existing batch-tag control. A sitting's books usually came from one conversation.
- **Browse:** the **Recommended by** root row → Groups (person tags) → people → their items, with per-person Groups / Rename / Merge. Merge is a drill-in pick, not a typed name.

### Voice

`read recommend` opens the section; `read <person>` drills to one person. Both are literals/Choices sharing the `read ` prefix with `read <book>` / `read <genre>`, so **`_sync_people_choices` drops any person whose name a book or genre already claims** and reports it on stderr — otherwise Dragon can't disambiguate and one of the two commands silently dies. Dropped people stay fully usable in the menu; only the spoken shortcut is withheld. `person-list` col 5 carries the actual spoken phrase (empty when dropped) so the menu's Say column can't advertise a phrase that does nothing.

**Not** `completion_friends.json` — that is a separate, older store for "log \<friend> walk" activity logging. Folding the two together is a sensible future move.

## Links — capture folded into the catalog (built 2026-08-21)

`web` was always a media type, but nothing *captured into it*. The `save link
<category>` command (`SaveCurrentLink` → `LinksaverToVault.ahk`) wrote markdown
bullets into `ObsidianVault\<Category>.md` against its own private inline-`#tag`
vocabulary. So there were two link libraries, and the 2026-06 migration
(phase 5) imported some of the same pages into `web.json` while the capture
command kept writing to the vault — **5 of 26 vault bullets were already
catalogued**. The fix was not a new section; it was pointing capture at the
section that already existed.

**`grab link`** (`Helpers/LinkGrabber.ahk`) replaces it. Jamie's rule: the word
"save" is retired across this system.

- **Routing.** `INIDATA/media_sites.json` (64 rules) maps a URL to a type +
  kind + status; `Scripts/MediaCatalog/media_sites.py` is the ONE interpreter
  (AHK shells it, so the two sides can't drift). YouTube → `youtube/video` or
  `/channel`, Letterboxd → `movie`, Wikipedia → `web/reference` **statusless**.
  Unmatched → `web/article`. A rule naming an unknown type/subtype is dropped
  with a warning, never stored — `media_sites.py check` validates all of them.
- **Categories became tags.** `PoliticalBooks` was a filename pretending to be
  a taxonomy; it is `political` + `books`, which the umbrella layer already
  models. `grab link <tag>` pre-seeds one. `AddSavedLinkCategory`,
  `saved_link_categories.json`, `vault_tags_scan.py` and the twelve `_Obsidian*`
  markdown helpers are gone (~370 lines + a second tag vocabulary).
- **Capture reads the URL properly.** The old path hand-rolled `Send "^l"` +
  clipboard save/restore (visible flash, ~5s hang on YouTube, drops fullscreen).
  Now `ChromeCurrentUrl()`.
- **Idempotent.** Re-grabbing a saved page opens *its* tags instead of making a
  near-duplicate. Dedupe is `_canon_url` identity (tracking params + fragment
  stripped) across **every** type, via `clog media-find-url`.

### The capture form is a persistent VIEWER (rebuilt same day)

The first build called `_MillerColumnPickGui` inline from the MAINFUN dispatcher
under a `; miller-modal-ok` exemption. That exemption is for a genuine
pick-and-dismiss modal; this is a form Jamie sits in, edits fields in, and opens
sub-dialogs from. She hit all three documented failure modes within minutes:

- it **floated** (always-on-top) over whatever she was reading;
- every sub-dialog (title box, new-type form) opened **behind** it, because a
  floating parent outranks a normal child window;
- it **died the moment she ran another voice command**, because it lived in the
  dispatcher process the next dispatch tears down.

Now it uses the enforced trio — `LaunchMillerViewer` → `Scripts/LinkGrabberViewer.ahk`
→ `_MillerViewerShow` (which hard-forces `always_on_top := false`). Verified:
`WS_EX_TOPMOST` is false, and the window survives an unrelated MAINFUN dispatch.

**State crosses the process boundary as a file.** Reading Chrome's URL must
happen in the dispatcher, where Chrome is foreground; by the time the viewer
exists, the front window is the form. So `GrabLink()` captures + resolves
defaults, writes one state JSON to `%TEMP%`, and passes the path as `A_Args[1]`.
An already-open form is **closed first**, because `LaunchMillerViewer` reuses a
same-titled window and would otherwise show the previous page's state.

The profile editor (`open link profiles` / "link defaults") is the same trio:
`Helpers/LinkProfilesMenu.ahk` + `Scripts/LinkProfilesViewer.ahk`.

**Testing note:** `ahk.py show GrabLink` cannot test this end-to-end — it runs in
no-focus-steal mode, so Chrome can't be foreground and `GrabLink` correctly bails
with "no URL from the browser". Test by launching `Scripts/LinkGrabberViewer.ahk`
directly with a state file. Tests: `Helpers/Tests/Gui/test_link_grabber_menu.ahk`
(+ fixture), registered in the runner — `MAINFUN.bat RunGuiTests link_grabber`.

### Per-site defaults — link profiles

A profile answers "when I grab a link from HERE, what should it already be?" —
type, kind, status, tags, creator, recommender, title cleanup. Store
`INIDATA/link_profiles.json`, engine `Scripts/MediaCatalog/link_profiles.py`
(this REPLACED the earlier `media_sites.json`/`media_sites.py` router, which
could only pick a type).

**Profiles LAYER, they do not compete.** Every matching profile applies,
least-specific first; scalars are overridden by the more specific layer and
**`tags` accumulate**. That is what makes "specific subreddits get their own
defaults" work without restating the parent's:

```
.reddit.com                -> type web, kind thread, strip " : r/…"
.reddit.com  ^/r/OCPoetry  -> tags [poetry, community]
---------------------------------------------------------------
r/OCPoetry  =>  web / thread / poetry + community
```

Specificity is **computed** (context, host length, path presence, path length),
not file order, so adding a profile can never accidentally shadow an existing
one by being inserted above it.

**Matching is by URL and/or CONTEXT.** `match.host` (leading dot = domain or any
subdomain), `match.path_regex`, `match.url_regex`, plus an optional `context`
token from the unified registry. Contexts are *available, never required* — most
sites will never warrant one, so a profile may use either or both.

Validation is strict-but-quiet: a profile naming a type/kind/status the registry
doesn't define is reported on stderr and that FIELD is dropped; the rest of the
profile still applies. `link_profiles.py check` validates all of them.

**Creating one is a pick, not a regex-typing exercise.** The grab form's
"Defaults" row shows which layers matched (so you can see *why* the form looks
the way it does) and offers `suggest`-derived options — "the whole site
(reddit.com)" vs "just this section (reddit.com/r/OCPoetry)" — seeded from the
form's current values. So "make this the default for this site" is one action.

### The type registry — types are data now

`MEDIA_TYPES` was a hardcoded Python list. It is now derived from
**`INIDATA/media_types.json`**, one row per type carrying `label` /`statuses` /
`status_labels` / `subtypes` / `hub` / `fs` / `recommend`. A built-in seed in
`media_catalog.py` keeps the catalog working if the file is corrupt (degraded,
not broken). Consequences:

- **Adding a section is a menu action** — `+ New section` in `open media`, or
  `＋ New type…` inside the grab form, both → `clog media-add-type`. Same for
  within-type kinds (`media-add-subtype`) and tags (`media-add-tag`, which wraps
  `quotes.py vocab-add` so a new tag is *pickable*, not just stored).
- **`web` is labelled "Links"** everywhere Jamie sees it, while the storage key
  stays `web` — ids like `web:mariame_kaba_wikipedia` keep working. Label and
  key are deliberately allowed to differ.
- **Statuses are per-type.** `web` has `queued`/`done`/**`reference`** and no
  `active` (you don't half-read a bookmark). `_valid_status` clamps per type.
- The hub's type list, the CLI's `--status` choices, and the pickers all read
  the registry, so none of them need editing when a type is added.

### What got folded in

| Source | Result |
|---|---|
| 26 vault bullets | 21 imported (5 were already catalogued); note name → tags; a Google-search URL rejected as not-a-resource |
| 10 `page-grabs/*.md` (Web Clipper) | indexed with **`archive_path`** → the full-text archive is finally queryable; it had **no index at all** |
| 23 numbered link slots | imported, slot category → tag (`cook`→cooking, `rot`→brainrot); titles fetched live (YouTube oEmbed + `<title>`) |
| existing 44 catalog items | **subtype backfill** — 36 YouTube channels, Wikipedia promoted to `reference` |
| `SavedLinks/*.txt` | deleted (dead since March, already migrated) |
| GenreLinks.ini (859 music URLs) | **left alone** — playback chain, not a reading library |

`web` 9 → 36, `youtube` 35 → 52. One-time migration:
`Scripts/MediaCatalog/link_migrate.py` (dry-run default, `--commit`, dedupes by
URL identity + title-id, `--no-fetch` for offline).

The numbered slots stay the **launcher** (`cook 3` must open instantly, no
Python on the hot path), but `set <cat> <n>` now also mirrors into the catalog
via `link_ingest.py` — fire-and-forget, same shape as `_SpotCatalogIngest`, so a
slow catalog write never costs the slot. **`SLOT_TAGS` is duplicated in
`link_migrate.py` and `link_ingest.py` and the two must agree** — one imported
the old slots, the other files new ones, and a drift would scatter the same
shelf across two tag names.

**Browsing:** a type with kinds gets a "By kind" lens above its flat list
(`media-query --subtype`, `--group subtype`), which is what keeps a growing
Links section legible. **Perf gotcha:** a Miller previews the highlighted row's
children eagerly, so the grab form caches the type registry and tag vocabulary
per capture (`_GLResetCaches`) — uncached it spawned a Python process per arrow
key, exactly the trap `MillerTags.ahk` documents.

## Media enrichment — "keep the best data possible"

**Full design: [[MEDIA_ENRICHMENT_SYSTEM]]** (locked 2026-06-19; build deferred). One-line: for every book (and later every media type), auto-collect the richest data possible — description, genre tags, cover, series, bibliographic — behind a Spotify-style confirmation step, on a generic provider seam so TV/movies reuse the core. Recommendations are out of scope for now; we bank genres/series/subjects as the future engine's fuel. See that doc for the API research, source strategy, data-model, genre-conforming, series handling, confirmation UX, and the deferred Anna's-Archive-download + StoryGraph-scraper seams.

## Episodes — the TV hierarchy, and "make favorite" (built 2026-08-25)

**Voice: "make favorite" / "make unfavorite".** Marks the episode playing right
now as a favorite. Nothing needs to be set up first — the show, season and
episode records are created on the spot if they don't exist.

### What was missing

`tv.json` held 27 flat show records and nothing below them. Seasons and episodes
existed only as a **denormalized blob** copied onto each caught item —
screenshots and song finds both carry `show: {title, season, episode}` plus a
`show_id`. So every surface knew *which episode it caught something in*, and
nothing could answer **"what else happened in S2E6"**, because there was no
episode to point at.

### The hierarchy is the one music already uses

`subtype` + `parent` were already in the item schema, proven by
artist → album → song. TV is the same shape:

```
tv:adventure_time                  (show)
  tv:adventure_time:s02            subtype season,  parent = the show
    tv:adventure_time:s02e06       subtype episode, parent = the season
```

Adding `season`/`episode` to the `tv` and `anime` subtype vocabularies was a
row in `media_types.json` — no Python. `media-query --subtype episode` and
`--group subtype` worked immediately.

### Ids are NUMERIC, never title-derived

`make_id` slugs the **title alone**, which is right for a top-level item and
wrong for a child — and it fails silently: the second item with a taken title is
refused with `exists:` and exit 1. Nothing is corrupted, nothing is logged, the
item just never arrives.

Music has been living with this. **`music:lover` belongs to Alice Phoebe Lou**,
so no other artist can ever have a track or album by that name — across 6,833
albums sharing one namespace with 3,172 artists. Episodes would be worse: every
show has a "Season 1" and "Pilot" repeats forever.

So seasons and episodes are keyed `s02e06`, not by title. That survives an
episode being renamed in Plex, zero-pads so plain string sort gives broadcast
order, and represents season 0 (Plex's specials) — a falsy value an
`if season:` guard would silently drop.

For everything else the fix is **additive and collision-only**: a child whose
flat id is taken gets rescoped to `<parent_id>:<slug>` (`music:taylor_swift:lover`).
Rewriting existing ids would orphan every `parent` reference pointing at them —
a migration, not a repair — so nothing that works today changes.

Pinned in `Scripts/codebase_tools/tests/test_episode_hierarchy.py`.

### Favorite is a TAG, not a field

`favorites` already existed in the shared vocabulary (scoped to `web`); it was
widened to every media type. That means `media-query --tag favorites`, umbrella
expansion, and every Miller tag picker worked on favorited episodes the day this
landed, with **no new query plumbing** — the same reasoning the catalog already
applies to ambient tracks: *a new grouping needs no new section, just a tag*.

**Unfavoriting removes the tag, never the record** — screenshots and song finds
point at that episode and would be orphaned.

### Asking Plex, not the window title

Every other catch surface identifies a show by parsing the Chrome tab title,
which is right there — the title survives fullscreen when the URL does not
(see `show_titles.py`). But the tab title is
`Adventure Time - S2 · E5 - Google Chrome`. It carries the show and the numbers
and **not the episode title**, so "Storytelling" is simply not recoverable from
it. Plex knows, so `plexlib.now_playing()` asks `/status/sessions`.

**`/status/sessions` reports ACTIVE PLAYBACK ONLY**, and the likeliest moment to
say "make favorite" is right after an episode ends — when the session list is
empty but the title still names what she was watching. So the title parse is the
fallback, feeding `plexlib.find_episode(show, season, episode)`, which walks
show → season → episode by **`index`, never by title** (a season is "Season 2"
in one library and "Series 2" in another).

Shows are matched on a normalized title because the tab title carries a year the
library record usually doesn't (`INVINCIBLE (2021)` vs `Invincible`). Plex's own
title filter is a prefix match and misses exactly that case.

A show that isn't in the Plex library at all still gets a full record from the
title parse, minus the episode name — the same call `ensure_show` already makes
about a show it can't resolve. Invincible is in the song finds and not in the
library; it backfilled as `tv:invincible:s04e01` and is a real record.

**Plex identity:** both ids are stored. `plex_rating_key` (601) is a row number
in *this* server's database and stops meaning anything if the library is
rebuilt; `plex_guid` (`plex://episode/5d9c0b7d…`) is global and survives that.
Guid is identity, rating key is the fast local handle.

### Generic by construction

`now_playing.py` is a **resolver registry** — one function per source, all
returning the same flat dict. Plex is implemented; YouTube is an explicit stub
carrying the shape it must return (a channel plays the part of the show, a video
the part of the episode, with no season between them — which the ensure-chain
already handles, since season is skipped when `None`). Adding a source is a
resolver there, **not** a second voice command.

### Where it lives

| Layer | File |
|---|---|
| Hierarchy primitives (`season_id`/`episode_id`/`child_id`/`ensure_item`/`add_item_tag`) | `Scripts/MediaCatalog/media_catalog.py` |
| **Record layer** — chain, links, reverse query | `Scripts/MediaCatalog/episodes.py` |
| **Resolver layer** — who's playing + favorite | `Scripts/MediaCatalog/now_playing.py` |
| Plex session + library lookup | `Scripts/acquire/plexlib.py` (`now_playing`, `find_episode`) |
| AHK entry points | `Helpers/MediaFavorites.ahk` (`MakeFavorite`, `UnmakeFavorite`, `ShowNowPlaying`) |
| Browse (show → episodes → catches) | `Helpers/MediaHubMenu.ahk` |
| Tests | `Scripts/codebase_tools/tests/test_episode_hierarchy.py` (25) |

## Everything plugged into the episode (built 2026-08-25)

The record layer and the "what's playing" layer are **two modules on purpose**,
and the split is about who calls which:

| | needs |
|---|---|
| `now_playing.py` — resolvers, one per source | the network, a foreground window |
| `episodes.py` — records, links, reverse query | nothing; pure catalog work |

`make favorite` needs both. The **screenshot and song-find capture paths need
only the second** — and they are hot paths (Print Screen; a song caught
mid-episode), so they must never pay for an HTTP round-trip. Everything in
`episodes.py` works offline from (show, season, episode), which is exactly what
those two already parse.

### The link

Both stores now carry `episode_id` beside their existing `show_id`
(`episodes.py link --commit` backfilled 11 rows). That pointer is the whole
point: the numbers in a show blob can *describe* an episode but can't be
**joined on**.

```
episodes.py caught tv:steven_universe_future:s01e15
  screenshot  shot00001  …_Jubilant-Longhorn.png
  screenshot  shot00002  …_Ironic-Kakapo.png
  screenshot  shot00003  …_Gargantuan-Dugong.png
```

A reader falls back to **deriving** the id from the show blob when the field is
absent, so rows written before the link — or a store restored from an old
backup — still join correctly.

`episode_id` is stored **beside** the show block, never inside it. It isn't a
property of the show, and `backfill-shows` decides whether a row changed by
comparing the freshly-parsed block against the stored one — an extra key in
there made every row compare unequal and a dry run reported eight rewrites that
weren't real.

### The placeholder rule

A capture creates the episode from **numbers alone** (`S02E05`) — no Plex call
on the Print Screen path. `episodes.py enrich --commit` fills the real titles in
afterwards, and any path that already knows the title upgrades the placeholder
**in passing**. Without that upgrade, whichever surface touched an episode first
would own its title forever, so a screenshot taken before the first favorite
would leave the record reading "S02E05" permanently. The upgrade is guarded on
the placeholder pattern, so a title Jamie edits by hand is never overwritten.

### Browsing it

`open media` → TV drills **show → episodes → what was caught in them**, with
catch counts on each episode row ("3 shots", "1 song"). Episode counts come from
**one** `episodes.py shows` call per node-list build, never one per row — a
per-row check would recreate the per-item-disk-pass shape that made the art
gallery take 48 seconds.

`media-query` gained **`--top-level`** (parent-less items only), and the TV/Anime
listing uses it. This is load-bearing rather than cosmetic: once `tv` became
hierarchical, the plain listing put three rows called "Season 1" and a row
called "S04E01" in among the shows alphabetically, which reads as a corrupt
catalog rather than a tree.

**Still on the denormalized blobs, deliberately:** `SongFindsMenu.ahk` and
`ScreenshotsMenu.ahk` group by parsed episode LABELS. They work, their sort
logic is non-trivial, and the records they'd point at are already reachable from
the hub — so repointing them is a real refactor with real regression risk and no
new capability. Worth doing when one of them is next opened for other reasons.

## `watch <show>` in the Plex desktop app (built 2026-09-07, rebuilt 09-08)

The web player was **software-transcoding Mushi-Shi's video AND audio** — 10-bit
4:4:4 h264 that Chrome cannot decode, plus FLAC pushed through `aac_mf` — and
the mangled audio was the "demon singing" bug. The desktop app renders through
mpv and **direct-plays both**, so the fix was to route the show there:

```
INIDATA/VoiceChoices/watch_shows.json → "Mushy": { "service": "plex_desktop" }
```

`_WatchIsDesktopService` makes `WatchShow` skip Chrome entirely for such a
service — otherwise the show opens in both and the two clients fight over the
same server session.

### Why it is UI automation

Three link routes were tested and **none exists**: no `plex://` protocol is
registered (the `plex://` strings in the bundle are server-side music-station
URIs); a Plex web URL on the command line does nothing warm *or* cold; and
`/clients` is empty, so the server cannot proxy a `playMedia` to it. The app has
no addressable pages.

### The route

**search → the result's TITLE → the show page's hero `Resume`** → answer
`Resume Playback` if it appears → `Expand Player` → `f` → assert it is playing.

Two other routes were built first and rejected **on measurement**, by planting a
known resume point and reading back where playback actually began:

| Route | Planted | Started at | |
|---|---|---|---|
| Continue Watching card's hover overlay | — | correct | but see below |
| The search **row's** own ▶ button | 5:00 | **0:00** | restarts the episode |
| The show page's hero `Resume` | 8:00 | **8:09** | correct, and stable |

The Continue Watching row resumes correctly but its play control **does not exist
until the card is hovered**, which makes the step a race — and the sidebar is a
drawer that expands on pointer proximity and paints *over* the row, so the
automation ended up holding it open with its own cursor and covering the very
first card. The show page's Resume has none of that: always present, no hover.

### What this app does to break automation

Every one of these produced the *same* visible symptom — "the click did nothing":

| Trap | Reality |
|---|---|
| **A stuck right mouse button** | Windows gives its owner a mouse capture and swallows every synthetic click machine-wide. Hover keeps working, so the target highlights and shows tooltips. Cost hours. `UiaReleaseStuckMouseButtons()` now runs inside the click guard. |
| Chromium's a11y tree is **lazy** | The first query after the window comes forward returns only the native title bar. `ElementFromPoint` wakes it. |
| The sidebar is a **hover-expanding drawer over content** | Rows lay out from x=96; the drawer swells to ~270 and covers them. UIA reports covered elements' rects perfectly happily. |
| Player controls **auto-hide** | `Pause` / `Minimize Player` / `Exit Full Screen` leave the tree entirely. A pointer *delta* raises them; moving to a spot the cursor already occupies emits no event at all. |
| The breadcrumb link's Name goes **stale** | Still read `TV Shows adtam • steelflicks` after navigating to hers, while its child text nodes read `TV Shows` / `JAMIE-PC`. |
| The sidebar toggle's Name is **inverted** | `Collapse` when collapsed, `Expand` when open — it names the state, not the action. |
| A card's play control is named by **watch state** | `Resume` part-watched, `Play` for a fresh episode. |
| A left-over **search panel** covers the page | And survives a bare `Escape`, because focus is not in the box. One failed run silently broke the next. |
| `f` is a **toggle** | Sending it when already fullscreen drops back to a window. |
| A stray click on the **video** toggles pause | Which yields a fullscreen, right-episode, direct-playing, perfectly *paused* show — the one failure a screenshot cannot tell from success. |

That last one is why the flow ends with a real postcondition: `plexlib playstate`
asks the server, and `_WatchPlexEnsurePlaying` un-pauses if needed.

`ExitPlexVideo` is the counterpart — *"normally the home button is not
visible"*: raise the controls, leave fullscreen, pause, minimize the player (it
covers the whole app, and Home is *underneath* it), then Home.

### The lesson worth keeping

`UiaClickElement` logging `success:1` means *a click was sent at those pixels* —
nothing about what was under them. Hand-written click sequences with no
post-conditions reported success while navigating into **another server's
library**. Every step now asserts what it was supposed to cause and logs what it
saw instead, which is what turned each of the traps above from a round trip into
a single log line. The recorder already emits the right primitive for this in
its DRAFT CLICK-THROUGH block (`UiaClickThenWaitFor`); ignoring it was the
original mistake. See `~/.claude/skills/ahk-functions/references/uia-clicking-debugging.md`.
