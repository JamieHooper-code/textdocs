---
tags: [design, meditation, lockout, media, scraping, architecture]
related: ["[[EXCEPTION_LOCKOUTS]]", "[[COMPLETION_LOG]]", "[[MEDIA_SYSTEM]]", "[[SETTINGS_SYSTEM]]", "[[TIMER_OVERLAY]]", "[[VOICE_COMMAND_SYSTEM]]"]
status: built (v1) — 2026-08-26
updated: 2026-08-26
---

# Guided-meditation packs

**Status: BUILT.** Voice: **"meditate guided"**, **"meditate short"**,
**"meditate loving"**, **"meditate watts"**, **"meditate flowers"**,
**"open meditate"**. The `L` key inside a meditation lockout opens the same
Miller.

This is the thing [[EXCEPTION_LOCKOUTS]] left open — *"Guided meditation pack.
The backend is built and logs a style; the audio pack doesn't exist yet. When it
does it plugs in as an audio source."* It plugged in.

## The decision everything hangs off: the clock is DECOUPLED from the audio

The first design got this backwards and it is worth recording, because the wrong
version sounds reasonable.

The wrong version: the session *is* the recording, so the timer follows the
track — warm-up + the recording's length + cool-down, retargeting when you skip.

Jamie's actual rule:

> "The limit is for the lockout itself. If we're playing a 50-minute meditation,
> the length of the actual lockout will be 22 minutes, but it will keep playing
> when it's done. It can play for an hour, but I can cancel it at 22 minutes.
> Skipping to another track does not change the clock at all."

So:

```
lockout_minutes = min(ceil(track_duration), pack.max_lockout_minutes)
```

An 11-minute recording gives an 11-minute lockout. A 50-minute talk gives a
22-minute lockout, Escape unlocks at 22, and the recording keeps going for
another 28 minutes if she wants it to. Sitting on past the unlock is *expected*,
and the existing overtime/loot ladder already rewards exactly that.

Three things fall out of this, and all three are why the decoupled version is
better than the one it replaced:

- **`--first-track` is load-bearing.** The clock has to be sized *before* the
  overlay opens, so the launcher resolves a specific starting recording up front
  and hands it over, rather than letting TimerAudio pick at random after the
  countdown has already started.
- **Nothing has to change in TimerAudio.** The coupled version needed playback
  gated on the warm-up boundary — the single biggest change in the whole
  proposal. The decoupled version needs none of it: the audio behaves exactly as
  a music pack's does.
- **`timer_extend.txt` stays out of it.** Skipping never retargets, so the
  mid-session IPC the `+`/`−` keys use is untouched.

"Up to 22 minutes" is therefore a **cap on the lockout**, never a filter on the
library and never a truncation of the audio. Every recording stays playable —
which matters, because a 22-minute *filter* would have excluded all 17 Alan
Watts talks (his shortest is 23:06) and 44 of the 116 Plum Village meditations.

## Why packs.ini and not a new registry

A meditation pack is registered in the **same `packs.ini`** as the music packs,
marked `kind = meditation`. That inherits, with no new code: favourites, the
global-vs-local skip model, the overlay's `F`/`S`/`L` keys, tagged playback,
`lockout switch`, TimerAudio enumeration, and `sound_source.py`'s `pack:`
adapter.

A separate `meditation_packs.json` was the alternative. It would have meant
re-implementing enumeration, favourites, skip and the picker — four things that
already work — to gain nothing but tidiness.

What INI genuinely cannot hold is **per-track** metadata, and here the per-track
duration is not a nicety, it *is* the session length. So that lives in a
`tracks.json` sidecar in the pack folder, next to the `url_tags.json` and
`favorites.txt` the music packs already keep there.

New fields on a pack section:

| field | meaning |
|---|---|
| `kind` | `meditation`. Absent = a music pack, as before. |
| `max_lockout_minutes` | The lockout ceiling. Not a limit on the recording. |
| `style` | The `meditation_styles.json` key these sits are logged as. |

Every meditation pack also carries `rotation_days = 0` and
`exclude_from = lockout`, so a bare **"lockout"** can never start a guided
meditation as programming background music. They stay in `[Rotation].order`
regardless — that is the file's own documented convention for "out of rotation
but remembered by the UI", and it is what makes them appear in the pickers.

## Packs and style are two different axes

Jamie's four example commands turned out to be three different axes wearing one
costume: `short` is a *length*, `loving` is a *practice*, `guided` is a *mode*,
`watts` is a *teacher*. Two of them (`guided`, `metta`/"loving kindness")
already existed as **styles** in `meditation_styles.json`.

The resolution: **a pack CARRIES a style, it is not one.** `meditate loving`
starts the Sharon Salzberg pack and logs `style = metta`. `meditate metta` still
starts a bare silent sit logged the same way. The pack is the audio axis, the
style is the practice axis, and the log keeps recording the practice.

### The GrammarError this would otherwise cause

`<med_pack>` and `<med_style>` are two `Choice`s in **one** `MappingRule`.
Offering the same word from both is what raises Dragon's *"Malformed recognition
data"* `GrammarError` — which does not fail politely, it **takes down every
lockout command at once, silently**.

So where a pack and a style want the same word, the pack wins and the style gets
`"spoken": false` — it stays in the registry as a **log value** (the completion
template's options are asserted equal to the registry's keys) but is left out of
the voice Choice. That applies to `guided` and `watts`. `metta` keeps its own
spoken form because the pack took `loving`, not `loving kindness`.

Dropping `guided`'s spoken form costs nothing: a bare "guided" sit with no
recording is a contradiction. Its old alias **"meditate guide"** was a live
habit, so the pack picked it up rather than orphaning it.

`test_pack_words_and_style_words_are_disjoint` pins all of this.

## One Miller, two meanings

`open meditate` is a normal own-process Miller. The `L` key **inside a
meditation lockout** opens the *same* menu, and the same tree does a different
thing in each place:

- **From the desktop** — Enter on a recording STARTS a lockout sized to it.
- **From inside a lockout** — Enter JUMPS the running session to it. Starting a
  second lockout on top of the first would be nonsense, and the clock is
  deliberately never re-sized mid-sit.

The branch is checked at **action** time, not build time, so a menu left open
across the start of a lockout still does the right thing. Every leaf's detail
line states which it will do — a picker that silently means two things is a
picker you cannot trust.

Music-pack lockouts keep the original ListBox picker. It works, and converting
it was not asked for.

### Why the overlay can host this at all

The Miller runs in **its own process**, and that is the entire reason this is
safe. The overlay is a fullscreen always-on-top window with *global* hotkeys;
hosting the Miller's own hotkey layer inside it would put the two in a fight.
A sibling process needs only two things from the overlay:

1. **The keyboard.** Every hotkey scope gained `&& !g_MedMillerOpen`. This is
   the same passthrough mechanism the exception lockouts use: the key reaches
   the other window only when **no variant is eligible at all**. A no-op handler
   would *swallow* it. Note the existing picker scope could not be reused — it
   deliberately *claims* Left/Right/Enter, which is the opposite of what a
   foreign window needs.
2. **The Z-order.** `_ApplyTopmost(g, false)`, which already encodes the trap
   that clearing `WS_EX_TOPMOST` does **not** lower a window (Windows raises it
   to the top of the non-topmost band) and pairs it with `WinMoveBottom`.

The watcher polls the viewer's **PID**, not a window title — the title changes
with the level she is on, and losing that race would leave the lockout
permanently unable to hear a keypress, the one failure this feature cannot
afford.

## Scraping: sources are plural, state is per-video

`INIDATA/MeditationPacks/sources.json` holds one row per **playlist**, each
carrying a `pack`. Several playlists may name the same pack — that is how a new
playlist gets *added to an existing pack*, and it is why the download queue
walks per-playlist manifests instead of a per-pack URL field.

`harvest/<playlist_id>.json` is the manifest: every video with its id, url,
title, duration and a `state` of `pending` → `downloaded` / `skipped` /
`failed`. Re-harvesting a playlist **preserves** state, so upstream adding
videos never resets what is already on disk.

### The drip

`med_download.py` walks pending items in priority order and runs **until
YouTube says stop**, which is Jamie's rule: *"just max the quota every day and
make it so that it continues until it hits the limit."* There is no per-day item
cap — the rate limit *is* the cap, and guessing a lower number just makes the
corpus take longer for nothing.

The judgement about *when* to stop is not re-implemented. It was lifted out of
the music-pack downloader into `Scripts/yt_error_taxonomy.py` and both now
import it. Two drifting copies would mean two answers to "should I keep going
after this error", and the wrong answer deepens an IP flag that takes 24-48h to
decay.

**The cooldown guard** exists because a daily task plus a bot-block is a trap:
the task fires tomorrow, gets blocked in one request, and every retry digs
deeper. A STOP of kind `bot_block` / `rate_limit` writes a cooldown-until stamp
and the next run **refuses to start** before it expires.

Scheduled task: `MeditationPackDrip`, daily 03:00 + logon+15m,
`StartWhenAvailable`, hidden, 8h cap. Registered via **`schtasks.exe /XML`, not
`Register-ScheduledTask`** — the cmdlet writes to the root task folder, which
needs elevation, and it fails with a *non-terminating* "Access is denied" that
sails straight past `$ErrorActionPreference` and prints a success message over
the top of its own failure.

### A file that EXISTS is not a file that is CORRECT

Jamie closed the download window mid-run on the first day. Everything about the
resume was fine — state is written per item, so the interrupted one stayed
`pending` and the 52 already marked `downloaded` all verified intact — but it
exposed a real trap on the *next* run.

The kill landed mid-encode, leaving a complete-looking `.mp3` that was **1321
seconds of an expected 2906**. `yt-dlp --no-overwrites` checks only that a file
*exists*, so the next pass would have skipped the download, printed the path,
reported success, and marked a half-length meditation `downloaded` — permanently,
with no error anywhere. Worse, the length in the manifest is what **sizes the
lockout**, so the clock would have promised 22 minutes over a recording that ran
out at 11.

So every download is now length-checked with `ffprobe` against the manifest
(tolerance: 2% or 10s, whichever is larger — a re-encode never lands exactly on
the source, and YouTube's own metadata is a second or two loose) and re-fetched
once with `--force-overwrites` on a mismatch. A second failure is marked
`failed` loudly rather than accepted. `med_download.py --verify [--repair]` runs
the same check over the whole corpus, which is the thing to reach for after any
interrupted run.

The same kill also left a **41 MB `.webm`** behind. Intermediates
(`.webm/.m4a/.opus/.part/.ytdl/.temp`) are now swept at run start — safe there
and only there, because the task is `MultipleInstances IgnoreNew`, so nothing in
the folder can be a live download.

### The console window that could be closed at all

The underlying cause of that interruption is worth its own note, because Task
Scheduler's `<Hidden>true</Hidden>` is misleading: it hides the task in the
Scheduler's own UI, **not** the console window its action spawns. A nightly
`cmd.exe /c run_drip.bat` therefore put a live window on the desktop for hours.

The scheduled action now runs `pythonw.exe` directly — no console exists to
close. `run_drip.bat` is kept deliberately for a manual pass you *want* to
watch, and `log()` guards its `print` because pythonw has no stdout. The durable
record is `INIDATA/MeditationPacks/_download.log` either way.

### The stale-yt-dlp trap (this one affects the music packs too)

Plum Village's "Guided Meditations" playlist reports 116 videos. yt-dlp
enumerated exactly **100**, twice, with and without `-I 1:500` — no error, no
warning. The installed yt-dlp was `2026.05.05`, flagged as >90 days old.
Updating it recovered all 16.

`add_pack.ps1` uses the same flat-playlist enumeration for the **music** packs,
so any playlist over ~100 videos may have been silently under-seeded there too.
`harvest_playlist()` now always compares harvested against `playlist_count` and
reports the gap; the manifest records `missing_from_harvest`.

## The corpus (harvested 2026-08-26)

| pack | playlist | uploader | videos | median | max | total |
|---|---|---|---|---|---|---|
| `guided` | Guided Meditations | Plum Village App | 116 | 12.0m | 57.8m | 33.4h |
| `watts` | Alan Watts \| All-Natural | Official Alan Watts Org | 17 | 39.7m | 62.5m | 10.9h |
| `short` | On-the-Go \| Short Guided Meditations | Plum Village App | 36 | 6.8m | 12.0m | 4.1h |
| `flowers` | Flowers in the Dark | Plum Village App | 27 | 3.6m | 27.6m | 2.2h |
| `loving` | Lovingkindness | Sharon Salzberg | 33 | 19.4m | 63.3m | 11.3h |

229 recordings, 61.9 hours. Download priority is Jamie's call — the regular
guided ones first, then Watts, then the rest — and it is **data** (`priority` in
sources.json), so re-prioritising is an edit, not a code change.

## Files

| Layer | File |
|---|---|
| Registry | `INIDATA/packs.ini` (`kind = meditation`), `INIDATA/MediaPaths.ini` |
| Sources + manifests | `INIDATA/MeditationPacks/{sources.json, harvest/*.json}` |
| Per-track metadata | `E:\Media\Meditations/<pack>/tracks.json` |
| Registry CLI | `Scripts/MeditationPacks/med_packs.py` |
| Downloader | `Scripts/MeditationPacks/med_download.py` (+ `run_drip.bat` for a *visible* manual pass) |
| Scheduled task | `Scripts/MeditationPacks/register_drip_task.ps1` |
| Shared yt-dlp taxonomy | `Scripts/yt_error_taxonomy.py` |
| AHK — reads + curation | `Helpers/MeditationPacks.ahk` |
| AHK — launchers | `Helpers/MeditationFunctions.ahk` |
| AHK — the Miller | `Helpers/MeditationMenu.ahk` + `Scripts/MeditationViewer.ahk` |
| Overlay integration | `Scripts/LockoutTimer/TimerOverlay.ahk` (`_ShowMeditationMiller`) |
| Voice | `rules/lockout_commands.py` (`<med_pack>`) |
| Tests | `Scripts/codebase_tools/tests/test_meditation_phases.py`, `Helpers/Tests/Gui/test_meditation_menu.ahk` |

## Adding another pack, or another playlist to one

```bash
py Scripts/MeditationPacks/med_packs.py register <pack> --playlist <url> \
    --display "Display Name" --style guided --cap 22 --aliases "alias one,alias two"

py Scripts/MeditationPacks/med_packs.py add-playlist <pack> --playlist <url>
```

`register` writes packs.ini, MediaPaths.ini, the folder and the favourites file;
`add-playlist` harvests and attaches. Both are idempotent. Then say
**"reboot caster"** so the `<med_pack>` Choice re-reads packs.ini.

## Known limits / still open

- **Loudness is not normalised yet.** `sound_source.py` already does
  clip-guarded per-file LUFS normalisation and it is *off* for the `lockout`
  profile. These recordings vary a lot in level and she listens quiet, so a
  `meditate` profile with `normalize_lufs = -18` is the obvious next step; it
  needs a live listen to confirm the target.
- **The music-pack `L` picker is still the old ListBox.** Deliberate — it works,
  and folding it into a Miller was not asked for.
- **`add_pack.ps1` may have under-seeded existing music packs** (see the stale
  yt-dlp trap). Worth a `-ReseedGenreOnly` pass over any pack whose playlist has
  more than ~100 videos.
