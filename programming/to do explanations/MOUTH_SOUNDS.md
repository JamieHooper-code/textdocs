---
tags: [design, accessibility, voice, audio, reading, parrot]
related: ["[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[READING_ROOM_BOOKS]]", "[[KEY_ACTION_BINDING_FEATURE_QUEUE]]"]
status: model trained; gesture engine + modes + Dragon<->Mouth switch built — 2026-10-08
updated: 2026-10-08
---

# Mouth sounds (Parrot.py)

Non-speech mouth sounds (cheek click, tongue click, lip pop) as input. The first
use: a cheek **click-click** turns the page in the reading room while she reads
with quiet music on.

## Why Parrot.py

- **Dragon can't do it.** It is a word recognizer, so a click is noise to it.
  Adding "tch tch" as a vocabulary word never worked.
- **Talon** recognizes only *pop* and *hiss* in the public build. Custom noises
  need the Patreon beta, and that beta runs Parrot models underneath anyway.
- **Parrot.py** (github.com/chaosparrot/parrot.py) trains a small model on HER
  sounds and HER mic. It runs on Windows with Python 3.12, reacts in about
  20 ms, and is still maintained.

## Layout

| Piece | Where |
|---|---|
| Parrot clone + its own `.venv` | `C:\Users\jamie\Tools\parrot.py` (not Desktop: OneDrive would sync ~1 GB of torch) |
| Her data: recordings, models, code | `C:\Users\jamie\Tools\parrot_data` (`PARROT_DATA_DIR`), kept outside the clone so re-cloning never loses trained sounds |
| Sound -> action registry | `AutoHotkey\INIDATA\mouth_sounds.json` |
| The Parrot mode (source of truth) | `AutoHotkey\Scripts\mouth_sounds\mode_mouth_sounds.py`, copied into `parrot_data\code` on every start |
| Gesture engine (pure, tested) | `AutoHotkey\Scripts\mouth_sounds\gesture_engine.py`, tests `Scripts\codebase_tools\tests\test_mouth_gesture_engine.py` |
| Live state | `INIDATA\mouth_sounds_state.json` (`{"mode": ...}`), `INIDATA\mouth_sounds_testing.flag` |
| CLI | `AutoHotkey\Scripts\mouth_sounds\mouth_sounds.py` with `on [mode]` / `off` / `mode [m]` / `status` / `testing` / `studio` / `test` |
| AHK | `Helpers\MouthSoundFunctions.ahk`: `ToggleDragonMouth` / `DragonToMouth` / `MouthToDragon` / `MouthSoundsMode` / `MouthSoundsTestingToggle` / `MouthSoundsOn/Off/Toggle/Status/Studio` |
| Voice | "set mouth" = `ToggleDragonMouth`, "testing mouth" = testing tooltips, "open mouth" = studio |

## How a sound becomes an action (v2, 2026-10-08)

1. Parrot classifies each 15 ms frame into class confidences (0-100).
2. `GestureEngine` turns frames into **hits**: a burst of one *token* (a sound,
   or a **pool** like `any click` = the SUM of the three click classes), judged
   by its PEAK frame.
3. Hits become **gestures**: ordered steps, each a token or a list of
   acceptable tokens. `cheek click x2` = a cheek click, then a cheek click OR
   any click.
4. The **mode** says which gestures are live and how strict each step is:
   per-step `min` confidence and `window_ms` between steps. The first step must
   stand alone; later steps only confirm, so they can be far more lenient.
5. Ties go to the most specific gesture (more steps, then fewer pooled tokens).
   A single that is a prefix of a live double is deferred until the window
   lapses: normal double-click behaviour, and the engine's only added latency.
6. The fired gesture is looked up in `bindings` for the **active mode** against
   the live foreground context chain. No match = nothing happens. `key` is
   pressed by Parrot (instant). `fn` runs `MAINFUN.bat` (**caution:**
   `#SingleInstance Force` kills any running MAINFUN call; prefer `key`).

Registry, mode and testing flag are re-read every 0.5 s: threshold edits apply
live, no restart. A **stale** context file (older than 150 s) **fails closed**.

Log: `MouthSound/hit` (every raw-sound burst + peak, "(too weak)" when it armed
nothing), `/fired`, `/unbound`, `/blocked`, `/config`, all with **millisecond**
timestamps. Fired lines carry the step peaks and the gap between clicks
(`[72/51 gap 380ms]`), which is the data for tuning.

## Modes

| Mode | Dragon | Strictness | Status |
|---|---|---|---|
| `reader` | off | cheek x2 = `[60, 45]` in 750 ms; lip pop x2 (back to Dragon) = `[85, 80]` | live |
| `always` | on | default 92 / 500 ms | placeholder |
| `wispr` | off (Wispr has the mic) | default 95 / 450 ms | placeholder |

**Dragon <-> Mouth** ("set mouth", `ToggleDragonMouth`) is the twin of the
Wispr handoff. To Mouth: `SilenceDragonMic()` (fully OFF, not sleep, since a
sleeping Dragon keeps a live wake grammar), then the listener starts in
`reader`; if the listener fails, Dragon is woken back up. Back to Dragon:
**lip pop twice**, since Dragon is off and can't hear "set mouth". It wakes
Dragon first, then stops the listener. `MouthSoundsOff` kills only the
listener's pid (no `/T`), because the back-to-Dragon MAINFUN call is the
listener's own child and must survive it.

## Diagnosis: the reader misses (2026-10-07/08 data)

Reader log, grouped into attempts: 29 clean page turns; **10** where both
clicks registered but more than 450 ms apart; **17** lone single clicks;
several where one click of the pair scored 63-89% against the old 90%
threshold. 37 of 60 cheek near-misses were in 70-89%. Some cheek clicks partly
scored as tongue clicks.

Replaying the training recordings through the live frame path:
- Real recorded cheek clicks are caught about 94% of the time even at 90%.
  (A third of the recorder's "cheek click" segments peak at 52-96 intensity,
  below the room's background median of 366: mislabelled silence, not missed
  clicks. Parrot's silence gate, `SILENCE_INTENSITY_THRESHOLD = 400`, is NOT
  the problem.)
- 171 s of background: cheek at 55% produced 4 stray single hits and **0 stray
  doubles**. The engine at reader settings: **0 page turns**.

So live misses are threshold + window + click-type confusion, not the model's
ability to hear. v2 fixes all three: `[60, 45]`, 750 ms, and an `any click`
second step.

**Next lever if misses remain:** retrain with LIVE-condition data. Record
cheek clicks at reading posture and volume with the music playing, clicked in
doubles. Record a second background session of reading noises (page turns,
swallowing, lip smacks, breathing) so leniency stays safe.

## Traps already handled

- Parrot's `BaseMode.start/exit` press **F4** (its eye-tracker toggle) into the
  foreground window. The mode overrides both.
- `SAVE_REPLAY_DURING_PLAY` defaults to True and grows a CSV forever. The CLI
  writes `False` into the generated config.
- Sounds declared in the registry but not yet trained are skipped (and logged
  once), so sounds can be added one at a time.

## Training workflow

`MouthSoundsStudio` opens Parrot's console menu:

1. **[R]ecord** each sound into a folder named exactly as in `sounds`:
   `cheek click`, `tongue click`, `lip pop` (spaces, as recorded). Also record **silence/background**:
   the room with the music playing, page sounds, breathing. About 40-80 s
   each; aim for similar amounts per sound.
2. **[L]earn** a model named `mouth_sounds` (the registry's `model`).
3. **[A]nalyse** to see the confusion matrix.
4. "testing mouth" pops a tooltip per hit and per gesture. Tune `min` /
   `window_ms` per mode in the registry; it applies live.

## Planned

- **Miller editor** for modes, gestures, thresholds and bindings, all in AHK
  with no Parrot GUI.
- **Bindings into the macro system:** mouth as a device in `bindings.json`
  (slot = gesture, layer = `mouth:mode`, scope = context), so the macro wizard,
  key-actions and scopes work for free. Detection tuning stays in
  `mouth_sounds.json`. Needs an always-on AHK receiver (WM_COPYDATA from
  Parrot) so a gesture resolves in about 15 ms instead of a MAINFUN spawn.
- `always` and `wispr` modes (Wispr: double click = new paragraph).
- Previous page.
