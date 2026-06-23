---
tags: [programming, media, books, enrichment, design-doc, apis, series, recommendations, voice-commands]
status: design — locked, implementation deferred
created: 2026-06-19
related: ["[[MEDIA_SYSTEM]]", "[[SPOTIFY_SCRAPING_INTERNALS]]", "[[SPOTIFY_CONFIDENCE_REWORK]]", "[[COMPLETION_LOG]]", "[[MUSIC_CATALOG_BACKLOG]]"]
---

# Media Enrichment System — design doc

**One line:** For every book Jamie adds (and later every media type), automatically collect the **richest data possible** — description, genre tags, cover, series, bibliographic — behind a **Spotify-style confirmation step**, built on a **generic provider seam** so TV / movies / etc. reuse the same core.

> Status: **design locked, build deferred.** Jamie greenlit the architecture on 2026-06-19; implementation will come back later. This document is the authoritative spec to build from. The only code shipped so far is the Kindle title/author parse fix (see [§12](#12-already-shipped)).

---

## 1. Goals & non-goals

**Goals**
1. **Best data per book, automatically.** When a book enters the system ("log read" / "add read"), pull description, genres, cover, series, page count, publisher, year, language, and stable IDs (ISBN / OpenLibrary / Google Books) — from the best available source, with graceful fallback.
2. **Confirmation Jamie can watch.** A confirmation modal (the book analog of the Spotify review modal) showing rich info + cover so she can confirm "yes, this is the same book." Single-book adds **always** show it; backfills auto-accept unambiguous matches.
3. **Conform genres exactly like the music system** — route every source's raw genre/subject strings through the **same** `canon_tag` + umbrella taxonomy that conforms last.fm tags.
4. **Clean series handling** — model series as a structured `{name, position}`, render the hub grouped + sorted by position.
5. **Genericize aggressively.** Books are the first consumer; the confirm / merge / conform / cover machinery and the provider seam are written to serve movies, TV, anime, etc. next. Jamie: *"how many of my media libraries will need so many similar systems… make generic anything that makes sense to."*

**Non-goals (for now)**
- **No recommendation engine yet.** Explicitly out of scope. We collect the **inputs** a future engine computes over (genres + series + subjects), not a recommendations list. (See [§6](#6-the-recommendations-reality) for why a per-book "similar" list can't simply be fetched.)
- **No auto-download yet.** Anna's Archive download automation is a *designed-in deferred seam* ([§11](#11-deferred-seams)), not v1.
- **No Chrome scraping for books.** Unlike Spotify, books have real HTTP APIs — the book path is pure HTTP. (Scraping returns only for the deferred StoryGraph recommendations seam.)

---

## 2. Guiding principle — mirror Spotify, but simpler

This system was designed **after studying the Spotify scraping pipeline** ([[SPOTIFY_SCRAPING_INTERNALS]]), at Jamie's explicit request. The Spotify system's proven shape is:

```
prefetch (gather all sources) → cache → confirmation modal → merge (keep curated) → write
```

with three properties worth lifting wholesale:
- **Failure-isolated enrichment.** Each source is fetched independently and stored as a `{source: blob}` bundle, so one source timing out never sinks the save (`enrichment.artist_summary`).
- **Merge never clobbers curated fields.** `_merge_into` keeps Jamie's tags / description / rating / status and refuses to overwrite real data with empty fetch results.
- **Confidence-gating.** Strong matches can auto-save; weak ones go to manual review.

**What books DON'T need** (because they have real APIs, where Spotify needed Chrome+UIA): no scraper, no producer/consumer spool, no stale-page guards, no UIA coordinate-pairing, no play-weighted identity overlap. The book path is dramatically simpler — a few HTTP calls.

| Spotify concept | Book equivalent |
|---|---|
| `_ConfirmationModalGui` review modal | reuse directly, **+ a cover image** |
| `{source: blob}` failure-isolated enrichment | genericize (provider results bundle) |
| `_merge_into` keep-curated precedence | genericize |
| confidence-gated auto-save vs review | genericize (auto-accept unambiguous on backfill) |
| shared `canon_tag` + umbrella taxonomy | **already unified** — books + music share it today |
| Chrome/UIA scraper, spool, identity overlap | **not needed** (real APIs) |
| Spotify `/related` behavioral graph | **no book equivalent exists** — see [§6](#6-the-recommendations-reality) |

---

## 3. API & source research (what each actually returns)

Probed live on 2026-06-19 against the test book ***The Galaxy, and the Ground Within*** by Becky Chambers (ISBN `9781473647671`, Wayfarers #4).

### 3.1 OpenLibrary — the workhorse (≈ MusicBrainz)
No auth, lenient rate limits. Two-hop: edition (by ISBN) → work (subjects + description).

- **Edition by ISBN** (`/isbn/<isbn>.json`): `publishers` `['Hodder & Stoughton']`, `publish_date` `2021`, `number_of_pages` `400`, `languages` `[eng]`, `isbn_13`, `series` (freeform), `lc_classifications`, `covers`, link to `works`.
- **Work** (`/works/<id>.json`): full **description paragraph**; `subjects` (15): *Science Fiction, American literature, Interstellar travel, Fiction, Extrasolar planets, Space colonies, Extraterrestrial beings, [French dupes], Space Opera, LGBTQ+, General*; `subject_places` `['Gora']`, `subject_times`.
- **Cover** (`https://covers.openlibrary.org/b/isbn/<isbn>-L.jpg`): **HTTP 200** ✅. Also by OLID / cover id.
- **Candidate search** (`/search.json?title=&author=&fields=…`): returns `title, author_name, first_publish_year, cover_i, edition_count, key` per hit — exactly what the picker needs. `edition_count` is a clean ranking signal (real book = **12** editions; box-set bundle = **1**).

**Verdict: PRIMARY.** Alone it delivers description + genres + cover + bibliographic + a rankable candidate list, no key, no throttle.

### 3.2 Google Books — best blurb, needs a key (≈ Wikipedia)
`/books/v1/volumes?q=isbn:<isbn>` or free-text `q=`. Returns `title, subtitle, authors, publisher, publishedDate, pageCount, language, categories, averageRating, ratingsCount, imageLinks, industryIdentifiers, description`.

- **Rate-limited hard when unkeyed** — hit **HTTP 429** on *both* probe attempts (earlier `book-resolve-kindle` calls had already burned the anonymous quota). **Confirmed: unusable at any volume without a key.**
- A **free API key** (Google Cloud console → enable Books API) gives 1,000 req/day. Store at `~/.googlebooks_credentials.json` `{"api_key": "…"}`, mirroring `~/.lastfm_credentials.json`.

**Verdict: SECONDARY** (keyed). Best for description prose + ratings where OpenLibrary is thin.

### 3.3 Hardcover — best genres + cleanest series (≈ last.fm)
GraphQL at `https://api.hardcover.app/v1/graphql`. **Free token** (signup → account settings), **60 req/min**, 30 s query timeout. Endpoint confirmed live (`{"error":"Unable to verify token"}` without a token).

Schema (pulled from `hardcoverapp/hardcover-docs/schema.graphql`):
- **Genres:** `taggings` / `cached_tags` (JSONB) — **community-driven tags, the cleanest genre signal**, the book analog of last.fm tags. This is *the* reason Hardcover is worth the signup.
- **Series:** `book_series { position: float8, series { name }, featured, compilation }` — **structured series + numeric position**, the cleanest of any source.
- **Per book:** `description`, `image`/`image_id` (cover), `rating` (numeric), `release_date`, `pages`; ISBNs on `editions` (`isbn_10`, `isbn_13`).
- **Recommendations:** **none usable** — only a weak `ReferralType { book_id, count }`. No per-book "similar books" query.

**Verdict: OPTIONAL v1** — promote it in when you want top-tier genres + authoritative series. Needs the free token.

### 3.4 Anna's Archive — the source aligned with Jamie's actual workflow
Jamie gets most new books here, so the md5 record page is the **natural ingestion point**. The page is **server-rendered (HTTP-fetchable now)** and aggregates OpenLibrary / Goodreads metadata. From the 2026-06-19 grab of the test book's md5 page:

- **Series, clean:** *"Wayfarers, #4"* ✅
- Publisher, year, language, format, size, fiction flag
- The **full publisher blurb** + **alternate descriptions** + the **table of contents**
- The **md5 id** (the key the future download API needs)

**Verdict: FIRST-CLASS PROVIDER.** Page-parse now (series + descriptions + ids); the paid **JSON API** (auto-download + metadata) is a deferred seam ([§11](#11-deferred-seams)). Capturing the md5 now means that seam slots in without rework.

### 3.5 Sources evaluated and rejected
- **StoryGraph** — best recommendations, but **no public API** and the `/similar` page is **login-walled** (verified: anonymous fetch → **HTTP 302** to sign-in, 0 bytes). Only reachable via an **authenticated Chrome+UIA scrape** → deferred seam ([§11](#11-deferred-seams)), not v1.
- **Libby / OverDrive** — OverDrive's "Thunder" API is **partner-gated (B2B only)**; Libby has no public recs API. Dead end.
- **Wikidata / Wikipedia** — structured genre (P136) + description, but **spotty coverage** for non-famous books. Possible future augment, not v1.
- **ISBNdb / LibraryThing** — paid / key-gated with no advantage over the above. Skip.

---

## 4. Locked source strategy

| Source | Role | Now / Later | Auth |
|---|---|---|---|
| **Anna's Archive** | series + rich descriptions; the download seam | page-parse **now**; paid JSON API **later** | none now |
| **OpenLibrary** | subjects→genres, cover, bibliographic | **now** | none |
| **Google Books** | best description + ratings | **now** | free API key |
| **Hardcover** | cleanest genres + structured series | optional v1 | free token |
| **StoryGraph** | real recommendations | **deferred** authenticated scraper | — |

The mapping to the music stack is almost 1:1 — **OpenLibrary ≈ MusicBrainz** (open, structured, the workhorse), **Google Books ≈ Wikipedia** (rich prose, keyed), **Hardcover ≈ last.fm** (community tags = best genres).

---

## 5. Data model

Books stay physically in `E:\Media\Books\<Author>\library.json` (Kindle-managed; already "author → book", which is the music system's artist → album shape). Only the **write target** differs from catalog types; everything else is shared.

### 5.1 Current book record (pre-enrichment)
`id, title, author, added, source, status, tags[] ({tag, src}), isbn?, recommended_by?, url?, kindle_title?, log_name?, rating?, review?`

### 5.2 Enrichment fields added
| Field | Type | Source | Notes |
|---|---|---|---|
| `description` | str | Google Books → Anna's → OpenLibrary | paragraph blurb; precedence highest-quality first |
| `cover_url` | str | OpenLibrary / Google / Hardcover | remote URL |
| `cover_path` | str | downloaded | `E:\Media\Books\<Author>\covers\<isbn-or-id>.jpg` |
| `subjects` | [str] | OpenLibrary (raw) | **lossless** raw subjects, kept for future use |
| `tags` | [{tag, src}] | conformed from subjects/categories/taggings | `src:"auto"`; **never clobbers** `src:"manual"` |
| `series` | {name, position} | Anna's / Hardcover (OpenLibrary fallback) | see [§8](#8-series-handling) |
| `published_year` | int | OpenLibrary / Google | |
| `page_count` | int | OpenLibrary / Google | |
| `publisher` | str | OpenLibrary / Google | |
| `isbn13` / `isbn10` | str | any | back-fills the existing `isbn` |
| `language` | str | OpenLibrary / Google | |
| `google_books_id` | str | Google | stable cross-ref |
| `openlibrary_id` | str | OpenLibrary | work/edition key |
| `annas_md5` | str | Anna's page | **the download-seam key** |
| `hardcover_id` | int | Hardcover | optional |
| `rating_external` | {source, value, count} | Google / Hardcover | external avg, distinct from Jamie's `rating` |
| `enrichment` | {source: blob} | all | **raw** failure-isolated bundle (lossless audit trail) |
| `enriched_at` | iso str | — | last enrichment timestamp |

**Principle:** store both the **conformed** view (clean `tags`, `series`) *and* the **raw** view (`subjects`, full `enrichment` bundle) so nothing is lost and re-conforming later is free.

---

## 6. The recommendations reality

Jamie's instinct — "having a lot of recommendations built-in will do so much for our recommendation engine" — is right in spirit, but the honest finding is:

**No free book API exposes a per-book "similar books" graph the way Spotify's `/related` does.**
- OpenLibrary: none (lookup-only, no social graph).
- Hardcover: GraphQL exposes only `ReferralType` — no similar-books query.
- StoryGraph: has the **best** recs, but **no API** and login-walls the page.

So the strategy is a **reframe**: a recommendation engine doesn't need a pre-baked list — it needs **rich signals to compute over**. For music, Spotify hands you the behavioral answer. For books, we collect the **inputs** and a future engine derives recs:
1. **Best-in-class genre tags** (Hardcover taggings + OpenLibrary subjects).
2. **Series + position** — series-mates are the most obvious recs of all.
3. **Subject co-occurrence** — OpenLibrary `/subjects/<s>.json` lists other books sharing a subject; harvesting that *builds* a content-based "similar set" we own.
4. **Author's other works + ratings.**

This is more durable than a vendor's list (we own the graph) and is exactly "keep really good data." The *behavioral* recs layer arrives later via the StoryGraph scraper ([§11](#11-deferred-seams)).

---

## 7. Genre conforming (mirror the music system)

The taxonomy is **already unified**: `clog._canon_subjects` and `clog.canon_book_tag` re-export `media_catalog`'s `canon_tag` / `expand_tags` / `primary_umbrella` — the *same* primitives that conform last.fm music tags. Books and music share one alias/stopword/umbrella file.

**Conform pipeline (per book):**
1. Collect raw genre strings from every source (OpenLibrary `subjects`, Google `categories`, Hardcover `taggings`).
2. Each → `media_catalog.canon_tag()` (alias-normalize; returns `""` for stopwords → dropped).
3. Dedupe; store as `tags` with `src:"auto"`.
4. Umbrellas (`theory`, `identity`, genre umbrellas) derived at **query/render** time via `expand_tags`, not stored.
5. **Keep the raw `subjects`** in the record (lossless).

**One fix vs. today's `_canon_subjects`:** it picks the broad bucket (fiction / nonfiction / poetry) **per subject**, so an unrelated subject like *"American literature"* added **"nonfiction"** to a clearly-SF book → the contradictory `fiction, nonfiction` we saw. **Decide one bucket per book** (priority/majority across all subjects) instead.

`_merge_into`-style precedence: auto tags are added alongside manual tags; a manual tag is never overwritten or dropped by re-enrichment.

---

## 8. Series handling

Model series as a structured pair on the book record:

```json
"series": { "name": "Wayfarers", "position": 4 }
```

- **Sources, by trust:** Anna's Archive page (`"Wayfarers, #4"`) and Hardcover (`book_series.position`) are authoritative; OpenLibrary's freeform edition `series` field is the fallback (parse `"<Name>, #<n>"` / `"<Name> #<n>"`).
- **Rendering:** the Reading / media hub groups by `series.name` and sorts by `position` — so the four Wayfarers books cluster in reading order.
- **Relationship modeling:** `{name, position}` is sufficient for v1 (no separate series entity). If a richer series object is ever wanted, the music `parent` pattern is the proven upgrade path. Supersedes the old "series in `notes`" note in [[MEDIA_SYSTEM]].

---

## 9. Generic architecture

### 9.1 Provider seam
New module **`Scripts/MediaCatalog/media_enrich.py`** (sits beside `media_catalog.py`; importable by both `clog` and `media_ingest`, mirroring how `clog` already imports `media_catalog as _mc`).

A **provider** implements two methods; everything else (orchestration, confirm, conform, merge, cover, write) is shared and type-agnostic:

```python
class Provider:
    def search_candidates(self, title, creator, isbn="") -> list[Candidate]:
        """Cheap search → ranked candidates for the picker. Each carries
        enough to display (title, creator, year, cover_url) AND to fetch
        full data (source, ref)."""

    def fetch_full(self, candidate) -> dict:
        """Expensive per-pick fetch → enrichment fields (description,
        subjects, cover_url, series, bibliographic, ids, raw bundle)."""

# Candidate (plain dict, JSON-serializable for the AHK handoff):
#   {source, ref, title, creator, year, cover_url, isbn, edition_count,
#    subtitle}   # subtitle carries the series hint for display
```

- **`BooksProvider`** wired now — fans out to OpenLibrary + Google Books + Anna's-page, merges per [§5](#5-data-model)/[§7](#7-genre-conforming-mirror-the-music-system).
- **`TmdbProvider`** (movies / TV) drops in **later** behind the identical orchestration + confirm + merge. That's the whole point of the seam.

### 9.2 Orchestration
```
enrich(title, creator, isbn, mode):
  candidates = provider.search_candidates(...)      # ranked, deduped
  if mode == "backfill" and unambiguous(candidates):
      pick = candidates[0]                            # auto-accept
  else:
      pick = confirm_via_picker(candidates)          # cover-grid modal
  fields = provider.fetch_full(pick)                 # failure-isolated
  fields.tags = conform(fields.subjects + categories + taggings)
  record = merge_keep_curated(existing, fields)      # never clobber manual
  download_cover(record); write(record)              # book → library.json
```

**`unambiguous`** = ISBN-direct single hit, OR exactly one candidate, OR top candidate whose title **and** author both strongly token-match the query **and** clearly leads `edition_count`. Otherwise → review.

---

## 10. Confirmation UX

**Decision (Jamie, 2026-06-19):**
- **Single-book add → ALWAYS show the picker** ("just so I can watch it working").
- **Backfill → skip the picker when unambiguous**, show it otherwise.

**The picker = a cover-grid candidate chooser**, reusing **`ThumbnailGridGui`** (already displays images in a grid). It shows the top 3–6 candidates as **cover thumbnails + title / author / year / series hint**; numpad-pick the right one, or reject → fall back to manual entry. This is the book analog of the Spotify review modal, but **cover-first** (Spotify's is text-only — adding the cover is the one genuinely new GUI capability).

Covers for the grid are downloaded to a temp dir during `search_candidates` so the grid can show them locally (AHK `AddPicture` wants a local path).

---

## 11. Deferred seams (designed-in from day one)

1. **Anna's Archive paid JSON API** → auto-download new books + pull metadata. The `annas_md5` id is captured **now** so this slots in without rework. (Jamie: *"we can eventually build a scraper that will auto download… probably use their JSON API but I need a paid account first… build with the fact that this will be happening eventually."*)
2. **StoryGraph authenticated similar-scraper** → the real recommendation graph. No API exists and `/similar` is login-walled, so this is a **logged-in Chrome+UIA scrape** — the *exact* pattern the Spotify `/related` pipeline already implements ([[SPOTIFY_SCRAPING_INTERNALS]]). When built, it populates a `similar` list and finally gives books a behavioral recs layer.

Both are isolated behind the provider seam / a clearly-marked TODO, so neither blocks v1.

---

## 12. Already shipped

- **Kindle title/author parse fix** (`clog.cmd_book_resolve_kindle`, 2026-06-19). Normal Kindle titles read *"Title by Author"* with no Anna's Archive `" -- "` separator, so the whole *"…by Becky Chambers"* landed in the title and author stayed empty. Added a conservative `" by <Author>"` split (name-like, ≤5 words; the lookup confirms/corrects). Verified: title/author now split correctly **and** the online lookup back-fills ISBN + tags. Anna's `" -- "` format and plain-title matching still pass.

*(Unrelated GUI fixes shipped the same day — dark-mode buttons + Ctrl+Enter save — are documented in [[GUI_VISUAL_THEME_SYSTEM]], not here.)*

---

## 13. Planned surface (to build later)

**clog CLI (books → `library.json`):**
- `book-enrich-search --title --author [--isbn]` → JSON candidates (+ `auto` flag + local `cover_path` per candidate) for the picker.
- `book-enrich-fetch --source --ref` → full enrichment JSON for a chosen candidate.
- `book-enrich-apply --id <bookid> --enrichment-json <path>` → merge-keep-curated + write.
- `book-enrich --id <bookid> [--auto]` → full pipeline (backfill convenience: search → auto-apply if unambiguous, else emit for review).
- `book-enrich-all [--limit N]` → backfill the existing library; auto-accepts unambiguous, collects ambiguous into a review list.

**AHK integration points (`CompletionLogFunctions.ahk`):**
- `LogRead` / `AddRead` → after add, run enrich with the **always-picker** path.
- A manual **"enrich book"** action in the Reading hub (re-enrich / fix a match).
- Backfill entry point (voice + Stream Deck) for `book-enrich-all`.

**Generic GUI:** the cover-grid picker added to / reused from `ThumbnailGridGui`, taking `[{title, subtitle, year, cover_path}]` → returns the chosen index (works for any media type).

---

## 14. Build phases (when we return)

1. **Python core** — `media_enrich.py` provider seam + `BooksProvider` (OpenLibrary + Google Books keyed + Anna's-page parse) + conform (per-book bucket fix) + series + cover URL. CLI-testable.
2. **clog commands** — `book-enrich-search / fetch / apply / book-enrich`, new record fields, `_merge_into`-equivalent. CLI-tested on the sample book.
3. **Cover download + cover-grid picker** — generic, reusing `ThumbnailGridGui`; the one new GUI capability (cover in the confirm surface).
4. **Wire into flows** — `LogRead` / `AddRead` always-picker; `book-enrich-all` backfill (auto unambiguous); manual "enrich book" action.
5. **Optional v1 add-ons** — Google Books key wiring; Hardcover token + provider.
6. **Deferred** — Anna's paid-API auto-download; StoryGraph authenticated similar-scraper.

---

## 15. Decision log (2026-06-19)

- Build API-based book enrichment, no scraping, reusing the Spotify confirm/merge pattern behind a generic provider seam. ✅
- Single-add **always** shows the cover-grid picker; backfill auto-accepts unambiguous. ✅
- Download covers locally **and** keep the URL. ✅
- Auto-enrich on every new book + a backfill pass + a manual enrich action. ✅
- Recommendation engine **deferred**; collect genres/series/subjects as fuel now. ✅
- Anna's Archive = first-class provider now + designed-in paid-API auto-download seam. ✅
- StoryGraph recs = deferred authenticated Chrome+UIA scraper (no API; login-walled). ✅
- Series modeled as `{name, position}`, hub grouped + sorted. ✅
- Genres conformed through the shared `canon_tag`/umbrella taxonomy; per-book bucket fix. ✅
- Promote Hardcover to optional-v1 for best genres + cleanest series. ✅
- Set up a free Google Books API key to avoid the unkeyed 429. ✅

---

## 16. References

- [[MEDIA_SYSTEM]] — the unified catalog this enriches (schema, types, query layer, hub).
- [[SPOTIFY_SCRAPING_INTERNALS]] — the pipeline this mirrors; the source of the confirm/merge/confidence patterns and the deferred StoryGraph scraper template.
- [[SPOTIFY_CONFIDENCE_REWORK]] — the unified-evaluator confidence model, if book confidence-gating wants the same treatment.
- [[COMPLETION_LOG]] — the book/reading hub + `clog` engine these commands extend.
- [[MUSIC_CATALOG_BACKLOG]] — sibling backlog for the music side.
- Sub-skill: `~/.claude/skills/ahk-functions/SKILL.md` → "Completion-log + unified media catalog".
- Code today: `Scripts/completion_log/clog.py` (books), `Scripts/MediaCatalog/{media_catalog,media_ingest,enrichment,prefetch}.py` (catalog + music enrichment), `Helpers/Gui/{ConfirmationModalGui,ThumbnailGridGui}.ahk` (confirm + image grid).
