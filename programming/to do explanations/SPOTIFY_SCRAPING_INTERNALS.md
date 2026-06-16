---
tags: [programming, music, spotify, scraping, reference, gotchas]
---

# Spotify scraping internals — reference

Authoritative technical reference for the Spotify /related + /discography UIA
scraping pipeline. Read this BEFORE touching `Scripts/MediaCatalog/prefetch.py`,
`Helpers/SpotifyAutoBatch.ahk`, or `Helpers/SpotifyPrefetchReview.ahk`.

Companion docs:
- High-level system: [[MEDIA_SYSTEM]]
- Shipping plan: [[MUSIC_CATALOG_BACKLOG]]
- Sub-skill: `~/.claude/skills/ahk-functions/SKILL.md` (links to this doc under
  "Completion-log + unified media catalog")

## What can — and CAN'T — be done via HTTP

Most Spotify data is hydrated client-side. The server-rendered HTML at
`https://open.spotify.com/artist/<id>` contains:

| Field | Available via HTTP | Notes |
|---|---|---|
| Artist name | ✅ `<meta property="og:title">` | What `verify-spotify-url` uses |
| Monthly listeners | ✅ `og:description` | Format: `"Artist · 31.3M monthly listeners."` |
| Schema.org JSON-LD | ✅ Embedded `<script type="application/ld+json">` | Only contains `name`/`description`/`url`/`potentialAction`. No tracks, no albums, no related artists. |
| Album list / discography | ❌ Client-side hydrated | Must use Chrome + UIA dump |
| Related artists | ❌ Client-side hydrated | Must use Chrome + UIA dump |

So Chrome stays the only path for the actual data we need. The HTTP gain is
**name verification** — bulletproof URL-points-to-the-right-artist check at
the cost of one ~500ms request.

## The /related UIA dump format

Each related-artist card on `<url>/related` emits this exact sequence in the
UIA dump (Daft Punk → MGMT card example):

```
[group] Name="MGMT" [3390,601 229x283]
[button] Name="MGMT Artist" [3390,601 229x283]
[button] Name="Play MGMT" [3550,769 49x49]
[link] Name="MGMT" [3402,825 51x23]
[group] AID="card-title-spotify:artist:0SwO7SWeDHJijQ3XNS7xEE-5" [3402,825 51x23]
[text] Name="Artist" [3402,851 36x21]
```

**Crucial invariant:** the `[link]` and `[group] AID` lines for the same card
emit **identical** `[X,Y WxH]` coordinates. The `AID` always has the format
`card-title-spotify:artist:<22-char-base62-id>-<position>`. The position
suffix `-N` is a 0-based card index Spotify added to disambiguate duplicate
cards on the same page.

### The parser bug we shipped (and fixed) on 2026-06-10

`_related_from_dump` originally used FIFO pairing: walk lines, queue every
AID, pop on the next button-name line. This is **fundamentally broken**
because every card's button-name line comes BEFORE its AID line in the dump.
The first card's button got skipped (empty queue), then every subsequent
button popped the previous card's queued AID — every (name, spotify_id) pair
was off-by-one.

**Symptoms** (each verified during the audit):
- 44 placeholders titled "Tab search" with valid Spotify IDs (the search-tab
  button name got paired with whatever AID had been queued)
- 207 duplicate-title groups (same artist name across many different IDs —
  e.g. "Father John Misty" ×4, "MGMT" ×5, "St. Vincent" ×5)
- Lana Del Rey's real ID stored under the "MGMT" placeholder, Taylor Swift's
  ID stored under "St. Vincent", etc.
- 100% disagreement when old vs new parser were run on the same Daft Punk
  dump (40 pairs differed, 0 matched)

**Fix shape** ([prefetch.py:79](file:///C:/Users/jamie/Desktop/Important/AutoHotkey/Scripts/MediaCatalog/prefetch.py#L79)):
1. Index every `[link] Name="..." [X,Y WxH]` line by its coordinate tuple
2. For every `[group] AID="card-title-spotify:artist:<id>(-<N>)" [X,Y WxH]`
   line, look up the link by coord — that's the name. O(1) per card, no FIFO
   drift.
3. Validate `<id>` is exactly 22 chars base62
4. Strip trailing `" Artist"` / `", Artist"` suffix from the link name
5. UI-label denylist (`tab search`, `home`, `search`, `your library`, etc.)
   as a defense-in-depth safety net

## Wrong-URL detection — three layers

The catalog can end up storing the wrong Spotify URL for an artist. This is
**not theoretical** — 7 of 65 saved artists had wrong URLs as of 2026-06-10.
The wrong-URL paths:

1. **MusicBrainz URL-relation rot.** MB's `url-rel` table for an artist links
   to a Spotify URL. These are user-contributed and frequently wrong or
   vandalized. `lastfm-import` resolves via `mb_url_rel` by default. This is
   how Elliott Smith ended up at `6r2WWT5OW7Xwev9B0qfAW0` (Newfound Interest
   in Connecticut), Built To Spill at Sparklehorse's URL, Tchaikovsky at
   Saint-Saëns's URL, etc.
2. **Old broken placeholder cross-check.** Before the parser fix, the
   `resolve_artists` placeholder cross-check would prefer a placeholder URL
   over the MB URL. With wrong (name → spotify_id) pairs in placeholders,
   this poisoned downstream resolution.

The defense is layered:

### Layer 1 — HTTP og:title pre-flight (`verify-spotify-url`)
- New CLI: `media_ingest.py verify-spotify-url --url <URL> --expected-name <X>`
- Implemented at [media_ingest.py:cmd_verify_spotify_url](file:///C:/Users/jamie/Desktop/Important/AutoHotkey/Scripts/MediaCatalog/media_ingest.py)
- Fetches `og:title` from `open.spotify.com/artist/<id>`, fuzzy-matches via
  token overlap ≥0.5
- ~500ms normal, retries once after 3s on transient errors
- AHK backfill calls this BEFORE Chrome navigation — wrong-URL artists are
  skipped in <1s instead of burning a ~30s Chrome navigation

### Layer 2 — Dump window-title guard (inside `save-placeholder-artists`)
- Parses `Spotify – Artists Fans of <ARTIST> also like - Google Chrome` from
  the dump's `[window]` line and fuzzy-matches against the expected source
- Returns `WRONG_PAGE` envelope if mismatch
- Catches Chrome navigation diverging from the URL we requested (redirects,
  session weirdness)

### Layer 3 — Discography Jaccard guard (pre-existing in `prefetch_artist`)
- After scraping `<url>/discography/album`, compare the page artist name
  against the queue's expected name. Reject if Jaccard < 0.5.
- This is what catches Simon & Garfunkel landing on "Simon & Garfunkels"

**All three combined** is bulletproof. Layer 1 should catch any wrong URL
before Chrome work; layers 2 and 3 are defense in depth.

## Missing-URL auto-heal (from neighbor scrapes)

Separate from wrong-URL *detection*: a saved artist can have an **empty**
Spotify URL (never resolved — e.g. added by name only). Such an artist is
invisible to the backfill (`_AutoBatchListSavedArtists` filters out rows with
no URL) and can never be scraped.

`save-placeholder-artists` auto-heals these. Every `/related` card carries the
neighbor's name **and** Spotify id+url straight from Spotify's own grid. When a
scraped neighbor's slug-name (`mc.make_id`) matches a saved (non-placeholder)
artist whose `spotify_id` is empty, the URL/id is written onto that saved row
right then — so an unbackfillable artist resolves the moment they appear as
*someone else's* neighbor (bedroom-pop artists surface fast: Alice Phoebe Lou
and Leith Ross both healed off Haley Heynderickx's page on 2026-06-10). On the
next backfill pass they're no longer blocked, so their own `related` fills in.

Two safety rules:
- **Only ever FILLS an empty id, never overwrites.** Wrong-URL repair stays
  verify-spotify-url's job; this path can't corrupt a good URL.
- **Runs BEFORE the spotify_id match** and retires any stale duplicate
  placeholder. The pre-heal code spawned a duplicate (e.g.
  `music:alice_phoebe_lou_<id8>`) carrying the real id when the saved row had
  no URL; without the early heal, `by_sid[sid]` would hit that duplicate and
  shadow the real row forever. The heal removes the duplicate and points the
  id index at the real row. Reported as `urls_healed` in the save envelope and
  the backfill summary.

### De-dup guard (real + placeholder sharing one spotify_id)

A separate pre-pass at the top of `save-placeholder-artists` runs on every
call: it groups all rows by `spotify_id` and, for any id held by BOTH a real
(non-placeholder) artist and one or more placeholders, collapses onto the real
row — merging the placeholders' `related_from` back-refs, then deleting the
placeholders (`duplicates_removed` in the envelope + backfill summary). This
fires when a wrong-URL artist's URL is corrected to an id a placeholder already
held (The Gaslight Anthem on 2026-06-10: real `music:the_gaslight_anthem` +
`music:the_gaslight_anthem_7If8DXZN`, same id → collapsed to one). It only ever
removes PLACEHOLDERS, and only when a real row exists to keep — two colliding
*real* artists are left alone (ratings / personal_status need human judgment).

## Spotify broken-state signatures

**Page-ready REQUIRES the loaded page to name the artist we navigated to.**
Two conditions must both hold in the SAME dump:
1. The window/document title reads `Fans of <ARTIST> also like` where
   `<ARTIST>` matches the requested artist (fuzzy: lowercase + strip
   punctuation + token-overlap ≥ 0.5, via `_ArtistNameMatches`).
2. At least one `card-title-spotify:artist:` AID rendered (the grid).

The title is the anti-stale-page guard. Spotify's SPA leaves the PREVIOUS
artist's grid on screen while it navigates, so a dump taken too early captures
the old cards. But the title only becomes `Fans of <THIS ARTIST> also like`
once THIS artist's related data has loaded — and the title and the cards come
from the **same fetch**, so a matching title proves the cards are this
artist's. Until the page finishes, the title is the URL fallback
(`open.spotify.com/artist/<id>/related`), which fails the match and keeps the
poll waiting.

This was a real poisoning bug. In the 2026-06-10 25-run, Elliott Smith's page
was slow; the dump captured Dr. Dog's (the previous artist) still-on-screen
grid, and Dr. Dog's neighbors got saved under Elliott Smith. A title-vs-
filename audit of the saved dumps caught it (24/25 matched; Elliott Smith →
"Dr. Dog"). The card-only readiness check that shipped earlier the same day did
NOT catch it, because Dr. Dog's cards were genuinely present — just the wrong
artist's. The title gate is what closes this hole.

The **left library sidebar** throws `Name="Something went wrong"` constantly,
even on perfectly good `/related` pages (it's a separate data fetch that fails
independently of the main content). It is therefore **NOT a brokenness
signal** — it is ignored. A page showing it transiently just keeps polling for
the grid, then (if the grid never comes) falls through to the never-rendered
path. This was a real false-skip bug: pre-2026-06-10 the loop checked
"Something went wrong" *before* the grid and bailed on good pages (Alvvays was
the first caught example).

Only genuine WHOLE-PAGE failures short-circuit the poll — the ones where the
grid will never render no matter how long we wait:

| Signature in dump | Reason | What it means |
|---|---|---|
| `[text] Name="upstream request timeout"` | Envoy edge proxy 504 | Page is mostly black with this lone text; Spotify's backend is down |
| `[text] Name="Couldn't find that ..."` | Page-not-found banner | Navigated successfully but the entity doesn't exist (or its fetch 404'd) |
| (neither of the above + no `card-title-spotify:artist:` AID after ~18s) | Silent failure | Page never rendered — likely a 504 without the text, or just slow |

`Name="Something went wrong"` is deliberately absent from this table — see
above.

**Abort logic** ([SpotifyAutoBatch.ahk](file:///C:/Users/jamie/Desktop/Important/AutoHotkey/Helpers/SpotifyAutoBatch.ahk)):
`consecutiveBroken` counter increments on any of the three broken shapes
above (504 / not-found / never-rendered). 3 in a row → `backfill_abort` log
line + run halts cleanly. Counter resets on the first successful render.
Avoids burning 25 × 18s = ~7 min on a fully-broken Spotify session.

## Generic scrape readiness — `WaitForScrapeReady`

The page-ready poll + classification is NOT bespoke to `/related` anymore. It
lives in `Helpers\ScrapingFunctions.ahk` and serves every Chrome+UIA scrape
path through one spec-driven function:

- `ClassifyScrapeDump(dumpText, spec)` — pure: ready | broken | loading.
- `WaitForScrapeReady(spec)` — navigates (optional), then dumps + classifies
  on a retry budget until terminal (ready / broken / timeout / aborted). The
  *per-run* policy (abort-after-3-consecutive-broken, batch counters) stays in
  the caller — it spans items, not one page.

A `spec` is a Map: `readySignature` (success substring), `titleRegex` +
`expectedName` + `matchFn` (the anti-stale-page identity gate), and
`brokenSignatures` (whole-page 504/404). Site-specific specs are built by
`_SpotifyScrapeSpecRelated` / `_SpotifyScrapeSpecDiscography` in
`SpotifyAddFunctions.ahk`:

| Page | readySignature | identity title |
|---|---|---|
| `/related` | `card-title-spotify:artist:` | `Fans of <X> also like` |
| `/discography/album` | `spotify:album:` | `<X> - Discography` |

**This fixed the album path too.** The prefetch path (`SpotifyPrefetchReview.ahk`)
scraped BOTH `/discography/album` and `/related` via a fixed `Sleep`, so a slow
album page could dump the previous artist's albums — the same stale-page class
as the Elliott Smith bug, latent on albums. Both now go through
`WaitForScrapeReady`. When the related page isn't ready in the prefetch path,
it writes an EMPTY related dump (0 related) rather than feeding a stale page to
the parser.

## Decoupled scrape pipeline — producer / consumer spool

`SmartScrapeAuto` / `BatchPrefetchAndSaveAuto` used to do, per artist, all on
the FOREGROUND (holding Chrome the whole time):

```
Chrome nav /discography -> scroll-dump -> Chrome nav /related -> dump
THEN ~4s prefetch-artist (parse + enrich) + add-from-uia + save-placeholders
```

Only the Chrome half needs Chrome. The Python half is ~half the per-artist
wall-clock (more for deep catalogs — it scales with album count; the Chrome
scrape doesn't) and held Chrome hostage for no reason. Measured 2026-06-13:
coupled flow was 18–23s/artist with Chrome locked the entire time.

**The split.** It's now a producer/consumer pipeline:

- **PRODUCER** — `BatchPrefetchAndSaveAuto` (foreground, `Helpers/SpotifyAutoBatch.ahk`).
  Per artist: `_ScrapeOneToSpool` does ONLY the Chrome work (nav /discography +
  scroll-accumulate + nav /related + dump), drops the raw dumps + a `meta.json`
  into the scrape spool, marks the queue entry **`scraped`**, and moves on. The
  WRONG_PAGE guard (detected from the page title DURING the dump) stays here —
  no spool entry exists yet, so recovery (`recover-url` + `queue-fix-url`) is a
  foreground step.
- **CONSUMER** — `media_ingest.py process-spool --watch` (detached background
  process, `Scripts/MediaCatalog/spool_consumer.py`). Drains the spool: runs
  `prefetch.prefetch_artist()` directly, then the SAME save/skip/recover
  decision the old AHK auto-review loop made (HIGH/MED → `add-from-uia` +
  `save-placeholder-artists`; LOW → skip; `wrong_artist` album-overlap → swap to
  `recovery_url` + re-queue, or fail). Marks the queue **`done`/`skipped`/
  `prefetch_failed`**. Launched once at batch start; its single-consumer lock
  makes a double-launch a no-op; it idle-exits ~30s after the spool drains.

Net (measured 2026-06-13, batch of 3): Chrome held **12s/artist** vs 18–23s
before (~40–45% less); the final save completed 10s after Chrome was freed. The
two halves are comparable in size, so the pipeline is well balanced and
foreground throughput roughly doubles. The end-of-batch modal is async ("Chrome
is free — saving in the background"); the batch manifest / "Review this batch"
triage fill in live as the consumer saves (it appends A/F lines to the same
`%TEMP%/spotify_last_batch.tsv` the triage reader parses).

**Spool layout** (`E:/Media/Music/DataBackend/scrape_spool/`):

```
building/<entry>/   producer STAGES a complete entry here (consumer ignores it)
pending/<entry>/    producer atomically DirMove's the finished entry in;
                    contains artist.txt, extra_NN.txt, related.txt, meta.json
working/<entry>/    consumer atomically renames pending/<e> -> here to CLAIM it
error/<entry>/      an unreadable/threw entry, kept for diagnosis
results.jsonl       one line per processed entry (producer's end-modal snapshot)
```

**Two non-obvious bugs the first live run exposed** (both fixed, both worth
remembering for any AHK→Python file handoff):

1. **BOM.** AHK's `FileAppend(text, path, "UTF-8")` writes a UTF-8 **BOM**.
   Python's `json.load` chokes on a leading BOM, so the consumer skipped every
   `meta.json`. Fix: producer writes `"UTF-8-RAW"` (no BOM); consumer reads
   `utf-8-sig` defensively. **Any meta/handoff file AHK writes for Python to
   parse must be UTF-8-RAW.**
2. **TOCTOU claim race.** When the producer built the entry directly in
   `pending/` and wrote `meta.json` last, the consumer (polling) claimed it —
   `os.rename` of the dir into `working/` — the instant `meta.json` appeared,
   i.e. BEFORE the producer's final `FileExist(meta)` success-check ran. The
   check then race-failed and the producer wrongly marked a perfectly-good
   entry `prefetch_failed`. Fix: producer builds in `building/` (unwatched) and
   atomically `DirMove`s into `pending/`; success = the move succeeded, never a
   post-hoc existence check the consumer can invalidate.

**Queue safety.** Two processes now write `scrape_queue.json` (producer marks
`scraped`, consumer marks `done`/`failed`). `write_scrape_queue` is atomic (no
torn files) but NOT lost-update-safe, so every queue mutation now holds
`media_catalog.scrape_queue_lock()` (a lockfile sibling of `catalog_lock`).
Catalog saves stay single-writer (only the consumer) and go through
`add-from-uia` as a subprocess so they inherit the central `catalog_lock`.

## Wrong-URL guard at the SOURCE — `lastfm-import`

The og:title check is no longer only a scrape-time pre-flight. The shared
`spotify_scraper.verify_artist_url(url, expected_name)` (which
`media_ingest verify-spotify-url` is now a thin wrapper over) is called inside
`lastfm_data.resolve_artists`: every `mb_url_rel` / `mb_search` resolution is
verified before it's cached or queued. A confirmed mismatch is dropped (the
`mb_url_rel` case falls through to the `mb_search` fallback), so MB-relation
rot is caught BEFORE a wrong URL enters the catalog — not just when the
backfill later trips over it. Net errors don't block (the backfill re-checks).

## Same-name collision guard — play-weighted album identity

`verify-spotify-url` / Jaccard / `WRONG_PAGE` are all **name-based**, so they
are blind to a same-name collision: the wrong "Cream" (an EDM act) genuinely
*is* named Cream, so every name check passes. On 2026-06-10 a 10-artist pull
saved the wrong **Cream** (EDM, 0 albums) and wrong **Love** (J-pop idols)
at HIGH — because confidence scores **web-metadata completeness** (bio / MB /
Wikipedia / related), never identity.

The discriminator is your own listening: **does the scraped discography match
the albums you've actually scrobbled under that name?** Implemented in
`lastfm_data.album_identity_overlap(conn, name, scraped_titles)`:

- **Play-weighted, NOT album-count.** Box sets / compilations / remaster
  naming make album-count overlap undercount even for the RIGHT artist
  (Daniel May: 20% by count, **88% by plays**). Weight by scrobble plays and
  the gap is unambiguous — wrong = 0%, right = 88–99%.
- Verdicts: `confirmed` (overlap ≥ 0.30) · `wrong_artist` (real album
  scrobbles but overlap < 0.30) · `unknown` (< 10 album-scrobble plays — can't
  judge; track-only listening or classical-via-performer → never blocks).
- The verdict is now ONE weighted signal in the unified evaluator
  (`prefetch.evaluate_artist`, see [[SPOTIFY_CONFIDENCE_REWORK]]), not a hard
  block. `wrong_artist` is a strong negative that **combines** with dampeners:
  a deep discography (≥20) or a classical genre shrinks it (overlap undercounts
  for those), a sparse one (≤3) amplifies it, and an og:title match can offset
  it. So the wrong "Cream" (sparse, no dampener) still rejects, while a real
  deep/classical artist that merely fails title-overlap survives to review.

**Gated on trust — this is the critical part.** It runs ONLY when the queue
resolved the URL via `mb_url_rel` / `mb_search` (MusicBrainz *guessed* it).
Spotify-direct URLs (`placeholder_ref*` / `cached`) came straight off Spotify's
own `/related` grid and are trusted, so they're NEVER second-guessed — a
neighbor you upgraded but never listened to stays HIGH (and also returns
`unknown` anyway, since you have no scrobbles for it). `resolved_via` is
threaded queue-pop → AHK `_PrefetchOneQueued` → `prefetch-artist --resolved-via`
→ `prefetch_artist`. Cached entries keep their original `resolved_via`, so a
cached wrong URL still gets guarded on re-import.

## Window title format reference

- `/related` loaded correctly: `Spotify – Artists Fans of <ARTIST> also like - Google Chrome`
- `/discography/album` loaded correctly: `<ARTIST> | Spotify - Google Chrome`
  (or similar — verify when relevant)
- Album page: `<ALBUM> - Album by <ARTIST> | Spotify - Google Chrome`
- Loading / failed: tab title falls back to the URL (`open.spotify.com/artist/<id>/related - Google Chrome`)

## NTFS Alternate Data Streams gotcha

Catalog IDs contain `:` (e.g. `music:father_john_misty`). Using one
directly in a filename creates an NTFS Alternate Data Stream — the file
appears as `backfill_related_music` (everything before the colon) in
directory listings, with the rest stored in a hidden stream. `FileCopy`
writes to the stream silently; `open()` reads it back; but the file is
invisible to debugging tools and gets overwritten on every iteration.

**Rule:** anywhere a catalog ID becomes a filename, sanitize it first:

```ahk
safeId := StrReplace(artId, ":", "_")
dumpCopy := A_Temp "\backfill_related_" safeId ".txt"
```

## Rate limiting

Spotify's HTTP edge will rate-limit aggressive callers. During the 2026-06-10
audit, 65 sequential `verify-spotify-url` requests hit transient 429s near
the end. Mitigation:
- `verify-spotify-url` retries once after 3s on any transient error
- Naturally throttled when called from the backfill (one request per ~17s
  Chrome cycle)
- For bulk audits, add a deliberate 1s sleep between requests

## Files touched in the 2026-06-10 fix session

- `Scripts/MediaCatalog/prefetch.py` — `_related_from_dump` rewritten (coord-pair)
- `Scripts/MediaCatalog/media_ingest.py` — new `cmd_verify_spotify_url`,
  wrong-page guard inside `cmd_save_placeholder_artists`
- `Helpers/SpotifyAutoBatch.ahk` — `BackfillPlaceholdersForAllSavedArtists`
  gained: maxCount param, HTTP pre-flight via `verify-spotify-url`, adaptive
  page-ready poll, broken-state short-circuit, abort-after-3, ADS-safe
  filename
