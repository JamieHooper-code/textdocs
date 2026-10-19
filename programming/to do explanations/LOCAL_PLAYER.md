---
tags: [programming, design-doc, music, local-player, mpv, lastfm, local-viewer, ahk]
created: 2026-10-19
updated: 2026-10-19
status: v1 built 2026-10-19 — albums play gaplessly from the drive, scrobble, bar + theater
related: ["[[MUSIC_FOLDERS]]", "[[MUSIC_GRAPH]]", "[[MEDIA_SYSTEM]]", "[[LOCAL_APP_HUB]]"]
---

# The local player — our own albums, played here

Albums on `E:\Media\Music` play **inside the music folders page** instead of on
Spotify: gapless, at full quality, scrobbled to Last.fm, with a Spotify-style bar
(seek line, times, track list, shuffle, repeat, volume) and a **theater** view (the
cover large in the middle, the page dimmed behind it) that opens by itself when an
album starts.

## The pieces

| Piece | File | Job |
|---|---|---|
| Library index | `Scripts/music_local/local_library.py` | One folder = one album. Tags (mutagen), track order (disc, track, filename), format label ("FLAC 24/96"), cover copied to the SSD, blank-file detection. Incremental by folder signature. |
| mpv client | `Scripts/music_local/mpv_ipc.py` | JSON IPC over `\\.\pipe\jamie-music`. **Two connections** (commands / events) — see Traps. |
| Player | `Scripts/music_local/local_player.py` | Starts/attaches mpv, the ALBUM queue, shuffle (albums) / repeat (off/album/track), listening-time counter for scrobbles, state for the bar. Lives in the folders server process. |
| Scrobbler | `Scripts/music_local/scrobbler.py` | Last.fm auth (once), now-playing, scrobble queue (offline-safe), local `scrobbles.db` write. |
| Server routes | `Scripts/music_folders/music_folders_server.py` | `/api/nowplaying` (local or Spotify, whichever plays), `/api/np`, `/api/local` (seek/track/volume/shuffle/repeat/play/play-folder/rescan), `/api/local/state`, `/api/local/spoken`, `/local-cover/<id>.jpg`. Format badges on cards (`_with_local`), local-first launching (`_launch_entry`). |
| Smart folder | `Scripts/music_folders/music_smart.py` | **Local albums** (`smart_local`): Recently added + everything by artist. |
| Page | `Scripts/music_folders/music_folders_page.html` | The bar, track list, theater, "Play local (N)" button. |
| AHK | `Helpers/LocalPlayer.ahk` | `PlayLocalAlbum`, `MusicRescan`, `LocalSeekTo`, `LocalPlayerShow`, `LastFmAuthorise(Finish)`. |
| Filer hook | `Scripts/acquire/library.py` `_index_for_local_player` | Every album the filer writes is indexed at once (torrent pipeline AND manual sort). |
| Data | `MusicData\local\` (`music_paths.LOCAL_*`) | `local_library.json`, `covers\`, `player_state.json`, `scrobble_queue.jsonl`. Session key: `~/.lastfm_session.json`. |
| Settings | `music.*` | `mpv_path`, `local_first`, `local_scrobble`, `local_exclusive` (off), `local_replaygain` (off), `local_theater`. |

## Decisions (Jamie, 2026-10-19)

- **mpv, not a browser `<audio>` player.** Chrome doesn't strip MP3 encoder delay
  (tiny gaps), resamples, and stops when the tab closes. mpv: true gapless, any
  format, Windows media controls (media keys, volume flyout) built in, keeps playing
  through page reloads and server restarts.
- **Shared mode, no ReplayGain by default.** Exclusive (bit-perfect) silences
  Dragon/toasts/YouTube while an album plays. Windows' shared mixer runs at 48 kHz,
  so 44.1 kHz albums are resampled by mpv (high quality); exclusive is the only way
  around it and is a setting away.
- **No SSD cache.** The drive is read continuously while an album plays, so it won't
  spin down mid-album. (The cache stays a back-pocket option if gaps ever appear —
  SSD wear from it would be well under 1% of rated life a year.)
- **The card says the file type** (FLAC 24/96, MP3 320, OGG 96), not "LOCAL".
- **Local beats Spotify** (`music.local_first`): any card / Random / Similar /
  Different whose album is on the drive plays here. Whichever player starts, the
  other pauses (from the folders page; starting Spotify inside Spotify itself does
  not pause the local player).
- **Shuffle is albums, never tracks.** "Play local" on Comfort plays its on-drive
  albums 1, 3, 7, 9, 2… each start to finish.
- **Everything scrobbles**, game soundtracks and ambient included.
- **Theater opens by itself** on every new start (`play_id`); remembered per browser
  so reopening the page doesn't reopen it.
- **No nightly scan.** The filer indexes what it files; "rescan" for hand-added albums.

## Voice (Jamie, 2026-10-19)

- **No "play local" command: `spot <name>` does it.** `SpotResolve` (LinkManager.ahk)
  plays from the drive when (a) the name resolves to a Spotify ALBUM url she also has
  on the drive (`local_library.py match <url>`, through the music catalog -- no
  network), or a slot that points at an album folder; or (b) nothing else matched and
  the words name a drive album (step 3b, `local_library.py spoken --strict`: every
  word in the album/artist name, at least one in the album's, so "spot big thief" is
  not an album). Drive album names are also in the `spot <spot_name>` Choice
  (`gen_spot_choices.py` step 4), regenerated whenever albums arrive or leave.
  The play goes through `MAINFUN.bat PlayLocalAlbumById`, because LinkManager is in
  the always-on closures and LocalPlayer.ahk is not.
- **"seek <words>"** (generic store) -> `LocalSeekTo`: one number under 10 = minutes,
  10-99 = seconds, 100+ = m:ss ("230" = 2:30), two numbers = minutes then seconds;
  number words work too ("two thirty", "ten oh five"). Rules in `_LpSeekSeconds`,
  pinned by `Helpers/Tests/test_local_player_seek.ahk`.
- **"show player"** -> `LocalPlayerShow`.
- **Last.fm needs no command:** clicking Allow on the page `LastFmAuthorise` opened is
  enough; the scrobbler collects the session key on its next call.

## Scrobbling (Last.fm's rules)

A track counts after **half its length or 4 minutes of LISTENING**, whichever is
first; under 30 s never. Listening time is counted once a second while not paused,
and a jump of more than 2.5 s (a seek) doesn't count. Timestamp = when the track
started. Each scrobble goes into `scrobble_queue.jsonl` first and leaves it only when
Last.fm accepts it (batches of 50), so offline plays send later. It is also written
straight into `scrobbles.db`; `lastfm_data.py` sync later inserts the same play with
OR IGNORE on UNIQUE(uts, artist, track), so nothing doubles.

**One-time auth:** `LastFmAuthorise` opens Last.fm's Allow page; after clicking Allow,
`LastFmAuthoriseFinish` stores the session key. Until then plays queue and still land
in `scrobbles.db`.

## Traps found building it

- **One pipe handle can't read and write at once on Windows** (synchronous I/O is
  serialised per handle): a reader thread blocked in ReadFile made the first command
  hang forever. Fix: two connections to mpv.
- **A BUSY pipe is not a missing pipe.** mpv creates one pipe instance at a time; the
  second open right after the first gets ERROR_PIPE_BUSY (Python: EINVAL). Treating
  that as "no mpv" started a SECOND mpv. Only `FileNotFoundError` means absent;
  busy retries. `os.path.exists` can't see named pipes at all.
- **Blank files that look complete:** 51 of 91 tracks in the Wilco *YHF Super Deluxe*
  folder are all zero bytes (right size, no audio — an interrupted download). The
  index skips them and the card says "51 damaged file(s)". Re-download that set to fix.
- **A wrong album-artist tag:** *Elephant* tags its album artist "Elephant"; an
  album-artist that just repeats the album name falls back to the track artist.
- **The page only polls while visible.** `LocalPlayerShow` must RAISE Chrome
  (`BringToFront`), not just select the tab — a selected tab behind another window is
  hidden, never polls, and the theater never opened.
- **No art on the drive** (Disintegration): the catalog's cover (`resolve_album`) is
  used for the card, bar and theater.

## Later

A drive album started by `spot` skips the lockout fence (LockoutBeginExternalMusic).
Media-key routing when Spotify and mpv are both
sessions. Lyrics. Re-downloading the damaged Wilco set. Gapless across albums (only
within an album now). A "Play here" for whole smart folders like Daily (works already
for the on-drive albums in them).
