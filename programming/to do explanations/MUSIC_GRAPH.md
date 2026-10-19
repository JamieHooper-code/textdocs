---
tags: [programming, design-doc, music, recommendations, lastfm, spotify]
created: 2026-10-13
updated: 2026-10-13
status: v2 built 2026-10-13 — artist graph + embedding + album sound + her listening + nightly refresh
related: ["[[MUSIC_FOLDERS]]", "[[MEDIA_SYSTEM]]", "[[SPECIAL_TODAY]]", "[[SPOTIFY_SCRAPING_INTERNALS]]"]
---

# Music graph — how music relates to music, and to Jamie

This is THE store for "what is this music like" and "what does Jamie like".
Anything new that needs that reads and extends it. Don't start a parallel one.
Engine: `AutoHotkey/Scripts/MediaCatalog/music_graph.py`. Data:
`C:/Users/jamie/MusicData/graph/` (`music_graph.db`, `artist_embed.npz`,
`nightly_status.json`). Every music data path is named once, in
`music_paths.py`. See "Where the data lives" below.

## What uses it

| Surface | What it does |
|---|---|
| Folders page **Similar** | Another album by an artist already in the folder. The artist is weighted by √(their items in the folder). The album is the one whose sound and era best fit the folder. |
| Folders page **Different** + **close ↔ far** slider | An album by an artist NOT in the folder who sits near what is. Close gives the obvious neighbours (Father John Misty, Bon Iver, Haley Heynderickx for Comfort). Far gives related-but-unexpected ones (Daniel Norgren, Julie Byrne, Flatsound). Each pick says why: "near Iron & Wine, Fleet Foxes, The Tallest Man On Earth". |
| Any album/artist card's ··· → **More like this** | That one record is the whole seed (`pick_like`, `POST /api/more-like`). Same ranking as Different, with the album chosen by THAT record's sound. It follows the close ↔ far slider. Pink Moon → Fairport Convention close, Gordon Lightfoot further out. |
| Voice **similar N / different N** (`music_dial.py`) | One 0–100 dial. 95+ = same artist; 40–95 = this graph (pick_like / pick_different, window slid by N, re-rolled if the pick shares under 0.12 tag cosine with the seed); under 40 = the best-of lists, picking by tag distance from the seed. Re-said within a minute = same seed. Details in [[MUSIC_FOLDERS]]. |
| Recommended smart folder → **Near what you love** | Albums by artists she barely knows (≤20 plays) who sit near the artists she *stayed with* this past year. Stable for the day. |

## The store

| Table | Holds |
|---|---|
| `artists` | key, name, when similar / tags / albums were last fetched |
| `edges` | `(src, dst, source, weight 0..1)`. Sources: **lastfm** (`artist.getSimilar` match, the backbone), **spotify** (scraped related[] lists, rank → weight), **coplay** (artists she played in the same session; cosine over ~8k sessions) |
| `tags` | artist tags: Last.fm toptags (count/100) + catalog tags |
| `albums` | an artist's albums by global Last.fm plays |
| `album_tags` / `album_meta` | per-album Last.fm toptags, fetched as albums come up |
| `listens` | HER per-album listening: plays, **runs5**, **runs1** (below) |
| `picks` | every Similar/Different press (folder, artist, album, distance, why) |
| `artist_embed.npz` | 128-d vector per node: ~28k artists + ~15.7k album nodes (`r:<artist>\|<title>`) |

**Nothing expires.** After 60 days a fetch is *allowed* to run again. Its answer
replaces the old data only if it succeeded and came back non-empty. (v1 deleted
first, so a Last.fm outage would have wiped an artist's links. Fixed.)

## Her listening — the 5-song rule

Jamie's rule (2026-10-13): finishing an album is NOT a signal, because she can
love an album and not finish it. What counts is staying with it.

- A **run** = consecutive scrobbles from one album, with no gap over 30 minutes.
- **runs5** = runs of 5+ songs. She stayed with it, which means she liked it.
- **runs1** = ONE song, followed within the session by a *different* album she
  stayed with for 3+ songs. She tried it and moved on. (v1 counted any one-song
  run, and shuffle made that 28k false "moved on"s. With the 3+ follow-up rule it's 3.4k.)

How they're used, gently:

- Album taste ×1.3 if she has stayed with it.
- ×0.6 if she keeps moving on from it, but only when she has played it 3 or
  fewer times and never stayed with it.
- Artist taste ×1.2 / ×0.7 by the same logic.

`picks` + `music_graph.py picks` reports each press as stayed / some / moved on
/ not played. That's for tuning by hand, not an automatic penalty.

## How "near" is scored

- **sim(seed, candidate)** = weighted mean of whichever parts are known:
  - edge 0.55: `1 − Π(1 − source_weight × w)`, with lastfm 0.9, spotify 0.75, coplay 0.6.
  - tag 0.2: TF-IDF cosine.
  - embedding 0.25: cosine of artist vectors.
- **folder score** = weighted mean of sim to every seed × (1 + 0.15 per extra
  seed it's near, sim ≥ 0.2) × her artist taste. That breadth bonus is the
  "intersection": near three folder artists beats very near one.
- **Embedding**: a spectral embedding of the whole degree-normalised edge graph
  (scipy `eigsh`, ~3s). It finds two-hop relations, e.g. Vashti Bunyan for
  Comfort with no direct link to anything in it. It also adds the 200 artists
  nearest the folder's centre as candidates.
- **Distance**: the top 150 are ranked, and one press samples a window of 25
  (weighted by score³). The slider slides that window from the top (0) to the
  bottom (1).

## Albums

- **Which albums count**: when MusicBrainz knows the discography
  (release_dates.db), it is the whitelist. Studio albums and EPs are allowed,
  plus compilations for Similar. Live albums, bootlegs and singles are not.
  Last.fm's album list alone mixes in singles ("Ragged Wood"), samplers and
  dated bootlegs, and no play-count threshold separates them.
- **The folder's sound**: a TF-IDF centroid of its albums' tags (falling back
  to artist tags), plus the mean and spread of their years.
- **Album choice**: √plays × (0.5 + 2·tag cosine to the folder) × an era fit
  (0.6–1.0) × her album taste.

## Nightly — `MusicGraphNightly` (04:30 + logon+20m, hidden pythonw)

`register_music_graph_task.ps1` (`-Remove` / `-RunNow`). Each step's result goes
to `nightly_status.json`:

1. Scrobble delta sync (`lastfm_data.py lastfm-sync`, NO_WINDOW child).
2. `build-personal`: co-play edges + listens.
3. `import-spotify`.
4. Warm: folder artists + top 1000 played (2h budget, skips anything not due),
   then album tags for every folder album.
5. Lists (`music_lists.py sync_all`, see below).
6. `build-embed`.
7. Covers (`music_covers.py`): seed priorities, then fetch. The budget is 2 h
   while more than 2,000 are queued and 30 min after that. Then evict if over
   the cap.
8. Mirror: `music.json` and `music_folders.json` are copied back into the
   `E:\Media` git repo.

## Search — "music <anything>" (added 2026-10-14)

Voice: **"music <words>"** (generic store → `MusicSearch` in
`Helpers/MusicFolders.ahk`). You can also type into the search box at the top
of the folders page. Either way the page opens at `/folder/search~<phrase>`.
The engine is `Scripts/MediaCatalog/music_search.py`, and the page assembly is
`music_smart.search_result`.

**1. Understanding a phrase (a plan).**
- Artist names in the phrase are found first, without any model ("radiohead"
  in "radiohead spanish").
- Then a **local** model via Ollama reads it: `SEARCH_MODEL`, which is
  `gemma4:26b-a4b-it-qat` (switched from gemma4:12b on 2026-10-14, see the model test
  below). It must never be Claude, because that costs Jamie money per call.
- The model only *proposes*. A tag only counts if our data has it; an artist
  only counts if it's in the graph. The 9B model invented "Vetusta Morales".
- Plans are cached per phrase in `search_plans`. A plan fixed by hand
  (`edited=1`) is never overwritten.
- A region the model adds ("europe" for Spanish) is dropped when a country or
  language is already given. Otherwise it let every British band through.
- Place words used as tags ("spanish", "japanese") are weighted down to 0.25,
  because the place factor carries them. At full weight, "spanish progressive"
  returned Spanish pop.

**2. Ranking.**
- **Artists:** tag cosine over Last.fm and MusicBrainz tags.
- **"Like X":** graph neighbours, plus *sound* (the seed's own tags and the
  embedding). Sound is what lets a seed cross into another place.
- **Place factor:** taken from `artist_place` (MusicBrainz country, see
  `music_places.py`), or from tags like "british" or "ethio-jazz". A known wrong
  country scores 0.
- **Lists:** each list's tag centroid × how specific the list is, + a bonus when
  the list's name says it, × the share of its artists from the right places.
- **Albums:** artist score × album tags × acclaim. Compilations only count if
  critics listed them (Ethiopiques yes, Kitaro's "Best of Silk Road" no).

**The page** has these sections:
- the "Read as" chips;
- a 0–10 rating row;
- Lists;
- Top albums (at most 2 per artist);
- Artists;
- You already play;
- New to you;
- Deeper cuts.

Results are cached for 10 minutes, so the page's 4-second refresh doesn't re-run
the search.

**History and ratings** (Jamie: "save a history of everything I do"):
- `searches`: each phrase's runs, plan, summary, rating and note.
- `activity`: every page open and every POST. The server wraps all POST routes
  once in `_log_posts`, so new routes are logged automatically.
- The **Searches** smart folder shows Recent, Best rated, and **To fix
  together (4 or under)**.
- In chat, `py music_search.py review` lists those low-rated searches, with each
  one's plan, ready to fix. Fixing means either `fix "<phrase>" '<plan json>'`
  for one phrase, or changing the scoring for all of them.

**Gaps:**
- A *thin* search (fewer than 2 lists, or fewer than 15 artists near the best
  match) records the tags it wanted in `search_gaps`.
- The server then scrapes Last.fm's top albums for those tags in the
  background, so the next run finds more. The nightly run does this too.
- "ethiopian jazz" did this on its first run.

**Model test, 2026-10-14** (RTX 5060 Ti 16 GB, 32 GB RAM; five phrases):
- All of qwen3.5:9b, gemma4:12b and qwen2.5:14b read "japanese environmental"
  as kankyō ongaku, and "spanish" as a language.
- gemma4:12b also named real example artists (Mulatu Astatke, Hiroshi
  Yoshimura, Haruomi Hosono).
- Speed: about 3 s warm, 15–30 s cold (gemma4:26b: about 2 s warm, about 35 s cold).
- **VRAM, measured:** gemma4:26b loaded = 15.5 of 16.3 GB in use. Idle draw was
  unchanged (10.5 W, 38 °C), so an idle model causes no wear, but it crowds out
  everything else on the GPU. 95% of page visits never search, so **the model
  loads on demand** and unloads 5 min after the last search (`KEEP_ALIVE`). It
  was pre-loaded on every page open until Jamie asked, 2026-10-14.
- **New phrases:** the server returns a loading view at once
  (`start_understanding` + `_loading_view`); the page draws a progress bar and
  re-asks every second until the results arrive. A cold first search took
  about 30 s.
- **Round 2, the same day, on 9 phrases including harder ones** ("sad brazilian guitar",
  "turkish psych", "like pink moon but from mali", "70s ethiopian funk"):
  - `gemma4:26b-a4b-it-qat` (16 GB, MoE with 4B active) was **faster warm (about 2.0 s vs
    3.1 s)** and named real deep cuts: Satoshi Ashikawa and Takashi Kokubo for
    kankyo ongaku; Erkin Koray, 3 Hurel and Altin Gun; Mahmoud Ahmed. Its cold
    start is about 37 s.
  - `qwen3.5:35b-a3b` (22 GB) was about as good: it found Triana for Spanish
    prog, but it read "radiohead spanish" as Radiohead's own Spanish songs (GB),
    and it is bigger.
  - **The default is now gemma4:26b.** The 12B invented more names ("Ali Farka
    Tuareg", "Chico Buleta").
  - Test harness: the five original phrases plus the four above; compare by
    eye, then re-run `music_search.py run <phrase> --fresh --model <tag>`.

**Data added for search:**
- `music_places.py` (artist country, area and MusicBrainz tags). Hiroshi
  Yoshimura comes back JP with the tag "kankyō ongaku".
- 60 more Last.fm genre lists (`SEARCH_TAGS`: Japanese, Spanish-language and
  Latin, more African, instrumental, prog).
- The album-tag backlog (`music_lists.py album-tags`). Albums Last.fm doesn't
  know (error 6) are stamped as tried, not retried.
- All of these run in the nightly `search_data` step.

```
py Scripts/MediaCatalog/music_search.py run "radiohead spanish" [--fresh] [--model gemma4:26b-a4b-it-qat]
py Scripts/MediaCatalog/music_search.py history | review | gaps | fill-gaps
py Scripts/MediaCatalog/music_places.py stats | show "Mulatu Astatke" | run --minutes 30
```

## Where the data lives (moved 2026-10-14)

Everything the folders page reads on open is now under
**`C:\Users\jamie\MusicData`** on the SSD (`music_paths.MUSIC_DATA_DIR`):
- `catalog/`: `music.json`, `music.cache.sqlite`, `music_folders.json`,
  `music_folders_cache.json`
- `lastfm/`: scrobbles
- `release_dates/`
- `recommendations/`
- `graph/`
- `covers/`

**Why it moved.** E: is a USB spinning drive (G-DRIVE) that Windows powers down
after 20 idle minutes. The first open of the morning waited 5–10 s for the
server, then sat on an empty page for another 5–10 s while the drive woke up.
With the drive awake, the same page builds in 0.6 s.

**To move it again** (for example to the coming 8 TB internal drive): copy the
folder, then change `MUSIC_DATA_DIR` in `music_paths.py` or set the
`MUSIC_DATA_DIR` environment variable. Nothing else names these paths.

**What happened to the E: copies.** The old E: DataBackend folders were renamed
to `*.moved-to-C` (lastfm, release_dates, recommendations, graph). Delete them
once nothing has complained.

**The git mirror.** `E:\Media\catalog\music.json` and `music_folders.json`
stay, because `E:\Media` is the private media-catalog git repo. The nightly
`mirror` step refreshes them, so their history keeps going. Nothing reads
them.

**What stays on E:** bestever, discogs, topalbums, toptracks, prefetch_cache
and scrape_spool. None of these are on the page's path.

## Saved album covers — `music_covers.py` (added 2026-10-14)

Before this, every open re-downloaded every cover. Cover Art Archive covers
also bounce through archive.org, which is slow.

**What is stored:**
- `covers/big/<ab>/<key>.jpg`: the largest version the source has, shrunk to at
  most **2,000 px** and never enlarged. Her main screen is 2560×1440, so a
  full-height "now playing" cover is about 1,440 px; 2,000 leaves headroom.
  JPEG q90, about 0.5–1 MB.
- `covers/small/<ab>/<key>.jpg`: 320 px, about 25 KB, for cards.
- `covers/covers.db`: the index.
- `<key>` = sha1 of the source url, first 20 hex characters.

**Biggest sources, measured:**
- Spotify: `ab67616d000082c1<hash>` is the upload, 1,400–2,000 px (`b273` is
  640 px).
- Last.fm: dropping the `/300x300/` segment gives the original, 500–3,000 px.
- Cover Art Archive: `front` is the original scan, 1,400–4,000 px; `front-1200`
  is the fallback.
- YouTube: `maxresdefault`.

**Priority:**
- 4+: albums played 3 or more times, ranked by plays, and folder entries.
- 2+: covers that appeared on her pages on **two different days**. A cover
  seen once isn't saved.
- 0–1: acclaimed list albums, ranked by acclaim.

**Cap:** 50 GB (her call). Over the cap, covers that were never played are
removed first: lowest priority, then longest unseen. At about 0.6 MB per
cover, 50 GB is roughly 80,000 albums.

**Page wiring (`music_folders_server.py`):**
- `api_view` returns a copy of each view with saved covers pointed at
  `/cover/<key>.jpg`, served with a one-year immutable cache. Every remote
  cover is noted as seen.
- `?s=big` returns the 2,000 px version, which is what a player screen should
  use.
- A background thread runs every 60 s: it flushes the sightings and saves up to
  40 covers of priority 2 or higher. The long tail is left to the nightly job.

**First fill (2026-10-14):** 10,266 queued (849 played albums, 23 folder
entries, 9,318 list albums). They were saved at about 110 a minute.

```
py Scripts/MediaCatalog/music_covers.py stats | seed | evict
py Scripts/MediaCatalog/music_covers.py fetch --minutes 60 --workers 4
py Scripts/MediaCatalog/music_covers.py show <cover url>
```

## Commands

```
py Scripts/MediaCatalog/music_graph.py stats
py Scripts/MediaCatalog/music_graph.py similar "Fleet Foxes"
py Scripts/MediaCatalog/music_graph.py folder <fid> --mode rank | different [--distance 0.5] | similar
py Scripts/MediaCatalog/music_graph.py near-you-love
py Scripts/MediaCatalog/music_graph.py taste "Fleet Foxes"     # her runs per album
py Scripts/MediaCatalog/music_graph.py picks                   # how presses went
py Scripts/MediaCatalog/music_graph.py nightly | warm | build-personal | build-embed | import-spotify
```

Page routes: `POST /api/random-similar`, `/api/random-different`
(`distance`) in `music_folders_server.py`. **Restart the folders server after
editing music_graph.py**, because it holds the module in memory (stale-code trap).

## Best-of lists and acclaim (added 2026-10-13)

**Why.** Jamie's point: the graph only spreads outward from what she plays, so
even "far" stays in her neighbourhood. Going somewhere genuinely new needs a
sense of QUALITY that is independent of her listening. That's consensus: an
album on many best-of lists, near the top, is strong whatever its genre. Engine:
`Scripts/MediaCatalog/music_lists.py`, which uses the same SQLite file as the graph.

**Sources, and what was ruled out:**

| Source | What | Status |
|---|---|---|
| MusicBrainz release-group **series** | ~270 critic lists as ordered data: Rolling Stone 500 (all 4 editions), Pitchfork, Paste genre lists, Uncut, NME, SPIN, Metal Hammer, Resident Advisor, Grammy genre categories… One API call per list returns artists + dates. CC0. | used (`mb`) |
| BestEverAlbums | overall top 1,000 (100 pages × 10) + top 200 per decade 1950s–2020s; resumable (a list cut off by the nightly time budget is stored and left due) | used (`bea`), one page per 6s; robots.txt allows general crawlers |
| Last.fm `tag.getTopAlbums` | top 100 albums for ~130 genres (Ethio-jazz, American primitivism, zeuhl, city pop…) with covers | used (`lastfm`); POPULARITY not acclaim |
| BEA genre charts | user-made charts only, with no genre filter on the main chart and a site search that won't return charts | later: find charts through album pages ("appears in N charts") |
| Rate Your Music | ToS forbids scraping | no |
| Acclaimed Music | Cloudflare challenge | no |
| r/indieheads, /mu/ | Jamie: later | parked |
| Spotify Web API (direct links) | needs a Premium developer account | no; links come from the search-then-learn path |

**Tables:** `lists` (source, name, kind, family, genre, size, coherence,
specificity), `list_items` (rank, artist, title, MusicBrainz release-group id
(`rgid`), year, cover), `lalbums` (per album: acclaim, n_lists, cover,
cover_ok), and `mb_series_seen` (every series the search found, accepted or not).

**List specificity, Jamie's caveat.** The Rolling Stone 500 spans every
genre, so it says two albums are both GOOD, not that they're SIMILAR.

- `specificity` = how tightly a list's artists' Last.fm tags cluster, measured
  above a random-artists baseline.
- It gets a 0.5 floor when the list's name names a genre.
- Every list feeds acclaim. Only lists with specificity ≥ 0.35 feed album-to-album
  edges, weighted by their specificity.

**Acclaim.**

- Position score = 1 − 0.5·(rank−1)/size.
- Editions of one list are one *family* (e.g. RS 2003/2012/2020/2023) and count
  once, at their best position.
- Weights: critics 1.0, BEA 0.7, Last.fm 0.
- `acclaim = log1p(raw)/log1p(p99)`.

Acclaim then boosts:

- Artist scores: ×(1 + 0.3·best album acclaim).
- Album choice: ×(1 + acclaim).
- Near what you love.

**Albums as nodes.** The embedding now holds album nodes (`r:<artist>|<title>`):

- Albums on specific lists, plus every album she has stayed with.
- Each joins its artist (`member`).
- List neighbours within 15 ranks join each other (`colist`).

35% of Different / More like this presses then land on an album node directly
("a record near this folder's records"). Albums near the folder's centre also
lift album choice.

**Covers.** Last.fm images from the tag lists. Otherwise Cover Art Archive by
`rgid`, HEAD-checked first (`cover_ok`), so a card never shows a broken image.
The page also swaps any image that fails to load for the coloured placeholder.

**Folders** (all smart, all seeded by the date so they're stable for a day).
Strangers, Canon and Haven't heard in years also get rows on **Daily**.

| Folder | Rule |
|---|---|
| **Strangers** | Two lanes, alternated: critics' acclaim (acclaim > 0.15) and the top 15 of Last.fm genre lists for genres far from her taste vector (weight ≤ 0.05). Artist played ≤3 times, genre familiarity ≤ 0.35, at most 2 per genre. Ranked together, acclaim always won and the row was all BRAT / Thriller / Appetite. Alternating brings in MPB, highlife, shibuya-kei, rocksteady and outsider music. This is the "very far" that the graph can't be. |
| **Canon you've missed** | highest acclaim, album never played |
| **Haven't heard in years** | albums she stayed with (or 15+ plays), not played in 3+ years |
| **Top of each genre** | Last.fm genre tops minus anything played, genres she knows LEAST first |
| **Best-of lists** | a card per list (critics' all-time / genre / by year and decade / BEA / Last.fm); each opens as its own folder `smart_list~<list id>`, played albums marked |

**Nightly** gained a `lists` step before `embed`:

- Discover new MusicBrainz series, then refetch lists older than 90 days (20 min).
- Refresh Last.fm genre tops (every 30 days).
- Crawl BEA (30 min). It's page-bound at 6s a page, and a list cut off part-way is saved and stays due.
- Tag list artists, then recompute specificity and acclaim, then check covers.

Tags and covers **adapt to the backlog**: an hour each while it's big (more
than 1,000 untagged artists or 2,000 unchecked covers), 15–20 minutes once
caught up. Both run a few at a time: 3 for Last.fm (it allows about 5 a
second), 4 for Cover Art Archive (no hard limit). Their failures are left for
the next run, never recorded as "no art". The task's time cap is 6h to fit a
catch-up night.

```
py Scripts/MediaCatalog/music_lists.py stats | lists [--source mb] | show <list id>
py Scripts/MediaCatalog/music_lists.py strangers | canon | revisit | genres
py Scripts/MediaCatalog/music_lists.py sync [--mb] [--lastfm] [--bea] | specificity | acclaim | covers
```

### Measured on the first full sync (2026-10-13)

- **Lists:** 265 MusicBrainz lists (~15.5k placements), 133 Last.fm genre tops,
  and the BEA all-time + decade charts. 18.9k albums, 9.3k with acclaim.
- **Specificity worked as hoped.** The Rolling Stone 500 editions score
  0.08–0.11, 1001 Albums 0.03 and BEA all-time 0.04. "30 Best Underground Hip
  Hop" scores 0.94, Grammy Rap 0.83, RS Hair Metal 0.80, Pitchfork Shoegaze
  0.70. 231 of 402 lists count as specific.
- **Acclaim** was first scaled to the 99th percentile, which made the top ~90
  albums all 1.0. It's now scaled to the max: Lemonade, To Pimp a Butterfly,
  good kid m.A.A.d city at the top. The critics' lists skew to 2010s/2020s
  year-ends; the BEA all-time chart is what balances that.
- **Embedding:** 64 dims with eigenvalue-weighted columns made 21% of nodes
  look >0.9 alike, so album-direct picks were near-random ("Midnite Vultures
  near Pink Moon"). 128 dims at eigenvalue^0.5 brings that to 0.4%. Now Our
  Endless Numbered Days → Laura Marling, Karen Dalton, Vashti Bunyan's
  Lookaftering, and Weaving a Basket → Still Corners, The Radio Dept., Cranes.
- **Speed:** opening a SQLite connection (and running the schema script) per
  lookup made a 120-card folder take 16–24s. One connection per thread, with
  the schema run once per process (`ensure_schema`), brings every view under
  0.4s (Daily 1.5s).
- **Known gaps:**
  - Last.fm genre tops are popularity, so Strangers' far lane includes some
    merely-popular records.
  - A few artist-centred MusicBrainz series ("Interpol: Top 10 albums") get
    past the name filter.
  - Strangers needs artist tags; the nightly fetches ~20 min of them per night
    until all ~10k list artists are tagged.

## Next ideas

- Album-level embedding (albums as nodes alongside artists), so "Different"
  can land on an album directly rather than artist-then-album.
- Seed from a single album ("more like this record") on any card's ··· menu.
- A radio queue from a folder (N picks in a row, close then drifting out).
- Feed `picks` outcomes back as a per-folder nudge once there are enough of them.
