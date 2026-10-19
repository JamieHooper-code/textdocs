---
tags: [ahk, chrome-extension, video, youtube, plex, voice, stream-deck, design-doc, accessibility]
related: ["[[GLOBAL_CLICK_HINTS]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]"]
status: BUILT 2026-10-18 -- verified live on YouTube and archive.org with the Chrome window in the background; key fallback verified with the bridge down. Speed memory (video > channel > Spanish > 1x) built + verified the same day, replacing Super Video Speed Controller. Plex not yet tried live.
created: 2026-10-18
---

# Chrome video control — through the AHK Macro Bridge

Pause, play, seek, speed, volume, mute and captions for **whatever video is
playing in Chrome** (YouTube, Plex, any HTML5 player), **without that tab being
focused and without sending keys**. It's the second feature on the AHK Macro
Bridge extension. Transport, install and the "a new feature is just an op"
rule are in [[GLOBAL_CLICK_HINTS]] § "Chrome extension — DOM hints".

Before this, every video command sent keys to the focused window. So Chrome
had to be in front, on the right tab, with nothing in a text box. Stream Deck
buttons and voice commands for a video on another monitor just didn't work.

## Pieces

| Layer | File | What |
|---|---|---|
| Page | `ChromeExtensions/macro-bridge/content.js` § media | ops `media.state`, `media.do` |
| Routing | `ChromeExtensions/macro-bridge/background.js` (`pickMediaTab`) | `args.target = "audible"` → the tab playing sound |
| AHK | `Helpers/ChromeMedia.ahk` | `ChromeMedia_Do` / `ChromeMedia_State` + the commands |
| Old names, rerouted | `Helpers/VideoWatching.ahk`, `Helpers/YouTubeFunctions.ahk` | bridge first, old keys/clicks as fallback |
| Voice | `caster/rules/youtube_commands.py` | "play" → `VideoPlayPause` |
| Test | `Helpers/Tests/test_chrome_media.ahk` | clock format + the bridge-down contract |

## The ops

`media.state` → `{found, kind, paused, ended, currentTime, duration, live,
playbackRate, muted, volume, captions{available,on,via,name}, count, url, title, host}`.

`media.do {action, value?, delta?}`, where action is one of
`play | pause | toggle | seek_rel | seek_abs | rate | volume | mute | captions`.
`seek_abs` with a negative value counts from the end. `rate` and `volume` take
`value` (absolute) or `delta` (a step). `mute` with no value toggles.
`captions` takes on, off or toggle. The reply is the state after the action,
with `ok:false` + `error` when the action itself failed (for example
`play refused: NotAllowedError`).

**Which element.** The largest one that is PLAYING. Failing that, the one that
played most recently (a paused video is still "the video"). Failing that, the
largest one with a source. "Largest" stops YouTube's small hover previews from
winning. The search walks **open shadow roots** and same-origin iframes:
archive.org's player is inside nested web components, so a plain
`querySelectorAll` found nothing there. A `play` event doesn't cross a shadow
boundary, so each element found also gets its own `play` listener.

## Which tab

`_ChromeMediaCall` in ChromeMedia.ahk:

1. **Chrome in front:** the tab she's LOOKING AT (title match, the normal
   bridge routing), if that tab has media. So "play" on a paused YouTube tab
   doesn't pause Spotify playing in another tab.
2. **Otherwise** (or no media there): `target: "audible"`. background.js picks
   the audible tab, preferring the one that went audible most recently. If
   nothing is audible (everything paused), it takes the **last tab that was
   audible**, remembered from `tabs.onUpdated` and mirrored to
   `storage.session` because the service worker sleeps. Failing that, the
   normal active tab.

That last-audible memory is what makes "pause … play" work from the
background. Once paused, nothing is audible any more.

## Fallbacks (bridge down / no media in the tab)

`ChromeMedia_Do` returns `""` when there's nothing it could do: host down,
Chrome closed, a restricted page, or no `<video>`/`<audio>`. Every caller
treats `""` as "do it the old way":

| Command | Fallback |
|---|---|
| `VideoPlayPause` | the system Play/Pause media key (reaches Chrome without focus; may hit Spotify instead) |
| `YoutubeSeekMinutes` + the `YoutubeSkip*` family | the old j/l key burst (Chrome must be in front) |
| `VideoSeekSeconds` | arrow keys, 5s each (Chrome in front) |
| `ChromeIncrease/Decrease/ResetVideoSpeed` | none since the speed extension was removed (tooltip "no video found") |
| `YouTubeSetClosedCaptions` / `ToggleClosedCaptions` | the old UIA caption-button path |
| `VideoSetSpeed`, `VideoMute`, `VideoVolume` | none, a tooltip says so |

**Cost of the fallback: about 2 seconds.** A refused localhost connection still
takes WinHTTP ~2s (it retries), so with the bridge down every command waits ~2s
before the keys fire. `test_chrome_media.ahk` bounds it under 3.5s.

## Speed memory (replaces Super Video Speed Controller, 2026-10-18)

Jamie removed the Super Video Speed Controller extension; our own setup now
owns speed. Her Stream Deck speed key is short = `ChromeDecreaseVideoSpeed 3`,
mid = `ChromeIncreaseVideoSpeed`, long = `ChromeResetVideoSpeed` (voice "slow",
"faster", "reset speed" call the same). One step = `CHROME_MEDIA_SPEED_STEP`
= **5%** (her number): a short press is −15%, a mid press +5%. Tooltips show
speed as a percent.

**Every speed she sets is remembered** for that video, its channel, and (when
the title reads as Spanish) Spanish videos. When a video loads, the first
match wins:

```
this video  ->  this channel  ->  Spanish title  ->  1x
```

- English videos stay at 100%: a video with no memory is put BACK to 1x.
  That matters because YouTube keeps the previous video's rate on the same
  `<video>` element when you click into the next one.
- **Channel beats Spanish** (her call): a Spanish channel she always slows
  down stays slowed even after another Spanish channel set the Spanish speed
  faster.
- Only HER speed commands save (`media.do rate`). Applying a memory doesn't
  save anything, and neither does YouTube's own settings menu.
- **Reset to 100% = forget** (her rule): the long press / "reset speed"
  (`media.do speed_reset`) deletes THIS video's and THIS channel's entries
  and plays at 100% now. It leaves the Spanish speed alone, so a Spanish
  video will pick the Spanish speed up again on its next load.
- **The Spanish speed has its own command**: voice "set Spanish speed <n>"
  (percent, generic store, `SetSpanishSpeed`), via `media.memory {spanish}`.
  It applies to the video on screen at once if that video has no speed of its
  own. **"set Spanish speed 100" forgets it** (just the Spanish speed, not any
  Spanish video's or channel's own entry). Changing speed on a Spanish video
  with the speed key still updates the Spanish speed too.
- Other sites remember per video (host + path). Channel and Spanish are
  YouTube only.

**Where:** content.js § "speed memory", stored in the extension's own
`chrome.storage.local` under `speedMemory` = `{v: {<video>: {rate, at}},
c: {<@handle>: {rate, name, at}}, es: {rate, at}}`. It's kept in the page so
the speed applies the instant a video loads, with no AHK round trip. Videos
are capped at 3000 (oldest forgotten); channels never are.

**Spanish:** `isSpanishTitle()`, a copy of `clog.py`'s `is_spanish_title()`
word lists (ñ¿¡ = Spanish; else ≥2 Spanish function words and more Spanish
than English). Keep the two in step.

**YouTube is a single-page app:** right after a navigation, the owner row and
the `<h1>` still show the PREVIOUS video. `mediaFacts()` waits (≤3s) until
`ytd-watch-flexy[video-id]` matches the URL's `v=` before reading the channel
and title. Triggers: `loadedmetadata` / `playing` (capture listeners), plus
YouTube's `yt-navigate-finish`. For 4s after the memory applies a speed, a
`ratechange` that moves it is undone (a player re-asserting its own setting);
after that, changes are left alone.

**Seeing / undoing it:** `ShowVideoState` says "speed remembered from:
video|channel|spanish", and a speed command's tooltip says what it saved
("Speed 0.7x · saved for video + channel + Spanish"). The `media.memory` op
reads it, sets the Spanish speed (`{spanish: rate}`, 1 = forget), or forgets
an entry with `{forget: "video"|"channel"|"spanish"|"all"}` (this page's
video/channel). Day-to-day forgetting is just "reset to 100" and
"set Spanish speed 100".

Verified live 2026-10-18: Blender video 1.2x then a fresh load → 1.2x
(video); Spanish video A slowed to 0.7x; Spanish video B (a different channel,
no memory) loaded at 0.7x (Spanish); B set to 0.9x; A with its video entry
forgotten reloaded at 0.7x (channel beat Spanish 0.9x); a related-video
click from the 1.2x Blender tab landed on an English video at 1x. The test
speeds were then cleared (`forget: all`). Then the 5% steps: +1 → 105%,
short press → 90%, reset → 100% "forgot video + channel", reset again →
"nothing was remembered"; voice "set Spanish speed eighty five" → a Spanish
video loaded at 85% (spanish); "set Spanish speed one hundred" → forgotten,
the same video back to 100%.

**Known gap:** the last-audible memory lives in the service worker and starts
empty after an extension reload. A video that was ALREADY playing before the
reload, then paused, isn't found again by a background command until it
plays once more. (background.js could seed `lastAudible` from the audible
tabs at startup.)

## Things measured that weren't obvious

- **(History, fixed by removing it.) A speed extension held the rate.** Jamie ran *Super Video Speed
  Controller* (the Alt+Shift+Y popup the old speed functions click). It
  re-applies its stored speed on every `ratechange`, so setting
  `video.playbackRate` snapped back to 1x within a second. Calling YouTube's
  own `movie_player.setPlaybackRate()` from a page-world script snapped back
  too, which is how the culprit was identified. Its in-page keys do NOT check
  `isTrusted` (`a` = +0.01, `d` = −0.01, `s` = reset to 1), so `setRate` in
  content.js sets the element first, and if it hasn't stuck 250ms later it
  walked that extension there in its own 0.01 steps. She removed the extension
  the same day and that workaround is gone: `setRate` is a plain set again.
  If a set ever snaps back, look for another extension doing the same.
- **YouTube's own shortcuts ignore synthetic keys** (Shift+. did nothing), so
  don't go down that road for YouTube.
- **YouTube captions aren't textTracks.** YouTube draws its own, so captions
  there go through `.ytp-subtitles-button` (`aria-pressed`). The button exists
  even on a video with NO captions: its label then reads "Subtitles/closed
  captions unavailable", and that is the availability test. Everywhere else
  it's the element's `textTracks` (subtitles/captions kinds; "on" picks the
  browser-language track first).
- **Autoplay policy.** `play()` from the extension is refused
  (`NotAllowedError`) on a page that has never had a real user gesture.
  Pages she started herself already have one, so this only shows up when a
  script opens a page and tries to start it. One real keypress unlocks it.
- **Hidden tabs throttle timers to about 1s.** Every wait in the media code is
  bounded by elapsed time, not by a count of short sleeps, or a silent
  background tab could outlast the bridge's 2.5s timeout. (A tab playing audio
  isn't throttled.)
- **No page reload needed after an extension edit for these ops.** After
  `ChromeBridgeReloadExtension` the old content script is orphaned, the send
  fails, and background.js injects the new one on demand. (`ChromeBridgeReloadTab`
  is still the clean way.)

## Limits

- A player in a **cross-origin iframe** (an embedded video on another site) is
  out of reach. content.js runs in frame 0 only. A **closed** shadow root
  is out of reach too.
- **Plex**: not yet tried live. Plex web plays through a normal `<video>`, so
  play, pause, seek and volume should work. Its subtitles may be drawn by Plex
  rather than through textTracks, in which case `captions` reports none.
- Volume is the PLAYER's volume, not Windows'. YouTube may put its own back
  when the next video loads.

## Verifying

```powershell
Start-Process MAINFUN.bat -ArgumentList 'ShowVideoState' -WindowStyle Hidden
```
Log lines: `ChromeMedia/do` (acted), `ChromeMedia/miss` (fell back, with why),
`TOOLTIP/ChromeMedia`.
