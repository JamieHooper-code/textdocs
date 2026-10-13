---
tags: [programming, design-doc, music, spotify, local-viewer, ahk, caster]
created: 2026-10-10
updated: 2026-10-10
status: v1 built 2026-10-10 — Spotify only; YouTube / local albums later
related: ["[[MEDIA_SYSTEM]]", "[[LOCAL_APP_HUB]]", "[[SPOTIFY_SCRAPING_INTERNALS]]", "[[REMINDERS_SYSTEM]]"]
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
- **Links and covers:** only about 40% of recently played albums are in
  music.json. Each row resolves catalog → release_dates.db `catalog_links` →
  Last.fm `album.getInfo` cover plus a Spotify SEARCH url (cards marked
  "search"). Clicking one plays the top result (`SpotPlayTopSearchResult`), then
  `music_smart.py learn` records the album url it landed on, so the next click is
  direct. Network answers are cached in `E:/Media/catalog/music_folders_cache.json`
  by a background pool. The page never waits on the network; it fills covers in
  on its 4-second poll. The cache is re-read when the file changes, because
  `learn` writes it from another process.
- **Fixed alongside:** `OpenOrNavigateChromeTab` now presses Delete before Enter.
  Chrome's inline autocomplete was turning a typed `http://127.0.0.1:8297/`
  into a deeper page from history (a deleted folder → "Not found").

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
  `E:/Media/catalog/music_folders.json` (`{folders:[{id,name,parent,aliases}],
  entries:[{id,folder,kind,url,title,creator,cover,catalog_id,added}]}`).
  `MUSIC_FOLDERS_STORE` env var = scratch store for demos/tests.
- Page: `music_folders_server.py` (:8297, `restart` after editing it or the
  store module, because of the stale-code trap) + `music_folders_page.html`
  (read per request).
- AHK: `Helpers/MusicFolders.ahk`. Voice: `rules/spotify_commands.py`,
  `rules/spotify_global_commands.py`.

## Later

YouTube links (thumbnail), local albums (folder art) — the entry shape already
allows any `{url, title, creator, cover}`. Drag-to-reorder. A Media hub Miller
branch.
