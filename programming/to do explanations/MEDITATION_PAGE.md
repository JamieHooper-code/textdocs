---
tags: [design, meditation, web, viewers, lockout]
related: ["[[MEDITATION_PACKS]]", "[[LOCAL_APP_HUB]]", "[[EXCEPTION_LOCKOUTS]]", "[[COMPLETION_LOG]]", "[[SETTINGS_SYSTEM]]"]
status: built (v1) — 2026-10-06
updated: 2026-10-06
---

# The meditation page

**Status: BUILT.** `http://127.0.0.1:8295`. The guided-meditation packs
([[MEDITATION_PACKS]]) shown as **series you work through**, like current reads
or watches. Before this, every way into a guided meditation started a lockout.

## Voice

| Say | Does |
|---|---|
| **meditate \<pack\>** | opens that series on the page and plays what's next. **No lockout.** |
| **meditate favorite \<pack\>** | same, from that pack's favourites |
| **open meditate** | the page (was the Miller) |
| **lockout meditate \<pack\>** | the timed lockout sit, unchanged |
| **[lockout] meditate \<pack\> \<n\>** | a spoken number still means a lockout of n minutes |

The Miller is still what the lockout's `L` key opens.

## The page

- **Continue card**: the recording in progress (Resume), else the next in
  the series (Begin). **Sit with lockout** hands that recording to
  `StartMeditationPack` via `StartMeditationFromPage`.
- **Series cards**: one per pack, finished/total, hours, what's next. Keys
  `1`–`9` switch series.
- **The list**: series order, length, ★ favourite, ✓ finished, ⊘ skip, and a
  thin bar for where you stopped. Filters: All / Unfinished / Favourites.
- **Player bar**: Space play/pause, ← 15s, → 30s, N next, F favourite.
- **⚙ Settings**: the `meditation` settings system through the web panel
  ([[LOCAL_APP_HUB]]).

## Rules

- **Series order is the playlist's order**: the harvest manifests' `n`, with
  playlists taken in `sources.json` priority. Files with no manifest row go last.
- **"Next"** = the pack's most recently played recording if it's still in
  progress (≥20s in, not finished), else the first one neither finished nor
  skipped. A finished series starts over. The page and `meditate <pack>` use
  the same rule (`meditation_store.next_up`).
- **"Finished"** = listening past `meditation.done_percent` (90%). Counted **once
  per play**: the page mints a `play_id` each time a recording starts and the
  store remembers the ids it counted. The page reports position every
  `save_every_seconds`, so without the id one sit would log dozens of times.
- A finish is **logged** to the completion log (`clog add meditate`,
  `source=meditation-page`, style = the pack's style, note = pack: title)
  unless `meditation.log_page_sits` is off. Ticking ✓ by hand never logs.
  Marking isn't sitting.
- **Favourites and skips are the lockout's own files** (`<pack>_favorites.txt`,
  `url_tags.json` `skip_meditate`), written in the same shape. ★ on the page
  and F in the overlay are the same star.
- **Video**: packs listed in `meditation.video_packs` play the YouTube video in
  an embedded player instead of the local mp3. Progress tracking is the same.

## Autoplay

Chrome won't start sound on a page nobody has touched, and a voice command
isn't a touch. If `play()` is refused, the page shows a **Press Space to begin**
gate and tells the server (`/api/autoplay`). `MeditatePack` polls `/api/state`
for about 6s and presses Space itself. A real keypress counts as a user
gesture. Chrome learns the origin over time and the gate stops appearing.

## Files

| | |
|---|---|
| Store (order, progress, next, fav/skip writes) | `Scripts/meditation/meditation_store.py` → `INIDATA/meditation_progress.json` |
| Server | `Scripts/meditation/meditation_server.py` (`serve`/`state`/`restart`) |
| Page | `Scripts/meditation/meditation_page.html` (read per request) |
| AHK | `Helpers/MeditationPage.ahk` |
| Voice | `rules/lockout_commands.py` (meditate block) + generic store row "open meditate" |
| Tests | `Scripts/codebase_tools/tests/test_meditation_page.py` |

**Stale-code trap:** the server holds `meditation_server.py` and
`meditation_store.py` (and `local_viewer.py`) in memory. After editing any of
them, run `py Scripts/meditation/meditation_server.py restart`.

**Lockout hand-off goes through a temp file**
(`%TEMP%\meditation_page_lockout.json`), not a MAINFUN argument. The filenames
carry fullwidth `｜` and `：`, which don't survive a batch-file argument list.

## Known data gap (2026-10-06)

The Lovingkindness manifest marks 32 recordings downloaded, but only 26 files are
on disk. `med_download.py --verify --repair` is the tool for that.
