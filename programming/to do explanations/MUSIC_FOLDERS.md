---
tags: [programming, design-doc, music, spotify, local-viewer, ahk, caster]
created: 2026-10-10
updated: 2026-10-14
status: v1 built 2026-10-10 — Spotify only; YouTube / local albums later
related: ["[[MUSIC_GRAPH]]", "[[MEDIA_SYSTEM]]", "[[LOCAL_APP_HUB]]", "[[SPOTIFY_SCRAPING_INTERNALS]]", "[[REMINDERS_SYSTEM]]"]
---

# Music folders — our own playlist folders

Spotify's playlist folders are a client-side filing cabinet with **no API**, so
nothing could read or write them and every one had to be made by hand. Music
folders are ours instead: a store on disk, a page that draws them the way Spotify
does, and voice commands that file whatever is on screen. They tie the music
systems together in one place Jamie can browse.

## What she says

| Phrase | Does |
|---|---|
| `grab this <folder>` | file the Spotify page in front: album, artist or playlist, whichever it is |
| `grab album / artist / playlist <folder>` | the same, but the page must BE that kind (otherwise a tooltip names the right phrase) |
| `grab playing <folder>` | file the album of the song that's playing, from any Spotify page |
| `spot folders` / `open folders` | the page, top level |
| `spot <folder>` | the page, opened on that folder |
| `spot daily` / `spot recent` / `spot recommended` | the smart folders (below) |
| `similar N` / `different N` | on Spotify or the folders page: play an album N% like / unlike what's on screen (below) |

## Smart folders — Daily, Recently Played, Recommended (added 2026-10-10)

Jamie's listening data was "all over the place" (Special Today on the new tab,
an unbrowsed recommendations store, 15 years of scrobbles). The folders page is
the surface she actually uses, so these folders **fill themselves**. Engine:
`Scripts/music_folders/music_smart.py`. They are generated, never stored, and
shown in a **Made for you** row above "Your folders".

- **Daily** is the home base: several dense rows on one page. Today
  (`release_dates.special_today`: anniversaries plus artist born/formed/died),
  Recently played, Recommended, Old favorites (30+ plays, not heard in 180
  days, weighted by plays), To listen (the catalog `to_listen` tag), and From
  your folders. Every card has one small grey line saying WHY it's there. The
  random rows are **seeded by the date**, so Daily stays the same all day and
  changes tomorrow.
- **Recently Played** groups recent listens into Today, This week and Earlier.
  **Recommended** is the recommendations store minus anything scrobbled since,
  in three rows.
- **Random** (on every folder, smart ones too) plays a random album or
  playlist. A stored folder's pool INCLUDES its sub-folders. Artists are picked
  only when nothing else is there.
- **Similar / Different** (added 2026-10-13) sit beside Random on every folder.
  Similar plays another album by an artist already in the folder. Different
  plays one by an artist who isn't there but sits near what is. A close ↔ far
  slider beside it sets how far out it reaches. Recommended gained a "Near what
  you love" row. All of it is scored by the music graph: see [[MUSIC_GRAPH]].
- **Links and covers:** only about 40% of recently played albums are in
  music.json. Each row resolves catalog → release_dates.db `catalog_links` →
  Last.fm `album.getInfo` cover plus a Spotify SEARCH url (cards marked
  "search"). Clicking one plays the top result (`SpotPlayTopSearchResult`), then
  `music_smart.py learn` records the album url it landed on, so the next click is
  direct. Network answers are cached in `MusicData/catalog/music_folders_cache.json`
  by a background pool. Covers seen often or played are then **saved locally**
  and served from `/cover/<key>.jpg` (see [[MUSIC_GRAPH]] → "Saved album
  covers"). The page never waits on the network; it fills covers in
  on its 4-second poll. The cache is re-read when the file changes, because
  `learn` writes it from another process.
- **Fixed alongside:** `OpenOrNavigateChromeTab` now presses Delete before Enter.
  Chrome's inline autocomplete was turning a typed `http://127.0.0.1:8297/`
  into a deeper page from history (a deleted folder → "Not found").

## World Music and African Music (added 2026-10-14)

Two smart folders, both generated from the best-of lists (`music_lists.py`):

- **World Music** has a section per region (Africa, Latin America, Caribbean,
  Asia, Europe & the Mediterranean). Each card is one list and opens as its own
  folder. Africa's **Show all** goes to African Music.
- **African Music** is a row of albums for every African list (31 of them:
  Afrobeat, highlife, juju, mbalax, soukous, kora, desert blues, tishoumaren,
  zamrock, Ethio-jazz, gnawa, raï, morna and more, plus the South African Music
  Awards). Played albums are marked.
- 26 world genres were added to the Last.fm sync (`WORLD_TAGS`). Each was probed
  first; ones whose list was mostly the wrong music (kwaito, kizomba, afro house,
  desert rock) or near-empty (fuji, benga, palm wine) were left out.

**Genre descriptions.** `INIDATA/music_genres.json` holds one row per genre:
a short `blurb`, its world `region`, and an optional display `name` (MPB, IDM).
The blurb shows after the genre's name on Top of each genre, African Music,
World Music, the Best-of lists cards, and a genre list's own page. To add a
genre: a row there plus the tag in `LASTFM_TAGS` / `WORLD_TAGS`. The file is
re-read when it changes, so no restart.

## Favorites (added 2026-10-14)

Every name on the page has a heart right after it: albums, artists,
playlists, her folders, smart folders, lists, and the big title of the folder
she's in. **Each press adds one** (an affinity, the same bands as the art and
media library: 1 favorite, 3 loved, 5 essential). Right-click a filled heart,
or use the ··· menu, for one less; 0 removes it.

- A favourite is keyed by what it IS (`fav_key`): an album by artist + title,
  an artist by name, a folder or list by id. So the same album hearted from a
  folder, a Daily card and a best-of list is one heart.
- Stored in `music_folders.json` under `favorites`.
- The **Favorites** smart folder (first tile) draws them by kind (Albums,
  Artists, Playlists, Your folders, Smart folders & lists), highest first.
- **By voice on Spotify:** `mark favorite` / `mark unfavorite`, and `make
  favorite` / `make unfavorite` (MakeFavorite branches to MusicFavoriteThis on
  a Spotify tab, Plex episodes everywhere else). It hearts the album / artist /
  playlist page in front, or the album playing.
- **Favorite Albums folder** (`auto_favorite: true`, aliases favorite /
  favorites): `grab this favorite` files it AND hearts it. Filing gives exactly
  one heart and never adds to an existing one; filing is not a second vote.
  The singular "favorite" names this folder; "spot favorites" (plural) opens
  the all-hearts smart folder.
- **Daily** has a Favorites row 4th, ONE line only (other rows get two), so
  Daily stays mostly new music. Picked for the day, weighted by hearts.
- **Hearts are taste** (`music_graph.hearts()`): an album heart lifts that album
  ×(1 + 0.25 per heart) and its artist by half; an artist heart lifts the
  artist ×(1 + 0.15 per heart); a hearted folder lends its artists 0.2 each.
  Caps at 5 hearts. Hearted artists also seed "Near what you love". This feeds
  Similar / Different / More like this / the dial.

## Similar N / Different N (added 2026-10-14)

Jamie's voice style: `similar 7`, `different 3`. The **spoken scale is 0–10**
(changed 2026-10-14: "similar 10 is the most similar"). N ≤ 10 is multiplied by
10 inside (`music_dial.similarity_of`), so a number over 10 still works as the
old percent. They're one dial: `different N` = `similar 10−N`. The seed is the
folder she's in, the Spotify album or artist page, or (any other Spotify page)
the album playing.

**"show similar N" / "show different N"** open a whole page instead of playing
one album. The flow is `ShowSimilar` → `/api/dial-page` → a stored page
`dial~<id>`, built by `music_dial.layout`:
- **How a page is built:** every candidate artist (about 9,500: the graph
  neighbourhood plus everyone on a list) gets one closeness score: graph 0.4,
  tags 0.35 (plus the seed album's own tags when it has them), embedding 0.25.
- **The band:** the dial picks a position in that ranking from the hand-tuned
  `BAND` table (7 ≈ the closest 3%, 5 ≈ 11%, 3 ≈ 30%, 1 ≈ 65%).
- **The draw:** the page is a random draw from that band. It's seeded by the
  page id, so a refresh keeps the same page; say it again for a new one.
- **Quality:** acclaim counts for more the further out the band is.

Rows on a dial page:
- More by the artist (at 10);
- the band itself;
- A step closer / A step further;
- Artists;
- Lists at this distance.

Each album is the artist's album whose own tags best fit the seed album, when
both have album tags; otherwise their most acclaimed. Pages are rated 0–10 like
searches and are listed in the Searches folder.

**Re-roll.** Said again within a minute, it uses the ORIGINAL seed, not
wherever the last pick landed, and each press restarts the minute. So
`similar 75` eight times on eight pages is eight albums at 75 from the first
record, never a drift outward. The anchor is `dial_anchor.json` in the graph
folder.

The lanes (engine `Scripts/MediaCatalog/music_dial.py`, see [[MUSIC_GRAPH]]):

| similar | does |
|---|---|
| 95–100 | another album by the same artist |
| 40–95 | the graph: Different's engine, its close↔far window slid by N |
| 0–40 | genuinely far: an acclaimed or genre-top album whose tags sit at a target distance from the seed's |

Measured on Pink Moon: 80 → Sibylle Baier, Iron & Wine; 55 → Sharon Van Etten;
35 → Springsteen; 10 → trap; 0 → The Meters.

`py Scripts/MediaCatalog/music_dial.py lanes` prints the dial;
`... pick "<artist>" "<album>" <N>` tries one. AHK `MusicDial(mode, n, dry)`
(dry = pick, play nothing). Voice `rules/music_dial_commands.py`, scoped to
Spotify tabs and the folders page (context `INIDATA/Contexts/music_folders.json`).
Tests: `Scripts/codebase_tools/tests/test_music_dial_and_favs.py`.

`<folder>` is free dictation resolved against the store when spoken, so a folder
made on the page is sayable straight away (no Choice regeneration, no reboot). A
nested folder can also be named by its path ("to listen jazz"). A name that
doesn't exist yet **asks** before creating, because a mis-heard word would
otherwise quietly become a one-album folder. The `grab` commands are scoped to
Spotify tabs (`spotify_commands.py`), which keeps them clear of the Kindle `grab`.

## The page

`http://127.0.0.1:8297`, in the ◇ hub header as **Folders**. Folders nest with no
depth limit. The top level is folders only. Every level has **+ New folder**, a
folder has **Add a link** (paste a Spotify link), and the ··· menus do rename /
move / delete and move / remove. Folder tiles show a 2×2 cover mosaic, from their
own items first and then from inside their sub-folders. Artists are round, as on
Spotify. Clicking a card plays an album or playlist in the Spotify tab
(`SpotOpenInChrome`, which mutes the mic first), and an artist opens on their
albums (`/discography/album`). The page re-polls every 4s, so an album filed by
voice appears without a reload.

**Delete:** a nested folder's contents move UP a level, so nothing is lost. A
top-level folder has nowhere to send its items, so those are removed (its
sub-folders survive, promoted to the top), and the dialog says how many first.

## Design decisions

- **Entries are snapshots plus a catalog link.** Title, creator and cover are
  copied in at filing time, so drawing a folder never opens the 10 MB music.json,
  and something NOT in the catalog still gets a proper card. `catalog_id` records
  the join when there is one.
- **Covers for anything not in the catalog** come from the Spotify page's own
  meta tags: `og:title` ("songs - Album by Adrianne Lenker"), `og:description`,
  and `og:image`. oEmbed (`open.spotify.com/oembed?url=`) is the fallback; it has
  a cover but no artist name.
- **Filing does NOT ingest into music.json.** The full add (`AddSpotifyAlbum`)
  pops the voice-phrase and genre dialogs, which is wrong for a quick
  "grab this to listen". Could be added later as a background spool job.
- **"grab playing"** reads Spotify's "Now playing bar". Its title hyperlink's UIA
  `Value` IS the album url (verified 2026-10-10); the artist links below it carry
  `/artist/` urls and are skipped. `SpotNowPlayingAlbumUrl()` is reusable.
- **`spot <folder>`** is tier 0 of `SpotResolve`, before the mic mute (it plays
  nothing). A folder wins over a genre or slot of the same name. It shells the
  store's resolver (exit 0 = found) rather than calling into MusicFolders.ahk,
  because LinkManager is in the always-on closures and pulling LocalViewer and
  ReaderFunctions in after it would bloat every always-on script. The open goes
  through `MAINFUN.bat OpenMusicFolders`. It's skipped entirely until the store
  file exists.

## Files

- Store + rules: `AutoHotkey/Scripts/music_folders/music_folders.py` →
  `C:/Users/jamie/MusicData/catalog/music_folders.json` (moved off the
  spun-down E: drive on 2026-10-14; `music_paths.py` names it, and the nightly
  job mirrors it into the E:\Media git repo) (`{folders:[{id,name,parent,aliases}],
  entries:[{id,folder,kind,url,title,creator,cover,catalog_id,added}]}`).
  `MUSIC_FOLDERS_STORE` env var = scratch store for demos/tests.
- Page: `music_folders_server.py` (:8297, `restart` after editing it or the
  store module, because of the stale-code trap) + `music_folders_page.html`
  (read per request).
- AHK: `Helpers/MusicFolders.ahk`. Voice: `rules/spotify_commands.py`,
  `rules/spotify_global_commands.py`.

## Recommended in every folder + mood folders (both built 2026-10-18)

**Mood folders, as built** (`Scripts/music_folders/music_moods.py`, registry
`INIDATA/music_moods.json`, tests in `test_music_recs.py`):
- `music_moods.py setup` made a **Moods** folder holding Chill, Existential,
  Hype and Springtime. Each is an ordinary folder flagged `mood: <id>`, so
  `spot chill` and `grab this chill` work with no new grammar. Its own entries
  are the **Residents**.
- Below the line, music_recs draws **Familiar** (played 5+ times or stayed
  with) and **New to you** (played at most twice) from `score_candidates`,
  instead of albums / artists / lists. Each side is built 48 deep.
- **Both / Familiar / New** buttons (added 2026-10-18; Jamie: "view them
  separately when I want to"):
  - **Both** shows the dial's share of each side, with a "Show all 48" per section.
  - **Familiar** or **New** shows that whole side alone.
  - The choice is remembered per folder: in memory, plus localStorage when it
    can (the preview pane has none, and the 4 s poll used to flip it back).
- The **dial** (◀ More familiar / More new ▶, 0–10, starting at 5) splits the 32
  albums Both shows, never fewer than 4 a side. It only RE-SLICES the stored
  lists, so nothing rebuilds and nothing reshuffles.
- Score = cosine of the album's tags (TF-IDF) to the mood vector, cut by avoid
  tags (out when they are over 0.12), plus a bonus for being on a mood word's
  Last.fm list (half for a stand-in genre's list, half again when the tags show
  no fit at all).
  - The mood vector = its `want` words, + 0.8 × the residents' centroid,
    − 0.5 × the centroid of "not this mood".
  - With residents, 20% of the score is embedding closeness to them.
  - Measured: two bossa nova residents turn Chill bossa-heavy; Boards of Canada +
    Bonobo keep it downtempo. Residents really steer.
- **Mood-word lists**: `music_moods.py fetch` (and the nightly `mood_lists` step
  in music_graph) stores Last.fm tag.getTopAlbums for every `lastfm` word as
  source `mood` in music_lists. They are kept out of the Best-of lists page.
- **Two no's**: "Not this mood" is the folder's dismissal (and steers away);
  "Not for me" goes in `not_for_me` in music_recs.json and is never offered in
  ANY folder.
- **Rating**: a 0–10 and a note per mood, with history.
  `music_moods.py review` lists the ones rated 4 or under, with their "not this
  mood" picks, to tune together. Tuning a mood = editing its row in
  music_moods.json (re-read on change, no restart), then Refresh.
- **Seasonal**: a mood with `season` months gets an "In season: <name>" row in
  Daily (Springtime, Mar–May).
- Card buttons: **+ Resident** (first, so a spoken number adds) and **▶ Play**.
- `music_moods.py try <mood> [--mix N]` = what a mood would offer, without
  writing anything.
- Fallback Jamie named: if the generated side doesn't work for a mood, cultivate
  it by hand like Comfort; the residents carry it.

The Recommended section:

**How the Recommended section is built** (`Scripts/music_folders/music_recs.py`,
store `MusicData/catalog/music_recs.json`, tests `test_music_recs.py`):
- 24 albums (up to 6 of them records found directly near the folder's sound),
  12 artists, 6 best-of lists. All are sampled from the top 120 of
  `rank_candidates`, weighted score⁴. Whatever was shown last time is ×0.5.
  Measured on Comfort: about 4 of 24 albums carry over per refresh, and earlier
  ones come back later.
- A build takes about 5 s (Last.fm fetches for what's due), so it runs on a
  thread. The page shows "Finding…" and fills in on its 4 s poll.
- `basis` is a fingerprint of the folder. Her own adds keep it in step, so
  they never trigger a rebuild. A change from anywhere else rebuilds once the
  section is 6 h old.
- Filed recommendations carry `rec: true` plus artist and album. A card with no
  known Spotify url is filed as a `search` entry, which plays the top result
  and learns the url.
- Voice: several numbers at once go through `LocalPageJumpMany` (LocalViewer.ahk),
  which sends "3 5 9" to kit. `Kit.jumpMany` looks up every target first, then
  fires a cancelable `kit:jumpmany` event. The page adds all its recommendations
  in ONE request. The grammar (`local_pages_commands.py`) lets a tens word or
  "hundred" come only LAST, so "twenty five" is always 25. Works on every local
  page, not just this one.
- `py Scripts/music_folders/music_recs.py show|build|dismissed <folder id>`.

Jamie's decisions:

**Recommended section (each of her own folders, not smart ones):**
- It sits below the folder's items, after a heavy, obvious divider. Rows go
  Albums, then Artists, then Playlists last. Playlists come from our best-of
  lists, since we have no playlist graph.
- The ranking is the folder-seeded Different engine ([[MUSIC_GRAPH]]). The seed
  includes sub-folders, the same way Random does.
- **Frozen while she works.** It's stored per folder. Adding a card marks it
  "✓ added" in place and never reshuffles, so the numbers stay put. It rebuilds
  on Refresh, or after the folder changed some other way and she comes back.
- **Dismissed = gone for good** for that folder. A refresh is NOT a blocklist:
  cards shown before can come back. Refresh should still change a decent amount,
  so last time's cards are down-weighted, not excluded.
- **Numbers mean what their section means.** A number on an item already in the
  folder (top) plays it. A number on a recommendation adds it to the folder and
  does not play it. A click on a card still plays it, as a preview.
- **Several at once, no verb.** Saying "three five nine" adds 3, 5 and 9. This
  is done in the voice grammar, not by splitting a digit string: a rule of
  repeated integers parses spoken words unambiguously ("two seven five" = 2, 7, 5;
  "two seventy five" = 2, 75), and a typed "359" stays card 359.

**Mood folders (chill, existential, hype, springtime, ...):**
- Generated, not hand-filled. Each mood is a row in `INIDATA/music_moods.json`
  (tags wanted, tags to avoid, residents, description), built on the search
  engine's plans. A mood is data, never code.
- Our data is genre-shaped, not mood-shaped. The plans lean on Last.fm mood tags
  plus stand-ins for words that are rare as tags (existential → introspective,
  philosophical; springtime → pastoral, sunny). Spotify's audio features (energy,
  valence) are closed to new apps, so "hype" comes from tags alone.
- Albums only for v1. Mood is really per song, but we have no track pipeline.
- **Layout:** Residents at the top (her permanent picks; a number plays), then a
  **Loved** section (music she already loves that fits the mood), then a **New**
  section. A **familiar ↔ new dial** per mood sets how big each section is.
- **Two buttons on every generated album:** Play, and Add as a permanent
  resident. A number on a generated card = add. Residents are the strongest
  signal and re-centre the mood. That's how she improves the algorithm by hand.
- **Feedback:** two different no's. "Not this mood" tunes the mood; "not for me"
  tunes her taste everywhere. Plus a 0–10 rating and note per mood folder, and a
  `review` list of low-rated moods to tune together in chat (like
  `music_search.py review`).
- Moods get the Recommended machinery too. Seasonal ones (springtime) show up in
  Daily during their season.
- Starter set: her four plus picks from rainy day, late night, slow morning,
  focus/programming, tender/grief, joyful, cathartic, cozy winter, summer heat,
  walking, cleaning.

## Later

YouTube links (thumbnail), local albums (folder art) — the entry shape already
allows any `{url, title, creator, cover}`. Drag-to-reorder. A Media hub Miller
branch.
