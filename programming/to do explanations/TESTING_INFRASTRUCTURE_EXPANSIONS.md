---
tags: [todo, autohotkey, testing, claude-tooling]
---

# Testing Infrastructure — Future Expansions

Four candidate improvements to the AHK testing infrastructure, surfaced from the April 2026 DF snap investigation. Listed in rough effort order. None are urgent; pick up when we hit a debugging session that one of them would help with.

---

## 1. `TestLogMarkers(fnName, *expectedMarkers)` — generic log-based test runner

**Effort:** small (~40 lines)
**Unblocks:** testing any non-window-changing function that emits `LogEvent` calls.

Sketched in `~/.claude/skills/ahk-functions/testing.md`. Lives in `Helpers/TestHarness.ahk`.

Pattern:
1. Snapshot `ahk_event.log` size before firing.
2. Fire the target function via the dispatcher (or directly).
3. Sleep briefly (~300ms) to let log writes flush.
4. Read the new log content (everything after the snapshot offset).
5. Assert each expected marker substring appears.
6. Return `OK` or `MISSING: <marker>`.

Usage:
```
MAINFUN.bat TestLogMarkers RestartStreamDeck "STREAMDECK/quit" "STREAMDECK/relaunch"
```

**When this matters:** debugging functions like `RestartStreamDeck`, `CopyPasteManager`, `SaveCurrentLink` etc. — anything where the log tells you what stage was reached. The snap harness can't help with these because the side effect isn't a window position change.

**Caveats:**
- Only useful for **non-destructive** functions (don't auto-test things that delete files, send messages, modify shared state — get Jamie's approval first).
- Requires the function actually emits LogEvent calls at the points worth verifying. If it doesn't, add LogEvent calls first.
- Asserts substrings, not exact-match. Markers should be unique enough not to collide with unrelated log lines.

---

## 2. Layer 1: `verify.ps1 ahk-run <fnName> [args...]` — fire any AHK function with log diff

**Effort:** small (~50 lines in `~/.claude/helpers/verify.ps1`).
**Unblocks:** quick one-shot AHK function firing without the snap-harness overhead.

Originally proposed in [`AHK_FUNCTION_TESTING.md`](./AHK_FUNCTION_TESTING.md) (Layer 1 of the old plan). Layer 2 doesn't need building, but Layer 1 is still useful as a small standalone tool.

Behavior:
1. Resolve `MAINFUN.bat` path (already on PATH).
2. Snapshot tail of `ahk_event.log` before firing.
3. Invoke `MAINFUN.bat <fnName> [args...]` with `Start-Process -WindowStyle Hidden -Wait`.
4. Diff log to find new lines.
5. Report `OK|ahk-run|<fnName>|dispatched` or `ERROR|ahk-run|<fnName>|no-dispatch`.

**When this matters:** debugging dispatch issues, testing whether a new function is reachable, sanity-checking after a `MAINFUNCTIONS.ahk` `#Include` change. Faster than spinning up the snap harness for "does this function exist and dispatch."

**Why this didn't get built today:** the snap harness was the immediate need. This is the natural complement for non-snap functions but can wait until we hit a case where it'd save time.

---

## 3. UIA-based inner-view inspection — close the harness's biggest blind spot

**Effort:** medium-large (research first, then ~100-200 lines).
**Unblocks:** detecting paint glitches inside apps where the outer window rect is correct but inner views are mis-rendered (the Notepad++/Scintilla bug from the DF snap investigation).

The current harness reads `WinGetPos` for outer rect. It can't see whether Notepad++'s Scintilla edit area was correctly resized. UIA (UI Automation) can introspect the control tree and read individual control bounds — `Helpers/UIA-v2-main/Lib/UIA.ahk` is already loaded by `MAINFUNCTIONS.ahk`.

Plan:
1. **Research first.** Use the existing `Diagnostics/UIADump.ahk` (or similar) to dump Notepad++'s UIA tree when it's correctly maxed and when it's broken-render-state. Find the control name + class for the Scintilla edit area. Confirm UIA exposes its bounding rect.
2. If yes: add `_THCaptureInnerView(hwnd, controlIdentifier)` to the harness that returns the inner control's rect alongside the outer window's rect. Log both in JSONL.
3. Update the analyzer to flag mismatches: outer max + inner small = the Scintilla bug.

**When this matters:** specifically when a window-management bug is invisible to `WinGetPos`. We hit this exactly once so far (DF snap). Probably not worth building speculatively, but the moment a similar bug shows up again, this is the path forward.

**References:** `Diagnostics/UIADump.ahk`, the existing UIA-v2 lib at `Helpers/UIA-v2-main/`. The `df_snap_investigation.md` retrospective documents the symptom.

---

## 4. `DisplayFusionCommand.exe` CLI exploration

**Effort:** small (10-min spelunk, then write findings into the AHK skill).
**Unblocks:** potentially bypassing the cursor-position dependency in DF's hotkey-based maximize.

Currently we route every snap through `Send("#^!N")` keystrokes that trigger DF's bound hotkeys. DF also ships `DisplayFusionCommand.exe` (path: `Cfg.DisplayFusion`), which has a CLI surface we've never explored. If it has commands like `MoveWindow --monitor N --maximize` or similar, we could call those directly and skip:
- The cursor-position dependency of `WindowCurrentMonitorMaximize`.
- The hotkey collision risk (DF hotkeys overlap with Stream Deck assignments).
- The "modifier order matters" registry editing pain.

Plan:
1. Run `DisplayFusionCommand.exe -help` (or `/?`) and capture the output.
2. Read the docs page if there is one (BinaryFortress site).
3. List the verbs that map to our existing AHK snap functions.
4. If the CLI is richer than the hotkey path: write a `Helpers/DisplayFusionCli.ahk` shim and migrate one snap function to use it as a proof of concept.
5. If the CLI is poor or hotkey-equivalent: document that finding so we don't re-investigate later.

**When this matters:** next time we need a DF-related fix and the cursor-position issue rears up again. Or if we ever hit a hotkey collision we can't easily fix.

**Why this didn't get done today:** out of scope for the snap fix; the AHK side now works.

---

## Order-of-operations recommendation

1. **TestLogMarkers** — small, useful immediately, generalizes to many future debugging sessions.
2. **`verify.ps1 ahk-run`** — small, complementary to TestLogMarkers.
3. **DisplayFusionCommand spelunk** — 10 minutes of exploration is cheap and the findings either save future time or rule out a path.
4. **UIA inner-view inspection** — bigger lift, only do if a second case justifies it.
