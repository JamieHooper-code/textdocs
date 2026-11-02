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

## Grab this, link playlists, headings, play counts (2026-10-29)

**"grab this"** (bare, on a YouTube tab) → `GrabThisYouTube`
(`Helpers\MeditationGrab.ahk`). It reads the url, asks YouTube what it is
(`med_packs.py peek`: nothing saved), then the question window asks: the
playlist or just this video (when the url has both) → **what is it?** (Series /
Yoga / each heading / Ambient / a new heading…) → **how to keep it?** (link /
audio ~1.4 MB a min / video ~6 MB a min). `med_packs.py grab` files it and
starts the download at once. "grab this <folder>" is unchanged (music folders).

- **Three modes** (`media` in packs.ini): `link` = nothing downloaded, plays
  from YouTube in the page (the old `video_packs` embed); `audio`; `video`.
  Any manifest item not on disk is a **link row** (`file = yt:<id>`), so a
  half-downloaded playlist is fully playable. Link items have state `link`,
  which the downloader never queues.
- **Buttons / keys / voice** on the page: Download audio (Shift+A, "download
  audio"), Download video (Shift+V, "download video"), Link only (Shift+L,
  "link only"), ⬇ per row / Keep (K, "keep this" = just this one, as the
  heading's `keep_as`), ↗ YouTube (Y, "open on YouTube"), Listen only (V).
  Voice rule `rules\meditation_page_commands.py`, scoped to the two page titles.
- **One download run at a time**: `_download.lock`; a button pressed during a
  run leaves its pack in `_download_requests.json` and the run picks it up
  when its queue empties. A YouTube cooldown is reported, not overridden.
- **Headings are data**: `INIDATA\MeditationPacks\sections.json` (id, label,
  page, log_as, default_mode, keep_as). `page = learn` headings are packs of
  `kind = learn`, filed under `E:\Media\Learn\`, shown only on the **Learn
  page** (`/learn` on this server, "open learn", ✦ in the hub header), never
  in the lockout or `<med_pack>`. A new heading logs nothing by default
  (`log_as` empty) but is still counted.
- **Single videos** go into the heading's `<heading>_singles` pack
  (`harvest/singles_<pack>.json`).
- **Ambient** → `clog media-add --type music --subtype ambient` + regenerate
  the spot names (a new name needs a Caster reload to be sayable).

**Play history** `INIDATA\meditation_plays.jsonl` (append-only, folded per
play_id, then per recording by video id so a count survives a link becoming a
download). COUNT = finished plays + opens on YouTube; SAT WITH = plays past
`meditation.partial_minutes` (default 5) that did not finish. The page counts
seconds actually played (`listened`) and sends them with each progress report;
once a play is a sit the store asks the server to `clog add` it, then to
`clog edit` its minutes as it grows (one entry per play, never one per
report). TimerAudio pushes lockout listening every 30s (source=lockout: counted,
no second log entry), and the lockout's own entry now names the recording
(`_MeditateRecordingNote`). Rows show `3× · sat with 2×`; **Most played** tab;
"meditate most played". The three finishes the progress file already knew
were backfilled (`meditation_store.py backfill-plays`). Older lockout sits
never recorded a title, so they can't be counted.

## Yoga and other video packs (2026-10-28)

A pack with `media = video` in packs.ini holds downloaded **videos** (1080p
.mp4), not mp3s. The first is **Seated Yoga** (Yoga With Adriene, 10 videos,
2h25m, ~1 GB at `E:\Media\Meditations\seated_yoga`). Voice: "meditate seated
yoga" / "meditate yoga" / "meditate chair yoga".

- **Grouping.** A pack's `section` key is the heading it sits under on the
  page. Meditation packs have none and sit under **Series**; yoga packs say
  `section = Yoga`. Card numbers, "jump N" and the 1-9 keys run across all
  groups in display order. More yoga playlists = more cards under Yoga; if a
  group ever wants its own drill-in level, that is a page change, not data.
- **One player.** `#audio` is a `<video>` element inside the hero's video box.
  Mp3 packs play through it with the box hidden; a video pack shows it.
  **Listen only** (button or V) hides the picture and keeps playing, and is
  remembered per browser (localStorage `med.listenOnly`). G or ⛶ or
  double-click = full screen; a click on the video = play/pause.
- **No lockout.** The lockout's TimerAudio plays audio files, so "Sit with
  lockout" is hidden for a video pack, the lockout's meditation menu leaves
  video packs out (`MeditationPackSections(false)`), and "lockout meditate
  seated yoga" opens the page instead with a tooltip saying why.
- **Logging.** `log_as = yoga` makes a finished play log the **Yoga**
  completion template (exercise / physical therapy / health, minutes), not
  `meditate`. Packs without `log_as` log `meditate` with their style as before.
- **Serving.** `local_viewer._send_file` streams files over 32 MB in 1 MB
  chunks; it used to read the whole file per Range request, which is fine for
  an mp3 and wasteful for a 100 MB video Chrome re-requests on every seek.
- The older `meditation.video_packs` setting is a different thing: it plays a
  pack's recordings as YouTube embeds. A `media = video` pack ignores it.

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
