---
tags: [programming, design-doc, acquire, torrents, jackett, qbittorrent, flaresolverr, music, tv, film, plex, cd-burning, miller, settings]
---

# Acquire — search, download, burn

Status: **live (2026-07-31).** Search → download → unpack → file → Spotify link
→ burn-queue is built and **verified end to end on a real download**: one
keypress produced a filed, catalog-linked album with no further interaction,
with qBittorrent's own log confirming the completion hook fired. The read-only
disc preflight is verified against the real USB burner.

The IMAPI2 disc **write** is verified against real hardware too, as of the same
day. It had been left untested on purpose — Jamie keeps no rewritable discs, so
every attempt risks a physical CD-R — and when it finally ran it failed twice
before working. Both failures cost nothing, because IMAPI rejects a bad track
before any data reaches the media. See failure #7 for what they were and for
the disc-free way to test the first one.

Entry points: **"open torrents"** (the Miller) and **"torrent \<query\>"** (search
straight into it, via the shared search registry).

Deliberately **content-agnostic**. Music is the main use, but the same path
serves CS-research datasets, books, and video; nothing in the pipeline knows
what the bytes are. Sources are a late concern, not the organizing principle.

## The stack

```
Acquire Miller (AHK)          Helpers\AcquireMenu.ahk + Scripts\AcquireViewer.ahk
      ↓ shells
acquire.py / burn.py          Scripts\acquire\
      ↓ HTTP (localhost only)
Jackett  :9117  ──→ FlareSolverr :8191   (Cloudflare-protected indexers)
qBittorrent WebUI :8080
```

| Piece | Where | Notes |
|---|---|---|
| Miller UI | `Helpers\AcquireMenu.ahk` | viewer trio via `new_miller.py`; own process |
| Viewer host | `Scripts\AcquireViewer.ahk` | include closure matters — see the trap below |
| Search / download | `Scripts\acquire\acquire.py` | Jackett Torznab + qBittorrent WebUI |
| Library sorting | `Scripts\acquire\library.py` | **move** (not hardlink) + catalog matching + the music guard |
| Unattended chain | `Scripts\acquire\pipeline.py` | armed + auto-file; driven by qBittorrent's completion hook |
| Job record | `INIDATA\acquire_pipeline.json` | what was filed where, and what was deliberately left alone |
| Catalog write | `clog media-set-local-path` | the ONLY writer of `local_path` |
| Burning | `Scripts\acquire\burn.py` | ffmpeg decode + IMAPI2 |
| Burn queue store | `INIDATA\acquire_burn_queue.json` | plain list of file paths |
| Window suppression | `Scripts\acquire\qbt_quiet.py` | keeps an auto-started qBittorrent off screen |
| Maintenance CLI | `~\.claude\scripts\jackett.ps1` | `jackett status` / `update` |
| Settings | `acquire.*` | 20 keys, all read by AHK (or by pipeline.py via `setting()`) |
| FlareSolverr launch | `Startup\FlareSolverr-AutoStart.vbs` | **hidden**; see below |
| Search registry entry | `INIDATA\VoiceChoices\searches.json` | `torrent` → `AcquireSearch` |

## Voice: reusing the search registry rather than a new command

`searches.json` already backs `<search_target> <textnv>` (amazon, plex, books,
pirate…). A provider of `type: "function"` calls `FnName(query)`, so adding a
torrent search was **one JSON entry** plus one AHK function — no new rule file,
no reboot. Saying *"torrent bach brandenburg"* runs `AcquireSearch(query)`.

The query travels to the viewer through a temp file, not a command-line
argument: it is dictated text and can contain quotes and apostrophes that would
otherwise need to survive escaping through cmd, AHK **and** Python in turn.
`AcquireSearch` also closes an already-open viewer first — `LaunchMillerViewer`
is reuse-or-spawn, so without that a second search would silently re-show the
previous results.

`open torrents` opens the Miller with no search (assigned via the generic store,
so it went live in ~5s without a Caster reboot).

## Why Jackett rather than qBittorrent's own search plugins

qBittorrent's `nova3\engines\*.py` plugins each know exactly one site. Jackett
knows ~655 and normalizes them all to Torznab, so adding a source is a config
change instead of a new plugin. The standalone `rutracker.py` plugin is
**disabled** (`rutracker.py.disabled-by-claude`) because Jackett's RuTracker
indexer supersedes it and the plugin could no longer log in at all.

## Twelve failures that cost an afternoon — read before debugging

### 1. Defender silently killed Jackett every day

Defender flags `JackettUpdater.dll` as `Trojan:MSIL/Barys.ABR!MTB` — a
machine-learning heuristic, and a false positive by every available signal (all
16 detections were this one file; the machine had no other detections). An
auto-updater that kills processes and overwrites files in `ProgramData` simply
looks like malware to the classifier.

It deadlocked: the installed `JackettUpdater.dll` was already gone (its `.exe`,
`.pdb` and `.deps.json` siblings were all still present — only the `.dll` was
missing, which is fatal because in a .NET apphost layout the `.exe` is a thin
launcher and the `.dll` holds the code), **and** every freshly downloaded
replacement was quarantined on arrival in `C:\Windows\SystemTemp`. So Jackett
exited to let the updater run, the updater died instantly, and nothing restarted
Jackett. Down since 2026-07-29 09:38; symptom was qBittorrent showing
*"connection error getting indexer list"*.

**Fix, and the standing invariants:**
- `C:\ProgramData\Jackett` is a Defender exclusion → the installed copy is safe.
- `C:\Windows\SystemTemp` is **deliberately not excluded** — far too broad.
- Therefore **auto-update stays off** (`UpdateDisabled: true`) and updates are
  manual via `jackett update`. Re-enabling auto-update restarts the whole loop.
- Jackett still needs updating periodically or indexer definitions decay.

### 2. RuTracker 403s every scripted request

`index.php` returns 200, but `login.php` and `tracker.php` return **403 to any
script regardless of user agent** — verified with both the plugin's ancient
Firefox 38 string and a current Chrome one. Jamie's credentials are fine; she
can log in via browser. Cloudflare is fingerprinting the client (TLS/HTTP2), not
checking the UA.

FlareSolverr (headless Chromium, `E:\Tools\FlareSolverr`, bound to
`127.0.0.1:8191`) solves the challenge and Jackett proxies through it —
`login.php` then returns 200. It is launched from `StartupOrder.bat`.

**It is needed to LOG IN, not to search** — an earlier version of this doc
called it a hard dependency for every query, which measurement disproved on
2026-07-31. With FlareSolverr fully stopped, a *fresh uncached* query still
returned 17 RuTracker results. The reason is in Jackett's own config:
`C:\ProgramData\Jackett\Indexers\rutracker.json` stores a ~600-character
`cookieheader`. Once that session exists, ordinary searches ride it and clear
Cloudflare unaided. FlareSolverr is what obtains the session in the first place.

Beware of testing this the lazy way: Jackett caches for 35 minutes
(`CacheTtl: 2100`), so re-running a recent query with FlareSolverr stopped
returns byte-identical results and *looks* like proof of independence. Use a
query the cache has never seen.

So the practical rule is: FlareSolverr must be up when the session needs
establishing or refreshing (empirically every week or two), and is idle
otherwise. Keeping it running costs ~48 MB RSS and effectively no CPU, which is
why it stays on rather than being started on demand. When the session *does*
lapse without it, Cloudflare-backed indexers return *nothing* — not an error,
just an empty result set, which reads exactly like "no matches."

Because that failure is silent by nature, it is surfaced in three places rather
than left to be discovered: `acquire.py search` probes FlareSolverr on **every**
search and emits a `WARN` row when it is down; the Miller turns that into a
"⚠ Results are incomplete" node at the root plus a 5-second tooltip; and
`jackett status` reports it. A degraded search now says so instead of looking
like a query with no matches.

**It runs hidden.** `flaresolverr.exe` is a console app, so launching it from
`StartupOrder.bat` left a terminal window on screen all session.
`FlareSolverr-AutoStart.vbs` runs it with `WScript.Shell.Run(..., 0, False)` —
same pattern as `QMD-Daemon-AutoStart.vbs` — and pins `HOST=127.0.0.1` so it
binds loopback only (the default is `0.0.0.0`, which would expose a
browser-driving service to the whole LAN).

There was also a genuine bug in the old plugin worth remembering, because the
same shape appears elsewhere: `rutracker.py:261` checks only that a cookie
*named* `bb_session` exists — the expiry comparison on the next line is
**commented out** — and returns early, so a stale cookie file meant `login()`
was never called again. A dead session locked the plugin out permanently while
looking perfectly configured.

### 3. qBittorrent had no autostart, so every reboot half-broke the system

Symptom (2026-07-31): search returned 8 results normally, then Enter produced
*"could not reach qBittorrent at http://127.0.0.1:8080: `<urlopen error
[WinError 10061] No connection could be made because the target machine actively
refused it`"*.

The wording sends you looking for a network or firewall fault. It isn't one —
10061 on loopback means **nothing is listening on that port**, i.e. the app is
simply not running. Jackett is a Windows *service* and FlareSolverr has a
Startup `.vbs`, so both survive a reboot; qBittorrent had **no autostart entry
anywhere** — not the Startup folder, not `StartupOrder.bat`, not `HKCU\...\Run`.
That asymmetry is what makes the failure confusing: the half of the system that
comes up on its own works perfectly, so search succeeds and only the *add* dies.

**Fix:** `ensure_qbt()` in `acquire.py` probes `/api/v2/app/version` (3 s) before
any qBittorrent call and, if refused, launches `qbittorrent.exe` detached and
polls up to 30 s for the WebUI. Measured cold start to a served request: **~3
seconds**. Both `add` and `active` route through it; `active` matters because
opening the Miller is usually the first thing that touches qBittorrent.

> Guarding only `acquire.py`'s own commands turned out to be half a fix — the
> same error resurfaced a week later through `pipeline.py`. See **failure #11**.

Two guards worth keeping: it only auto-launches when the base URL is
**localhost** (starting a local client would silently target the wrong machine
if the WebUI were ever pointed remotely), and it launches with
`DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP`, because the caller is a
short-lived CLI spawned from an even shorter-lived AHK process and qBittorrent
must outlive both.

Note the WebUI binds *before* resume-data checking finishes, so a cold start
answers the API while every torrent still reads `checkingResumeData` at 0%.
That is normal and resolves on its own — don't read it as corruption.

**Starting it must not put anything on screen**, and qBittorrent will not
cooperate. Measured on 5.1.4:

| Setting | Written? | Honoured? |
|---|---|---|
| `General\StartMinimized=true` | yes, qBittorrent preserves it | **no** — window shows anyway |
| `General\MinimizeToTray=true` | yes | yes, but only for a *later* minimize |
| `General\CheckForUpdates=false` | yes | **no** — the update dialog still appears ~10 s in |
| `--no-splash` | n/a | yes |

`StartMinimized` being broken on Windows is upstream and long-standing
(qbittorrent/qBittorrent#14451, #3656, #12943; #23318 requests a `--minimize`
flag *because* the preference is ignored). Do not spend time re-deriving this —
the settings are written anyway, harmlessly, so they start working for free if
upstream ever fixes them, but nothing may depend on them.

The window is therefore handled from outside by `qbt_quiet.py`, aimed at the pid
`ensure_qbt` spawned so a qBittorrent Jamie opened herself is never touched. It
runs detached (the update dialog appears long after the WebUI is usable, and
`add` must not wait on it) and polls for ~25 s. Dialogs are **closed** before the
main window is **minimized** — the update dialog is modal to the main window, so
minimizing first strands it on screen alone. Minimize rather than `SW_HIDE`,
because with `MinimizeToTray` set qBittorrent's own handler takes it to the tray,
a state it understands and can restore from.

Verifying this needs `IsIconic`, not `IsWindowVisible` — a minimized window is
still "visible" to Win32, which makes a naive check report failure on a window
that is correctly in the tray.

### 4. `subprocess` + `text=True` decodes with cp1252, not UTF-8

Symptom (2026-07-31): arming a download died with

```
Could not start it — Exception in thread Thread-1 (_readerthread):
  File "...\Python312\Lib\threading.
```

which names a thread and a stdlib file and says nothing about the actual
problem. The real error, several frames down:
`UnicodeDecodeError: 'charmap' codec can't decode byte 0x90`.

**`text=True` decodes child output using the LOCALE codepage** (cp1252 here),
but every script in this stack calls
`sys.stdout.reconfigure(encoding="utf-8")`. The mismatch only shows up when a
release title contains a non-ASCII character — and music titles are full of
them. The one that broke it was a bullet: *"Masterpiece • Obra Maestra"*. The
decode happens on subprocess's internal reader **thread**, so the traceback
names `_readerthread` instead of the call site.

**Every `subprocess.run` in this stack passes
`encoding="utf-8", errors="replace"`.** There were seven, all latent. The worst
was not the one that broke: `probe_tags()` reads **ffprobe's tag output**, where
non-ASCII is the norm — and its `except Exception` would have swallowed the
error and returned no tags, silently falling back to folder-name guessing for
exactly the classical and non-English releases the tag reader matters most for.

Rule for anything added here: `text=True` alone is a bug. Always name the
encoding.

### 5. qBittorrent's add endpoint cannot tell you what it added

`/api/v2/torrents/add` answers a bare `"Ok."`. Not a hash, not a name — and it
answers `"Ok."` for a **duplicate** exactly as it does for a new torrent. So
"which torrent did I just add?" is genuinely unanswerable after the fact.

The first version of `arm` diffed qBittorrent's torrent list around the add and
took the new hash. That works exactly once per release. Add the same one twice
and the second add is a no-op, no new hash appears, and the diff loop waits out
its full 25-second timeout before reporting that it "could not identify" the
torrent — while the download from the *first* attempt is running perfectly.
Jamie hit precisely this: attempt one crashed on the encoding bug *after* the
add had already succeeded, so attempt two was a duplicate.

**Resolve the infohash BEFORE adding** (`acquire.py infohash <index>`):
- magnet → parse `xt=urn:btih:` straight out of the URI
- Jackett `/dl/` link → these 302 to a magnet, so the hash is in the redirect's
  `Location`. **urllib refuses to follow a redirect to a non-HTTP scheme and
  raises**, so the header must be read off the `HTTPError`, not the response.
- real `.torrent` → SHA1 of the bencoded info dict (`_bencode` re-encodes with
  sorted keys, which valid torrents already use, so it round-trips exactly)

That turns a 25-second failure into a 0.8-second success, and makes "already in
qBittorrent" a first-class answer: the existing download gets armed instead of a
second one being started.

Related UI rule: **`OK` plus `WARN` is not a failure.** The tooltip keys off
whether an `ARMED` line came back — "Armed:" when the job was recorded,
"Started:" when the download is running but the arming was degraded. The old
code printed "Could not start it" over a torrent that was already downloading,
which is the most confusing thing a message can do.

### 6. Unattended work must be seen but never *seen doing it*

Two opposite complaints from the same run, both fair:

**Command prompts flashing in succession while filing.** Windows gives every
console-subsystem child its own console window unless told otherwise, and this
pipeline is nothing but console programs — worse, `guess_album()` shells
**ffprobe once per track**, so a 12-track album flashed a dozen prompts.

- Every `subprocess` call in `acquire/` passes `creationflags=CREATE_NO_WINDOW`
  (`0x08000000`). `DETACHED_PROCESS` is **not** sufficient: detaching stops a
  child *inheriting* a console, it does not stop Windows *allocating* one.
- The hook itself is spawned by qBittorrent, so its creation flags are not ours
  to set. The only lever is which interpreter: the autorun command points at
  **`pythonw.exe`** (GUI subsystem, no console). `py.exe` and `python.exe` both
  get a window.

**"No indication of what is going on."** The opposite failure: the only feedback
was a single 6-second tooltip at the very end. It fired correctly — the log
proves it — but it landed while she was mid-sentence in another window, which
is functionally invisible. The whole premise of the pipeline is that she walked
away, so anything it says has to survive not being watched for.

- A tooltip **before** the slow step, not only after it. Filing takes long
  enough that silence reads as nothing happening.
- Terminal messages run 10–15 s and state **where it went** and **what to do
  next**, e.g. "Queued for a CD — open Acquire ▸ Burn queue to write it".
- Every refusal-to-burn says *why* and where to finish it by hand.
- Deliberate no-ops stay silent: a completed TV download is skipped by design,
  and a tooltip for each one would be noise. That state is visible in the
  Downloading list instead.

### 7. What the first real burn actually caught

The IMAPI2 write was written from the contract and never executed. Two genuine
bugs surfaced on the first attempt, and **neither cost a disc** — both failed
before any data reached the media, because `PrepareMedia()` claims the disc but
does not write and the `except` calls `CancelAddTrack()`. The disc read back
blank and READY after every failure.

**a. `AddAudioTrack` needs a COM interface pointer, not a ctypes one.**
```
argument 1: AttributeError: 'LP_c_void_p' object has no attribute 'QueryInterface'
```
comtypes marshals the argument by calling `QueryInterface` on it. A raw
`ctypes.POINTER(ctypes.c_void_p)` has no such method. Declare
`SHCreateStreamOnFileEx`'s out-parameter as `POINTER(comtypes.IUnknown)`; IMAPI
QueryInterfaces it to `IStream` itself.

*Testing this without a disc:* call `AddAudioTrack` **without** `PrepareMedia`.
A type bug still raises `AttributeError`; a correct type gets rejected with
`"only valid when media has been prepared"` — a state error. That distinction
is free and needs no media.

**b. Audio must be a whole number of CD sectors.**
```
The provided audio stream is not valid.
```
IMAPI takes audio one 2352-byte sector at a time and rejects anything else with
that message — which says nothing about length being the problem. ffmpeg emits
exactly as many samples as the source holds, which is almost never a multiple:
track 1 was 20,847,936 bytes, i.e. 2160 over. `decode(raw=True)` now pads with
silence to the next boundary; the most it can ever add is 2351 bytes, under
14 ms, at the very end of a track.

**Staging is not a prerequisite for burning.** `cmd_burn` decodes each track
into its own temp stream, so staging first decodes the whole album twice for
nothing. Staging exists to hand a WAV+CUE set to an external burner — a
different job. The unattended `--burn-now` path no longer stages.

**Enter on a disc BURNS it.** It used to stage, with the burn hidden behind
`9`. That cost a real round of confusion: pressing Enter on "Disc 1" says
"Decoding…", which looks exactly like a burn starting, so "I told it to burn"
and "nothing happened" were the same keypress. Jamie's verdict was blunt and
correct — *"if I press enter on the line that says burn I want to burn"*.

The safeguard was moved rather than removed, because a hidden key is not a
safeguard: burning runs the read-only preflight and shows a confirm box quoting
its verdict, which is something you can actually read and refuse. Decode-only is
now action `2` — it is the rare case (feeding an external burner), not the
default.

**The burn window IS the progress UI.** `burn.py` narrates every step flushed
per line — preflight, claiming the disc, decode/write per track, and the
finalize — and the Miller runs it in a PowerShell window rather than a hidden
`RunWait`. The two stretches that previously looked like a hang are the per-
track decode (~10–20 s) and `ReleaseMedia` at the end (lead-out + TOC, which can
sit for a minute with no output at all); both now announce themselves before
they start. The window stays open on success as well as failure — one that
closes the instant it finishes is one she can miss entirely.

### 8. An auto-refresh that moves the selection is a destructive bug

Adding `refresh_seconds` to the Miller so download speeds update on their own
looked harmless. It applied to **every** level, including static ones, and each
re-render threw the selection back to row 1.

That is not merely annoying. A list that jumps under a keypress lands that
keypress on **a different row than the one being looked at** — and in the Burn
queue one of those rows is "Clear queue". It wiped a queue mid-pipeline, and the
unattended burn that followed died with "burn queue is empty" on a download that
had otherwise worked perfectly. The visible symptom ("my download failed") was
three steps removed from the cause.

Both halves are required, and either alone is insufficient:
- **Scope it.** `refresh_paths` names the root keys whose subtrees are live
  (`["active", "pipeline"]` here). Static levels are never re-rendered.
- **Preserve the caret.** The refresh sets `state["restore_sel"]` to the focused
  row first; `_doRender` honours it in the same pass, so there is not even a
  flash of row 1.

General rule for any live-updating list: if a refresh can move the selection,
every destructive action reachable from that list is now reachable by accident.

### 9. An unattended burn must burn only what it just filed

`--burn-now` used to add the filed album to the burn queue and burn *the queue*.
Caught in the act: a queue already holding Masterpiece's 12 tracks got
Capacity's 11 added, and 23 tracks across two unrelated albums still fits inside
79 minutes — so it would have burned a mixed disc without anything looking
wrong. The one-disc check passed, because it *was* one disc.

`_finish_by_burning` now snapshots the queue, replaces it with exactly the album
being burned, and restores the snapshot in a `finally`. The restore is
try/finally rather than a call before each `return` because the function has
four exits and a fifth would eventually be added without one — silently leaving
Jamie's hand-built queue replaced by a single album.

### 10. A test suite that cannot report failure

`RunGuiTests` wrote `OK|RunGuiTests|<time>` to `ahk_last_result.txt` whether the
suite passed or not — the pass/fail count existed only in a tooltip and the
event log. Anything reading the result file, human or otherwise, saw OK and
believed it. That is exactly how a "full GUI suite OK" claim got made over a run
with two failures in it.

It now throws when the summary reports a non-zero failure count, so the
dispatcher writes `ERROR|` and shows a tooltip. Verified both directions.

The failures it had been hiding: `dumps_menu_opens_and_valid` was **real and
pre-existing** — the fixture calls `_DumpPresetsAsNode`, which lives in
`DumpPresetsMenu.ahk`; the real app resolves that through GuiHost's closure, but
a fixture builds its own and `ahk_include_closure.py` does not check
`Fixtures/`. Same shape as the viewer trap: compiles clean, throws at runtime,
and surfaces only as "menu-invalid". The other two were load flakes.

### 11. A fix that only guards the door it was written for

Symptom (2026-08-01): the *exact* WinError 10061 from failure #3 came back, on
the one-click download-file-burn action. `ensure_qbt()` had been working for a
week, and qBittorrent really was down — it exits when Jamie shuts down, so it is
down most mornings.

`ensure_qbt()` lives in `acquire.py` and was wired into `acquire.py`'s own
commands — `add` and `active`. But **`pipeline.py` and `library.py` each speak
to the WebUI directly**, with their own `qbt_get`/`urlopen`, and neither had ever
called it. `pipeline.py arm` reaches the WebUI *before* it shells out to
`acquire.py add`: it checks `hook_installed()`, and on a False installs the
autorun hook. So the chain died at its first request, several steps before the
one function that knew how to start the app.

The tell was in the message. #3's read `could not reach qBittorrent at …`;
this one read **`could not read qBittorrent preferences: …`** — the `install`
path, not the `add` path. Same socket error, different sentence, different file.
Reading past the WinError to the prefix is what located it.

Two things made the gap easy to miss:

- `hook_installed()` swallows its exceptions and returns False. "qBittorrent is
  down" and "the hook is not installed" are indistinguishable to the caller, so
  a dead client looks like an ordinary first run and the code confidently tries
  to configure it.
- The broken path was the *newest* feature. `add` was hand-tested cold many
  times; `arm` was only ever exercised while qBittorrent happened to be up.

**Fix:** `wake_qbt()` in `pipeline.py` imports `ensure_qbt` from `acquire.py`
(import-safe — everything is behind `if __name__ == "__main__"`) and runs at the
top of `arm`, `install` and `uninstall`. `library.py cmd_finished` got the same
guard: the Miller calls it just to *browse* what is waiting to be sorted, so
browsing was dying on a socket error too. Deliberately **not** added to
`complete`/`run` — qBittorrent invokes those itself, so it is running by
definition, and launching a second copy from a detached completion hook would be
a bad way to be wrong. Verified cold, from an actually-stopped qBittorrent:
`install` 3.8 s, `finished` 3.2 s, both answering normally.

**The rule:** when a helper exists because a *dependency can be absent*, the
guard belongs at every entry point that touches it, not on the one command that
first exposed it. Grep the whole surface for direct calls the day you write it —
`qbt_get`, `urlopen`, `_qbt_post` — rather than trusting that everything routes
through the polite wrapper.

That sweep (`grep -n "/api/v2" *.py`) leaves exactly two call sites deliberately
unguarded; do not "fix" them:

- **`acquire.py cmd_doctor`** — its whole job is to report UP/DOWN. A doctor that
  starts the patient before taking a pulse has nothing to report.
- **`library.py stop_torrent` / `remove_torrent`** — they already return a
  message instead of raising, and they run at the *tail* of a sort whose
  `finished` listing woke the client. By then the album is filed; failing to stop
  seeding is a loose end, not a lost record.

### 12. "It's not working" was three unrelated things at once

Symptom (2026-08-01): an armed download sat at 0% with a 40-character hex string
where its name should be. Reported as *"it seems like it's still not working,
the name of the torrent is weird"*. Three separate causes, none of them the add:

**a. The name was never wrong.** The Jackett link redirected to a magnet, and a
magnet has no name until its metadata arrives — so qBittorrent displays the
infohash. Normal, but it reads as corruption. We *do* know the title (the
pipeline stores it at arm time), so `_AcqIsHashName()` detects a bare infohash
and the Downloading list substitutes the armed title.

**b. The swarm was dead, and nothing said so.** `state=metaDL`, 0 seeds, 0
peers, `total_size=-1`, indefinitely. TorrentDownload advertised **16 seeders**;
ten public trackers plus DHT found **zero** in two minutes. Indexer seed counts
are marketing, not measurement. There is no error and no timeout for this — the
torrent just sits — so `cmd_active` now emits a **peer count** (field 10) and
the list says `no peers found` instead of a truthful, useless `0 KB/s`.

Worth knowing: `add_trackers_enabled` is **off**, so bare magnets get only the
one tracker the indexer supplied. Turning it on would give them a fairer chance,
but it appends trackers to *every* torrent, which is the wrong thing to do to a
private tracker — decide deliberately, do not switch it on reflexively.

**c. Looking at qBittorrent killed the downloads.** Because we launch it hidden,
the natural way to see it is to run the exe again — the single-instance handoff
raises the existing window. Closing that window **quit the whole app**, taking
every download with it. The event log shows exactly that: a second instance at
12:18:01, then `CloseWindowOrTab()` at 12:18:16, then nothing running.
`General\CloseToTray=true` is now set in `qBittorrent.ini`.

**A testing note that nearly produced a wrong conclusion:** the first check of
that setting sent `WM_CLOSE` to the main window and the process died — "the ini
key is ignored, same as `StartMinimized`". Wrong. The window was still *hidden*
(qbt_quiet had minimized it), and closing an already-hidden window takes the
quit path. Repeating it against a normally visible window showed the process
surviving and the window hiding to tray — the setting works. Same shape as the
`IsWindowVisible`/`IsIconic` mistake in failure #6: **when testing a
window-state behaviour, put the window in the state a human would actually
produce.** A programmatic shortcut to that state can silently change the answer.

## Swarm checking — why the seeder column cannot be believed

Every indexer reports a seeder count and **it is frequently fiction.** Measured
on 2026-08-01 across 36 results for one album: **23 had no swarm at all**,
including entries advertising 38, 16 and 12 seeders. Two of those were armed for
download on the strength of that number and sat at 0% indefinitely.

Nothing about that failure looks like a failure. qBittorrent accepts the
torrent, reports `metaDL`, and waits — because "nobody on earth has this file"
is not an error condition in BitTorrent, it is just silence. There is no
timeout, no retry limit, and no state that ever becomes `failed`.

**The fix is to ask a tracker instead of an indexer.** A UDP scrape (BEP 15) is
two round trips and answers with the tracker's own count for that infohash:

```
acquire.py peers <index>          # a cached search result
acquire.py peers --hash <infohash>
→ ALIVE   13 seeders / 0 leechers (asked 5)
→ DEAD    no seeders on any of the 4 trackers that answered
→ UNKNOWN no tracker answered — swarm health could not be checked
```

Three design rules, each learned the hard way in one sitting:

1. **Refuse only on positive evidence of death** — at least one tracker
   answered, and every tracker that answered said zero. `answered == 0` means
   the check failed, not that the torrent is dead. A sanity check that blocks
   good downloads whenever the network hiccups is worse than no check.
2. **Retry the all-silent round.** UDP is lossy. Six consecutive runs answered
   3/4/5/3/6 of six trackers — and one answered none, which let a known-dead
   torrent through, because "unknown" has to fail open. Retrying only that case
   is free on the normal path and made 8/8 subsequent attempts refuse correctly.
3. **Say when the check did not run.** Silence is indistinguishable from a pass,
   which is exactly what makes a safety net untrustworthy. The first version
   rendered an unchecked result as a *blank row*, sitting directly under the
   indexer's claim of 19 seeders — the precise failure this feature exists to
   abolish, reintroduced by the feature itself. Unknown now renders as
   `could not check — <reason>`.

Getting the infohash is the expensive half, not the scrape. Three sources, in
order of cost: the Torznab result often carries `infohash` outright (free, but
only ~1 in 10 results have it); otherwise the indexer's link is fetched and
either the magnet redirect or the .torrent's info dict is hashed (~7 s); and
**1337x cannot be resolved at all** — Jackett has to scrape the site to produce
the link, and it times out past 25 s. Those results honestly report
`could not check` rather than being silently skipped.

Where it runs:

| Point | Behaviour |
|---|---|
| `acquire.py add` | refuses a dead torrent before qBittorrent is touched (~3 s) |
| `pipeline.py arm` | same, using the hash it already resolved — saves re-fetching the link (~7 s) |
| Search | spawns `swarmscan` **detached**; verdicts for all results cached in ~4 s |
| Detail pane | reads the cache only, **never blocks** — `Health` (claimed) sits directly above `Swarm` (measured) |

The detail pane is cache-only for a measured reason: computing it inline cost
**10.7 s on first view**, and that pane redraws while arrowing down a list. The
slow part is resolving the infohash — fetching the indexer's link — not the
scrape. So the work happens once, in the background, right after the search.
Failures are cached for 120 s rather than not at all, because not caching them
meant re-fetching on every redraw for exactly the results least likely to answer.

Setting: `acquire.check_swarm_before_add` (default on), `--no-peer-check` to
override per-invocation.

**The general lesson:** when a remote system reports a number that drives a
decision, ask whether anything ever verifies it. This one had been believed
without question since the system was built, and the failure mode it produces —
indefinite silence — is the hardest kind to attribute.

### Where the verdict is stored, and why not by link

The first version cached verdicts in files keyed by a hash of the result's link.
It appeared to work and was almost entirely useless: **Jackett mints a fresh
`/dl/` link for every query**, so the next search for the same album produced
different keys and every row came up unmarked. Only Torrents.csv and other
magnet-serving indexers ever hit the cache, because a magnet URI is stable.

Verdicts are now written back into the result cache itself (`live` for the list
column, `live_note` for the detail sentence). That is the file the list and the
pane already read, so there is no second lookup and no keying problem.

Two consequences worth knowing:

- The scan **re-reads the cache before writing**. Jamie may have started another
  search while it was running, and stamping stale verdicts onto new rows would
  mislabel them. The link is the identity check.
- The Miller re-reads rows via `acquire.py results` on every render of the
  results level, rather than keeping the rows it parsed at search time. Rows
  parsed at search time can never show a verdict, because the scan finishes
  *after* the search returned. Re-running the search would be worse than
  useless: it overwrites the cache with unstamped rows.

## Search speed — one indexer was most of it

Searches took ~30 s and the cause was a single source. Measured on two uncached
queries (Jackett caches for 35 min, so a repeated query proves nothing):

| | |
|---|---|
| `/indexers/all/` aggregate | **12.8 s** |
| 1337x alone | **12–14 s** |
| the other eleven, each | **≤ 2.5 s** |

Jackett's aggregate endpoint already queries indexers in parallel — but it
answers only when the slowest one does, and offers no way to cap a single
indexer. So every search ever run had been paying 1337x's latency in full.

`fetch_indexers()` now does the fan-out here instead: each configured indexer is
queried separately and concurrently with a per-indexer deadline
(`acquire.indexer_timeout`, default 5 s). A slow source costs that deadline, not
the whole search, and is named in a `WARN` row rather than silently dropped.
Measured after: **6.3 s at a 6 s cap**, i.e. the search now finishes as soon as
the cap expires rather than whenever 1337x feels like answering.

Two safeguards, both learned by breaking them:

- The indexer list comes from `/api/v2.0/indexers`, which is an **admin route** —
  the api key alone gets a 302 to the login page, so it needs the
  cookie-carrying opener from `jackett_session()`. Getting that wrong made the
  fan-out find zero indexers.
- If the fan-out yields nothing **for any reason**, it falls back to the
  aggregate endpoint. The first version guarded that fallback on "and no
  warnings", so one slow indexer suppressed it and the search returned empty —
  turning a speed optimisation into a total failure.

## Judging a result without opening it

The results list shows six columns: `Release / Fmt / Size / S / Live / Source`.

- **S** is the indexer's claim. **Live** is the measured seeder count, or `dead`,
  or `?` when it could not be checked. They disagree constantly — one search
  showed a row claiming 31 seeders that really had 12, and rows claiming 19 and
  13 that had none.
- **Source** is there because indexer quality varies enormously and that is only
  learnable by seeing which source a result came from. On the evidence so far,
  TorrentDownload inflates heavily, and Torrents.csv and The Pirate Bay report
  close to the truth.

The results row also survives closing the viewer now: the root seeds its rows
from the cache when the process has not run a search itself. Searches are the
slow part, so re-running one to recover results still sitting on disk was the
expensive way to get back something never actually lost.

## The Miller

Root: Search… · Results (N) · Downloading · Indexers · Burn queue · Health · ⚙ Settings.

Results render as a table (`cols`: name / format / size / seeders), sorted by
`acquire.format_priority` first and seeders second, so FLAC surfaces above MP3
without reading titles. Enter downloads; `N.2` adds paused; `N.3` copies the link.

**Links never touch AHK.** Magnets routinely exceed 2000 chars and Windows
command lines die at ~8191, so `acquire.py search` writes the full result set to
a JSON cache and prints only short rows carrying an index; `add`/`link` re-read
the cache by index.

**Burn safety:** on a disc row, Enter *stages* (decodes to WAV+CUE on disk —
harmless, repeatable) and `N.9` *burns*, behind a confirm. Writing a disc is
physical and irreversible; it should never be one stray Enter away.

## What to do with a finished download

Every finished row under **Downloading** drills into its own action list —
file it, file-and-queue-to-burn, queue as-is, open the folder. This lives here
rather than only under Library because Downloading is where you look when
something completes; having the next step be somewhere else read as "there is no
next step".

**Order matters, and the UI encodes it.** Filing MOVES the files (stop seeding,
then move — deliberately not hardlinks). So queueing a burn from the download
path and *then* filing leaves the queue pointing at a folder that no longer
exists. "File it, then queue it to burn" is one action precisely so that trap
cannot be stepped in: it files first, reads the `DEST` line back, and queues
*that* path. `_AcqReportSort` returns the destination for exactly this reason.

Queue-as-is is still offered — burning something without filing it is a
legitimate one-off — but its description says what it costs.

## The unattended pipeline (`pipeline.py`)

Row actions `4` and `5` on a search result arm the whole chain: download →
unpack → file → (optionally) queue for burning. Root node **Automatic** shows
what is in flight.

**Completion is qBittorrent's own "run external program on finished" hook**, set
through the WebUI API (`autorun_enabled` / `autorun_program`) by
`pipeline.py install`, which `arm` calls automatically if it isn't set. There is
no polling daemon: nothing to supervise, nothing to restart after a reboot, and
no window where a completion is missed because the watcher was down.

Two consequences of that hook being **global** — it fires for *every* completed
torrent, not just armed ones:
- `complete` looks the hash up in the job store and exits 0 silently when it is
  not ours. That is the common path and must stay cheap and quiet.
- `install` refuses to overwrite an `autorun_program` that isn't ours rather
  than clobbering an unrelated hook.

`complete` hands off to a detached `run` and returns immediately, because
qBittorrent is waiting on it and the real work takes minutes.

### Auto-filing (the default path)

With `acquire.auto_sort` on — it is by default — **every** finished download is
considered, not just armed ones. If it is confidently music it is filed, linked
and recorded without anyone pressing anything. That is the 99% case: Jamie
downloads an album and it is simply in the library afterwards.

The safety rule is asymmetric on purpose. `music_confidence()` in library.py
judges by **byte share, not file count** — a season of TV carrying one bonus
track has a negligible audio count but is obviously not an album, and a single
45-minute classical FLAC is one file but obviously is. Any real video content
(>10% of bytes) disqualifies outright, as does dominant ebook or software
content. A false positive drops a TV rip into the music library and Jamie has to
notice and undo it; a false negative just means she files it by hand, which is
what she did for everything before this existed. **When unsure, refuse.**

Verified against her actual download list: the Big Thief FLAC filed at 99%
audio; five TV rips, an ebook and a software package were all refused.

Two different bars, deliberately:

| Route | Bar | Why |
|---|---|---|
| **Armed** (she picked it from a result) | contains audio at all | her explicit choice; a booklet-heavy classical release shouldn't be second-guessed |
| **Auto** (nobody picked it) | full `music_confidence` | nobody is watching, so it must look convincingly like an album |

A skipped download is recorded as `skipped` **with its reason**, never dropped
silently — "we looked at this and left it alone because there's no audio in it"
is information; a download that just sits there is not.

Spotify linking is gated separately at `acquire.auto_link_min_percent` (80% by
default, much higher than the 45% used for the interactive picker). Unattended is
exactly where a mediocre guess does damage: nobody is looking at the candidate
list, so a "probably that one" would be accepted in silence. Below the bar the
album is still filed, just unlinked.

### Where the record lives, and why the Downloading list joins two sources

Filing **moves** files and — with `acquire.remove_after_sort` on, which is the
default — **removes the torrent**. So qBittorrent cannot answer "where did that
album go": the row is gone entirely. The job store
(`INIDATA\acquire_pipeline.json`) is the durable record, and `_AcqActiveNodes`
renders a join of the two:

- live torrents, annotated `filed → <path>` or `kept — <reason>`
- below a `filed automatically` divider, albums whose torrent no longer exists

Without that second section, filing something would make it vanish from the only
list Jamie looks at — the exact opposite of "it keeps track of where it is now".
The destination is shown **relative to the library root**, because the root
repeats on every row and pushes the part that differs off the column.

**Where it deliberately stops: QUEUED. Not staged, not burned.**
…for the AUTO path and for row action `4`. Staging is skipped there because how
the queue splits across discs depends on the *whole* queue, so auto-staging
could quietly compose a disc mixing this album with whatever else was queued.
A discography is filed but never auto-queued: "which of these 30 records goes on
the disc" is not a question this should answer by itself.

**Row action `5` does go all the way** — download, file, stage, write the disc.
Jamie asked for it explicitly after being told the tradeoff ("I won't use it
that often, so it will be fine"), so it exists, but it is the one action in the
system that shows a confirmation dialog first, because it spends a physical
write-once thing.

`_finish_by_burning` refuses in three cases, and **every refusal leaves the album
filed and queued** so the fallback is always "burn it from the queue yourself",
never a lost album and never a spoiled disc:
- the plan needs more than one disc (would need a swap it cannot ask for)
- `burn.py media` does not report `VERDICT READY` (no blank disc, wrong media)
- staging fails (nothing is written)

`run` reads `acquire.music_library` and `acquire.qbt_url` from the settings store
directly (`setting()`), because it is started by qBittorrent rather than by the
Miller and nothing hands it the configured paths. Hardcoding a default here
would silently ignore the setting the moment the library moves — which is the
one thing that setting exists to make easy.

## Library sorting and Spotify linking

A finished download becomes `<library>/<Artist>/<Album>/` and, when the album is
already in the Spotify-derived catalog, that catalog entry gains a pointer to
the files.

**Stop seeding, then move** (Jamie's choice, 2026-07-31 — an earlier build used
hardlinks and was changed). Order matters: the torrent is stopped **before** any
file moves, so qBittorrent never watches files vanish from under a live torrent.
Then the files move, and optionally the torrent is dropped from the client with
`deleteFiles=false` — essential, because the files now live in the library and
`true` would delete them there.

Note qBittorrent 5.x **renamed the endpoint**: `/api/v2/torrents/pause` is gone
(404) and `/api/v2/torrents/stop` replaced it. Only the new name is used.

Two toggles, both default on: `acquire.remove_after_sort` (drop the stopped
torrent) and `acquire.prune_after_sort` (delete the emptied download folder).
Pruning only fires when no audio remains under the folder, so a partial sort
never deletes files it did not move.

*Historical note:* the one album filed under the old hardlink build (The Glow,
Pt. 2) was migrated on 2026-07-31 — torrent stopped and removed, and the
download-side hardlink deleted so the library holds the only name. Because
hardlinks share bytes, deleting that name freed nothing and lost nothing; the
library copy was verified intact afterwards.

**`local_path` is RELATIVE** (`Microphones/The Glow, Pt. 2`), resolved against
`acquire.music_library`. Jamie expects the media directory to move; relative
means that is one settings edit rather than a rewrite of 6,831 album entries.

**Writes go through clog.** `library.py` reads `music.json` directly for
matching but never writes it — `clog media-set-local-path` was added for that,
because clog owns all catalog JSON logic. Pass an empty `--path` to unlink.

**Matching proposes, never decides.** Torrent tags are unreliable and album
titles collide, so `match` prints scored candidates and the Miller lists them
best-first; picking is one keypress and rejecting costs nothing. Scoring is
70% title / 30% artist deliberately: on classical releases the artist tag is
usually a *performer* while the catalog lists the *composer*, so a title-only
agreement still has to surface. Editions (`(Remastered 2024)`, `Deluxe`) are
stripped before comparing. Verified: "Microphones / The Glow, Pt. 2" → 0.95
against the right entry; "Trevor Pinnock / Brandenburg Concertos" → surfaced the
Bach albums despite no artist overlap at all.

### Discographies

One download is often a whole artist's catalogue. `library.py albums --path`
finds every album in it and `sort --all` files each as its own `Artist/Album`
folder rather than burying thirty records under the torrent's name.

The test for "is this an album" is **directly contains audio files**, which
handles all three shapes without special-casing: a single album folder returns
itself, a discography returns one entry per album, and a loose file returns its
parent. Multi-disc sets are the exception that needs a rule — `CD1`/`CD2`/
`Disc 2`/`Vol. 3` folders fold into their parent, so a 2-CD album is one album
and not two.

**Tag collisions are expected.** Sloppy discography rips share one album tag
across the whole set, which would merge every record into a single folder and
strand the later albums' files. On a repeat destination the release's own folder
name wins instead, reported as a `RETAG` row. Verified on a synthetic 3-album
discography with deliberately identical tags: 3 folders, all files moved, source
pruned.

**Seeing the grouping.** Drilling into a recent download shows the actions, a
divider, and then **what is actually inside it** — one row per detected album,
marked `single` or `3 discs`, with a `◈` on multi-disc sets. That breakdown is
the point: a 3-CD set reads as one album made of three discs, not as three
albums, and it is visible before anything is filed rather than after.

**Archives just work.** Windows ships **bsdtar** as `tar.exe` (libarchive
3.8.x), which reads zip, 7z, RAR and RAR5 — so there is no third-party install
and no format gap. `extract_archives()` runs **automatically** before album
detection and before filing, unpacking each archive next to itself as
`<name>_unpacked`; already-unpacked archives are skipped, so re-running is
cheap. A failed extraction removes its half-made folder so a retry starts clean.
Verified end to end: a discography that existed *only* as a `.zip` unpacked
itself, reported 2 albums (one of them a 2-disc set), and filed correctly.

### Ranking search results

`acquire.sort_mode` offers smart / seeders / size / format, and
`acquire.filter_format` offers any / FLAC / lossless / MP3. Both are pick-lists
in a **View** row at the root, not settings-only, because changing the order is
something you do *while* looking at results; picking re-runs the last search
immediately.

`smart` is deliberately **tiered, not a weighted score**:

| tier | meaning |
|---|---|
| 0 | lossless AND ≥ `acquire.healthy_seeders` (default 10) |
| 1 | lossless, poorly seeded |
| 2 | lossy but well-seeded |
| 3 | everything else |

Within a tier: most seeders, then largest. A weighted sum would silently trade
40 seeders against a format change and become impossible to reason about when it
put the wrong thing first; a tier does exactly what Jamie described — "anything
FLAC with more than 10 seeders goes to the top" — and stays explainable.
"Lossless" means FLAC/ALAC/WAV/APE/DSD, since those are the same audio in
different wrappers.

`acquire.filter_format` is a **preference, not a filter** — "FLAC first, not
FLAC only" (Jamie, 2026-07-31; an earlier build filtered strictly and was
changed). Matching releases sort to the top, then a real horizontal divider,
then everything else, still ranked and still reachable. Releases whose format
could not be read from the title fall into the lower group rather than being
discarded.

The divider carries **no index**, so result numbering stays 1:1 with the cache
that `add` and `link` read — the row after the divider continues the sequence.
It is only emitted when both groups survive the result limit.

Artist and album come from a **majority vote** over the first 40 files' tags,
not from the first file — compilations routinely have one track whose tags
disagree. If tags are unreadable it parses the folder name and says so
(`SOURCE: folder name`), because that guess is much likelier to be wrong.

## Burning

Audio CDs are Red Book: **44100 Hz, 16-bit signed LE, stereo**, always. IMAPI2's
`AddAudioTrack` does not resample, so a 24/96 FLAC handed over verbatim plays at
the wrong pitch. Every track goes through ffmpeg with those three parameters
pinned — verified: staged output probes as `pcm_s16le / 44100 / 2`.

Capacity is **time, not bytes** (79:57 on an 80-minute disc). A 2 GB FLAC album
and a 200 MB MP3 rip of it occupy identical space once decoded. `plan` splits
greedily **in queue order** — a bin-packer that reordered tracks to waste fewer
seconds would scramble album order, which is worse than a half-empty disc.

`stage` emits a standard WAV+CUE set, so ImgBurn/foobar2000/CDBurnerXP can burn
it even if IMAPI2 is unavailable. CDBurnerXP itself was rejected: unmaintained
since 2019, awkward CLI, and IMAPI2 ships with Windows.

### Not wasting discs

Jamie has **no rewritable discs**, so every avoidable failure costs a CD-R.
`burn.py media` is a purely read-only inspection — media type, blank state,
whether the selection fits, and the drive's real write speeds — and the same
report runs automatically as a preflight before any write. If it fails, nothing
is written and the burn confirm shows the reasons instead of asking blindly.

**The preflight never calls `PrepareMedia()`.** `NumberOfExistingTracks` and
`FreeSectorsOnMedia` both raise *"only valid when media has been prepared"*, and
preparing a disc just to read a property risks spoiling the very disc the check
exists to protect. `MediaPhysicallyBlank` and `CurrentPhysicalMediaType` work
without it, which is enough: blank + CD-R is the decision.

**Media type is the trap.** The drive is a *DVD writer*, so a DVD-R in it looks
perfectly writable — and would never play in a CD player. Only `CD-R` and
`CD-RW` (IMAPI types 2 and 3) pass.

**Write speeds are sectors/second, not multipliers.** The drive advertises
`[1800, 1199, 750]`, i.e. 24x/16x/10x at 75 sectors per second. Passing the
`acquire.burn_speed` multiplier straight to `SetWriteSpeed` would have requested
~0.1x; it is converted and snapped to a speed the drive actually advertises.

**Splitting always warns.** `plan` emits a `SPLIT` row whenever the selection
exceeds one disc, and the Miller shows it at the top of Discs. The usual cause
is a deluxe edition where only the first N tracks were wanted, so the fix is to
exclude tracks, not to accept a second disc. Queue entries carry an `on` flag;
Enter on a track toggles it (✓/✗) and excluded tracks stay listed so they can be
flipped back. Verified: 20-track album → exclude 13-20 → 12 tracks, 40:31, one
disc, no split warning.

Capacity is stored as **whole seconds** (`acquire.cd_max_seconds` = 4797). Storing
79.95 minutes rendered in the Settings UI as `79.950000000000003`, because
float64 cannot represent it exactly.

## The results level

Opens **directly on the results** when the viewer was launched by a search
(`initial_path: ["results"]`) — asking for a search and then landing on the hub
costs a keypress every time. Row 1 is `⟳ Search again…`, because refining a
query is the most common thing to do while looking at results.

Every row shows **Release · Fmt · Size · S · L**, so the list can be judged
without drilling. Each result is a **branch, not a leaf**: the Miller previews a
branch's children in the right-hand pane, so arrowing onto a release shows its
size, health (with seed:leech ratio), format, indexer, category, publish date
and full title with no keypress at all. A leaf has no children, which is why the
right pane was blank in the first build. Downloading stays one keypress away as
row action `N.1`; `N.2` adds paused, `N.3` copies the link.

Torznab category IDs are mapped to names (`3040` → `Audio/Lossless`), falling
back to the parent decade before giving up, because indexers invent sub-ids
freely inside the standard ranges.

## Traps

- **The viewer's include closure is not checked.** `ahk_include_closure.py`
  skips `Scripts/`, so a missing `#Include` in `AcquireViewer.ahk` passes both
  `validate` and the closure check and then throws at runtime. The only real
  proof is running the node builders in a viewer-shaped process — see the
  harness pattern used during the build.
- **AHK v2 closures capture the loop VARIABLE, not its value.** A fat-arrow
  written inline in a `for` loop makes every row act on the last item. All row
  actions go through factory functions (`_AcqAddFn`, `_AcqStageFn`, …) precisely
  to get a fresh scope per row.
- **Unique temp file per shell-out.** AHK timers interrupt a waiting thread, so
  a debounced preview firing mid-`RunWait` would delete the file the outer call
  is waiting on. Same reasoning as `_BMShell` in [[BookManagerMenu]].
- **`Log` is a built-in AHK v2 function.** Assigning to a variable named `LOG`
  is a hard error ("This Func cannot be used as an output variable").
- **PowerShell 5.1 `Set-Content -Encoding utf8` writes a BOM**, which breaks
  Python's `json.load`. Use `UTF8Encoding($false)` for anything Python reads.
- **Every Miller level must declare the SAME number of columns.** AHK v2
  ListView headers are immutable once created, so the control is built with the
  column count of the FIRST level and deeper levels can never add more. A root
  level with 2 columns and a results level with 5 renders the last three
  **blank** — the node data is correct and the header strip even prints all five
  names, but the ListView physically has nowhere to put them. Pad the shallower
  levels with empty headers.
- **Declared column widths are only a starting point.** `_Mcp_FitColumns`
  re-sizes every column to its widest cell on each render and hands the leftover
  to column 1, so tuning the numbers in `levels[]` mostly does nothing. Fix
  layout problems by fixing the column COUNT, not the widths.
- **Don't declare a fractional setting as a float.** `0.45` and `79.95` render in
  the Settings UI as `0.45000000000000001` and `79.950000000000003` — float64
  cannot represent them exactly. Declare whole units instead
  (`cd_max_seconds` = 4797, `match_min_percent` = 45) and convert in the AHK
  reader. This bit twice.
- **Never leave a backup file in the Startup folder.** A `StartupOrder.bat`
  backup saved beside it as `.bak-before-claude` made Windows pop a "select an
  app to open this .bak-before-claude file" dialog on every boot — Startup runs
  everything in the folder regardless of extension. Backups live in
  `INIDATA\_backups\`.
- **Match processes by command line, not by name, when killing.** A
  `Stop-Process` matching `AutoHotkey*` killed every always-on script (footpedals,
  Q0 Max numpad, CapsLock macropad, Parent, MicDuck) along with the intended
  target.

## Debugging order when a search returns nothing

1. `jackett status` — service, port 9117, **FlareSolverr**, auto-update state.
2. Acquire Miller → Health (same three probes, in-UI).
3. `py Scripts\acquire\acquire.py search "test"` — raw, unmediated by AHK.
4. `C:\ProgramData\Jackett\log.txt`.

## Video — TV and film (2026-08-14)

Built after Jamie downloaded a batch of public-domain shows and filed them by
hand. Same destination as music — a library Plex reads — but arrived at by a
**different strategy**, and the difference is the whole point of this section.

### Why video does not wait, and music must

Music's destination is unknowable until the files exist: it comes from the ID3
tags *inside* them. So the music path is necessarily

```
add → download → read tags → MOVE into place
```

A video release names its own show and season in the torrent title, which
qBittorrent knows the moment a magnet resolves. So video can do

```
add → compute destination → download DIRECTLY into place
```

`video.py place` implements the second: the torrent's save path is set to the
show folder and every file is renamed into the season layout through
qBittorrent's API, before a byte arrives. Three things fall out of that, and
only the third is the one that was expected:

- **Seeding never stops.** Nothing is moved out from under the torrent, so a
  100 GB pack keeps seeding from the library indefinitely. The old path had to
  stop the torrent to move it — there is already a `missingFiles` torrent in
  the client pointing at `E:\Media\TV\Sense8.S01-S02...` from doing this by
  hand.
- **Episodes appear as they finish.** Episode 1 of a ten-season pack is
  watchable the evening the download starts, instead of when episode 200 lands.
- **The move was never the expensive part.** `E:\Downloads` and `E:\Media` are
  one volume, so `shutil.move` is a rename — a 100 GB season and a 3 MB single
  cost the same. The wait was the cost, not the copy.

Renaming inside a torrent is client-side bookkeeping: pieces map to byte
offsets, not names. The swarm neither knows nor cares.

### The layout

```
E:\Media\TV\Show Name (2010)\Season 01\Show Name (2010) - s01e02.mkv
E:\Media\Movies\Film (1999)\Film (1999).mkv
```

Specials go to `Season 00`; video with no episode number goes to the show's
`Extras\`, which Plex reads as bonus material rather than failing to match it
to an episode. Samples (by name **or** under 50 MB) and tracker adverts are
dropped. Subtitles follow their video and take its new basename so Plex pairs
them. Films keep only the **largest** video as the feature — film torrents ship
trailers, and Plex would list one as a second copy.

`Season 1` and `Season 01` are both valid to Plex, so no normalisation pass was
needed for the existing library (`Peaky Blinders` has unpadded folders). New
folders are padded; an existing unpadded one is reused rather than duplicated.

### Matching a show to a folder that already exists

`existing_show_dir` matches on the normalised name, so a release that omits the
year finds `Steven Universe (2013)` instead of creating `Steven Universe`
beside it — which would show the series twice in Plex with half the episodes in
each. Trailing pack wording (`Public Domain Collection`, `Complete Series`,
`Box Set`) is stripped for the same reason: it describes the release, not the
show, and leaving it on splits one series across two folders.

### The catch-up ledger

Completion is one callback from qBittorrent. That is the right mechanism, but
it fires **exactly once** — if the client is closed, killed or updated at the
moment a torrent finishes, that notification is simply lost and the download
sits complete on disk forever.

So the **ledger** (`INIDATA\acquire_media_log.jsonl`, append-only), not the
callback, is the record of what has been dealt with. `pipeline.py sweep` files
anything complete with no terminal ledger row, and it is hung off every
completion — so finishing any download also catches up everything missed
earlier. A row is written for "left alone" as well as "filed", or every ebook
in the downloads folder would be re-classified on every sweep forever. A
**failed** row is deliberately not terminal: that is exactly the case that
deserves another try.

Append-only rather than a rewritable dict because this is the record of what
*happened*; a crash mid-write truncates the last line, which the reader skips,
instead of corrupting everything before it.

The sweep also scans the downloads folder for **orphans** — folders no live
torrent claims. `remove_after_sort` takes a torrent out of the client the
moment it is filed, so anything downloaded and then removed *without* being
filed exists only as a folder, and no callback will ever mention it again.
That is the state of everything downloaded before this was built. Orphans get
a synthetic `path:<name>` key; `_real_hash` strips it so no qBittorrent call is
ever aimed at a hash that does not exist.

### Traps this hit

- **`--save-path` is always passed.** The Miller passes `acquire.save_path` on
  every arm, so testing `not args.save_path` to mean "no override" disabled
  add-time placement *everywhere it mattered* while still working perfectly
  from the command line. `_explicit_path` compares against the configured
  default instead.
- **A folder name outvoting the disk.** A three-episode public-domain season
  classified as a **film** because the ≥4-file heuristic missed it — it would
  have been filed into Movies. A `Season N` subfolder is now decisive evidence,
  whatever the release is called.
- **`sample.mkv` is a video file.** So `prune_empty` counted it as "something
  of value" and kept every scene release's download folder alive forever after
  its episodes were filed. The test that refuses to *file* a sample has to be
  the one that refuses to be fooled here.
- **`finished` matches on a qBittorrent hash**, so it cannot see an orphan
  folder at all — an album sitting right there reported "no audio found".
  `_inspect_album` reads the folder directly when there is no torrent.
- **The settings drift checker only knew `Setting("sys.key")`.** Every setting
  read only by Python (which is all of the Plex ones — filing runs from
  qBittorrent's hook, where there is no AHK) was reported as "nothing reads
  it", which is precisely backwards and trains you to ignore the warnings. It
  now also recognises `setting("sys", "key")` and the bare-key helpers.
- **A screenshot caught mid-render is not a bug.** The first capture of the
  Miller showed 3 of 11 rows and looked exactly like an exception killing the
  root build. It was the window being photographed before the list finished
  populating. Re-shoot before diagnosing.

### Plex

`plexlib.py`. The token lives in the **registry** on Windows
(`HKCU\Software\Plex, Inc.\Plex Media Server\PlexOnlineToken`), not in
`Preferences.xml` as on every other platform; both are read, registry first.
Sections map onto the folders directly (`1 movie E:\Media\Movies`,
`2 show E:\Media\TV`), and a scan is **path-scoped** — a full section refresh
takes minutes and re-reads everything, while a scoped one is near-instant.
Matching is done on the path, not the section title, so renaming a library in
Plex changes nothing.

`plexlib check` answers the separate question of whether a file will actually
*play*. The expensive case is image-based subtitles (PGS/VOBSUB): Plex cannot
overlay them, so enabling them forces a **full video transcode**. DTS/TrueHD
forces a cheap audio transcode. Her Sense8 rips are HEVC with text subs — one
flag, no burn-in risk.

### Files

| File | Owns |
|---|---|
| `mediacore.py` | Everything shared by *any* filer: paths, moving, unpacking, settings, classification, the ledger, the small qBittorrent client |
| `library.py` | Music only. Re-exports the mediacore names it always exported |
| `video.py` | TV + film: name parsing, the Plex layout, `place` and `file` |
| `plexlib.py` | Token, sections, scoped scan, direct-play verdict |
| `pipeline.py` | Arming, completion dispatch, the sweep, the log |

## Not built yet

- **The actual disc write.** Everything up to it is verified against the real
  drive (`HL-DT-ST DVDRAM GT50N`, F:): media reads as CD-R, blank, READY, with
  10x/16x/24x available. Only `AddAudioTrack` + `ReleaseMedia` remain unproven,
  and proving them costs a disc. First real burn should use the shortest
  selection that is still worth keeping.
- **A real video download end to end.** Every piece is tested — the parser
  against 10 real release names, the layout against synthetic seasons and
  films, the rename plan against a three-season pack, the classifier against
  real folders — but no TV torrent has yet been armed and watched all the way
  into Plex. That is the one thing left to prove.

Related: [[MEDIA_SYSTEM]] · [[COMPLETION_LOG]] · [[SETTINGS_SYSTEM]]
