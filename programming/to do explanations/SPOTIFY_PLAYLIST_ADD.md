---
tags: [autohotkey, spotify, playlists, music-catalog, voice-commands, caster, design-doc]
---

# Adding songs to Spotify playlists — the 2026-08 rebuild

Built 2026-08-01. Replaced a system written in one commit on 2026-05-10 that
predated the proper data backends. See also [[MEDIA_SYSTEM]] (the catalog this
now lives in) and [[SPOTIFY_SCRAPING_INTERNALS]] (the UIA machinery it borrows).

## The constraint that shapes everything: there is no Spotify API

Not "we chose not to" — **there is no available path.** Spotify's February 2026
Web API change requires the app OWNER to hold an active Premium subscription for
*any* Development Mode app, and a free account can no longer register one at all.
Jamie is on Free. This was already confirmed empirically on her own account:
`C:\Users\jamie\.spotify_credentials.json` carries a note from 2026-06-06
recording a **403 "Active premium subscription required for the owner of the
app"** against those credentials.

So every "just call the API" instinct is dead on arrival:

| Want | API endpoint | Status |
|---|---|---|
| add a track to a playlist | `POST /playlists/{id}/items` | blocked (no app) |
| list her playlists | `GET /me/playlists` | blocked (no app) |
| what's playing now | `GET /me/player/currently-playing` | blocked (no app) |

The last one is replaced by reading Spotify's own now-playing bar (the
`Now playing: <Track> by <Artist>` group). **SMTC was tried first and is wrong
for this** — it reports the system-wide session, so with a YouTube tab playing
it returns that video while Spotify holds something else. See the verified
section below.

Everything else is driven through the web player's own context menu. Do not
re-litigate this without first re-checking Spotify's developer policy — the note
in the credentials file claiming "currently-playing works on Free accounts" is
**stale**, written before the app-owner Premium gate existed.

## What was wrong before

1. **Two registries that didn't talk.** `INIDATA/VoiceChoices/playlists.json`
   held 6 hand-typed entries and was the *only* thing the add flow could target.
   `E:\Media\catalog\music.json` held 63 playlists. `add playlist` wrote the
   second; the add flow read the first. A playlist added the "modern" way was
   never addressable — and they had already drifted (`cedar` in SpotLinks.ini
   pointed at a different playlist than `Cedar` in playlists.json).
2. **Hardcoded pixel bands.** Track rows were found by filtering
   "More options for ..." buttons against `winX+300` / `winY+540` /
   `winH-110` — magic numbers that mis-count the moment Spotify reflows.
3. **Two blind keystroke assumptions.** `{Down}{Enter}` assumed "Add to
   playlist" is always the first context-menu item; `{Down}{Down}{Enter}`
   assumed the first real result is always exactly two rows down. Either
   changing silently files the track somewhere wrong.
4. **One playlist per invocation**, and only by row number (capped at 20).
5. **Hand-curated databank** — three InputBoxes per playlist.

## The shape now

```
voice / Stream Deck
      |
Helpers/SpotifyPlaylistAdd.ahk        <- UI automation + the picker
      |
Scripts/MediaCatalog/playlist_targets.py   <- ALL data logic
      |
E:\Media\catalog\music.json  (subtype "playlist", add_target true)
      |
INIDATA/VoiceChoices/playlist_targets.json  <- generated {phrase: title} cache
```

**One store.** Playlists live in the catalog like everything else. Three fields
this system owns on a playlist row:

- **`add_target: true`** — the curation flag. music.json has 69 playlists, most
  of them scraped noise; this marks the handful Jamie actually files into, so
  the picker stays short. Promoting a scraped playlist does **not** duplicate it
  (`add-target` dedups by Spotify id / URL / title first).
- **`voice_phrase`** — **opt-in, never bulk-assigned.** Only `add playlist
  target` sets one. Jamie asked for this explicitly: she does not want every
  playlist speakable. `save playlists` deliberately assigns none.
- **`add_history`** — our record of adds we performed (`[{track, at}]`), capped
  at 500. **Not Spotify truth** — there is no API to ask — so the picker labels
  it as "already added (by this system)". It exists to make a double-add visible.

**Why a generated Choice cache:** music.json is ~56 MB. Caster must not parse
that on every rule reload, so `playlist_targets.py` rewrites a tiny
`{phrase: display_name}` file after *every* mutation (`sync_choices` is called
inside the same catalog lock as the write, so it cannot drift). Same pattern as
`hardcoded_spot_names.json`. Never hand-edit it.

## Hardening the add mechanism

- **Name-based row lookup.** Track rows are the more-options buttons whose name
  contains `" by "` (see the verified section — DataGrid scoping does NOT work).
  Only the now-playing bar is excluded positionally.
- **Named menu lookup.** "Add to playlist" is found by NAME, not by position,
  hovered before clicking, and the open is retried and verified. Committing
  SEARCHES: clear the retained query via the Clear button, type the name only
  once the box reports `HasKeyboardFocus`, then click the prefix match.
  Scrolling the unfiltered list is the fallback when focus can't be confirmed.
- **One scan, N adds.** The UIA scan happens once; the loop re-clicks the same
  element per playlist. Previously every playlist cost a full window scan.
- **Real mouse clicks, not `Invoke()`.** Spotify's handlers ignore UIA Invoke —
  the documented webview gotcha, and the original code learned it the hard way.
- **Per-playlist reporting.** The final tooltip names what succeeded AND what
  failed, rather than claiming success blindly.

## Voice commands

| Phrase | Does |
|---|---|
| `add song` | currently-playing track -> multi-select picker |
| `add song <n>` | Nth visible row -> multi-select picker |
| `add <playlist>` | **"add Charli"** — playing track straight into that playlist |
| `add song <playlist>` | same, longer form |
| `add <n> [<playlist>]` | Nth row; named playlist goes direct, no name opens the picker |
| `add playlist target` | register the open playlist as a target (+ optional spoken phrase) |
| `save playlists` | read playlists off this page, PICK which become targets |

Notes worth keeping:

- **`add playlist target` MUST stay declared above `add playlist <textnv>`** in
  the mapping. That Dictation sibling otherwise swallows the literal, matching
  it with `textnv="target"` and silently running the wrong function.
- **`save playlists`, not `sync playlists`** — `sync` isn't in the verb lexicon
  and `voice_index check` flags it. `save` is, and the phrase came back clean.
- **`add <option_number> [<playlist>]` was kept** (Jamie's muscle memory) but
  rewired onto this backend; it used to be a Dragon mouse-grid Playback chain.
- **`spot <track_number> <playlist>` was retired** — one playlist only, and it
  read the dead registry.
- `save playlists` is a **pick-list, not a bulk import**, because Jamie has a lot
  of playlists she never files into and importing them all would make the add
  picker useless.

## Retired

- `INIDATA/VoiceChoices/playlists.json` -> renamed `.retired` (migration reads
  either name, so `migrate-legacy` stays re-runnable).
- `AddSpotPlaylist` -> now a redirect to `AddPlaylistTarget`, and **moved** from
  `VoiceConfigManager.ahk` into `SpotifyPlaylistAdd.ahk`. It had to move:
  VoiceConfigManager is in the always-on curated `#Include` closure, and calling
  a Spotify-only helper from there fails `ahk_include_closure.py`.
- The `spot_playlist` entry in `add_voice_choice.py` — the Choice cache is
  DERIVED and must not be written through the generic registry path.
- `load_playlist_urls()` — always returned data nothing called; now returns `{}`.

## What the live page actually looks like (verified 2026-08-07)

The first cut was written without a live Spotify page to test against and got
several things wrong. Ground truth, from UIA probes of a real album page:

- **Track rows** are buttons named `More options for <Track> by <Artist>`. The
  page header carries `More options for <AlbumName>` — no " by " — so the
  presence of `" by "` is the discriminator. **Scoping the scan to the page's
  DataGrid does not work**: a DataGrid is found, but the buttons aren't
  reachable beneath it (that scan returned 0 rows on a page with 11 tracks).
- **The now-playing bar has NO more-options button.** Its buttons are Now
  playing view / Add to Liked Songs / shuffle / prev / play / next / Lyrics /
  Queue / Connect / Mute / Miniplayer / Full screen. What it does expose is a
  Hyperlink with the track title — **right-clicking that** raises the same menu
  (Add to playlist · Save to your Liked Songs · Add to queue · …).
- **`Now playing: <Track> by <Artist>`** is a Group name in the footer. That is
  the authority for what Spotify is playing. **Do not use SMTC for this** — it
  reports the system-wide session, and was observed returning a YouTube video
  while Spotify held a different track.
- **A menu row's accessible name has the containing FOLDER appended.** The row
  for "For Charli: Heavy Changes" reports as
  `"For Charli: Heavy Changes MIXTAPES"`; "Charli's Album Recs for Jamie <3"
  reports as `"... PLAYLISTS"`. **Exact matching therefore never hits** — match
  by PREFIX. This single fact made real playlists look nonexistent.
- **The submenu has a `Clear search field` button**, which is the clean way to
  reset the retained query — no keystrokes, so nothing can leak to the page.
- **The add-to-playlist submenu opens flakily.** The identical sequence opened
  it on one run and did nothing the next, so the open is retried and verified by
  the presence of the "Find a playlist" box.
- **~48 playlists render taller than the window**; ~20 sit below the fold with
  un-clickable coordinates, so the target is scrolled into view before clicking.

### The bug that made every add fail

**Spotify RETAINS the "Find a playlist" query across menu opens.** A stale word
silently filters the list, and the target playlist simply isn't in the tree —
indistinguishable from "that playlist doesn't exist". Reading the menu returned
all 45 playlists once and then 13 after a probe left `Stuff` in the box.

So the box is cleared at the start of every commit, **through the UIA
ValuePattern rather than Ctrl+A/Delete**. That distinction matters: when a focus
click misses, those keystrokes land on the PAGE and drive Spotify's own search,
navigating Jamie away from what she was looking at. Keystrokes survive only as a
fallback, and only once the box reports `HasKeyboardFocus`.

### Names must match Spotify exactly

The commit matches a menu row by exact name, so a drifted title fails. The six
migrated targets had **four wrong names**, inherited from the old hand-typed
registry — which means the *old* system was mistyping them too:

| Was | Actually |
|---|---|
| Random Songs | Random |
| Jess albums | Jess Albums <3 |
| Jess headphones | Jess Headphones <3 |
| I Do Believe You | I Do Believe You Gave It Your Best Try |

**Correction (later the same day):** `For Charli: Heavy Changes` and
`For Cedar Vol. 1` DO exist — an earlier read wrongly reported them missing
because the list was still filtered by a retained query AND because matching was
exact. See the folder-suffix note below.

This is why `save playlists` reads **Spotify's own submenu** rather than
scraping the sidebar: that menu is the exact set of valid targets, spelled the
way the commit step needs them.

## Verified end-to-end (2026-08-07)

Voice `add song` -> picker -> commit was driven through the whole chain on a
live page: Dragon heard it, `AddCurrentSongToPlaylists` dispatched, the picker
opened showing the right track, and `AddSongToPlaylistNamed 0 Random` really
added the playing track to **Random**, with the add-history tick showing on the
next open.

Still unverified: `save playlists` end-to-end (the reader it depends on IS
verified — it returned all 45 playlist names), and adding to more than one
playlist in a single pass.

If an add fails, the tooltip says which step: "playlist menu didn't open"
(submenu), "No playlist named X" (name drift -- check it against
`_SpPlReadPlaylistNames`), or "Couldn't scroll X into view".

## Engine CLI

```
list [--targets-only] [--json]        rows (targets first, then alphabetical)
add-target --title T [--url U] [--phrase P]
remove-target --id ID                 unflag; catalog row survives
set-phrase --id ID [--phrase P]       blank clears; rejects a duplicate phrase
migrate-legacy [--commit]             playlists.json -> music.json
parse-sidebar --file DUMP [--report]
sync-choices                          rebuild the voice cache by hand
log-add --id ID --track "Artist - Song"
history --track "Artist - Song" [--json]
```
