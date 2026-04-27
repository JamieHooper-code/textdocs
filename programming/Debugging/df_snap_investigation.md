---
tags: [debugging, autohotkey, displayfusion, window-management]
---

# DisplayFusion Snap / Window Placement — Investigation Retrospective

Date: 2026-04-26
Branch: main (caster/rules)
Related code: [[WindowSnappingFunctions.ahk]], [[OpeningAndClosingFunctions.ahk]], [[TestHarness.ahk]]

## Original symptom

When voice/Stream Deck commands like `open friends main` fired, the window's path was visibly ugly:
1. App launches on whatever monitor it remembered.
2. Code does `_DFMovePrimary` (move to primary) → window briefly flashes to monitor 1.
3. Code does `_DFMaximize` → window maxes on monitor 1.

For windows that opened on monitor 2 and then got moved to monitor 1, the bounce was unavoidable but ugly. For other paths (e.g. `*Twin` to send to monitor 2) the bounce was MORE pronounced because the chain was `MovePrimary → MoveNext → Snap`, three steps instead of one.

Jamie wanted: smarter, no bouncing.

## What we tried, in order

### 1. Conditional moves based on current monitor *(KEPT)*
Added `_GetActiveWindowMonitorIndex()` and `_ActiveWindowOnPrimary()` helpers that read the active window's center-of-rect against `MonitorGet` work areas. Rewrote `SnapSecondaryTop/Bottom/Main` to skip the cross-monitor move when the window is already on secondary. **This works and is kept.**

### 2. `_DFMaximize` → `WinMaximize("A")` *(REVERTED)*
We discovered DF's `WindowCurrentMonitorMaximize` (Win+Ctrl+Alt+7) uses **cursor position**, not window position, to decide "current monitor." If the cursor is on monitor 1 and the active window is on monitor 2, the hotkey *relocates* the window to monitor 1 and maxes there. We swapped to AHK's native `WinMaximize("A")` which is window-position-based.

That fix was correct in isolation but introduced a different bug: `WinMaximize("A")` races with the launching app's startup geometry application. Notepad++ and Everything apply their remembered restored geometry ~150-300ms after launch, even when our maximize already fired. The result: window reports `min_max=1` to Windows but the actual rect is half-size (or worse — Notepad++ sometimes reports max but renders only the title bar because Scintilla didn't relayout). This is a known upstream Notepad++ multi-monitor bug class — see GitHub issues #6284, #8660, #13346, #3457. We can't fix it from AHK.

`_DFMaximize` (Send the hotkey) doesn't have this race — DF's own implementation handles app startup timing. So we reverted to `_DFMaximize` and accepted the cursor-position dependency. In practice the cursor is usually on the user's main monitor, so this almost always works.

### 3. `RunWithStartupPos` via `CreateProcessW` + `STARTUPINFO` *(REVERTED)*
Tried launching apps with `dwX/dwY/dwXSize/dwYSize` + `STARTF_USEPOSITION` + `STARTF_USESIZE` + `STARTF_USESHOWWINDOW`. Idea: open the window already at target rect + maxed, frame zero, no race for the post-snap to fight. Notepad++ honored `SW_SHOWMAXIMIZED` (good — opened maxed) but ignored `dwX/dwY` (it overrides with its remembered position). For our two-monitor flow, that's not enough — we still needed a cross-monitor move. The post-launch race for half-snaps was unimproved. Net: more code, no benefit.

### 4. Ghost-state recovery toggle (`WinRestore` → `WinMaximize`) *(REVERTED)*
When Notepad++/Everything end up "maxed but small", a `WinMaximize` is a no-op (Windows already thinks the window is maxed). Toggling `WinRestore` then `WinMaximize` forces a clean state transition. This worked but introduced a visible flicker, and the heuristic for "is the window in a ghost state" wasn't perfect (width-only heuristic missed the Scintilla-only-paint variant).

### 5. `WM_SIZE` post-message + `SetWindowPos(SWP_FRAMECHANGED)` *(REVERTED)*
Last attempt to force Notepad++'s inner Scintilla to relayout without a visible flicker. Posted `WM_SIZE(SIZE_MAXIMIZED, current_client_dims)` and a `SetWindowPos` with `SWP_NOMOVE | SWP_NOSIZE | SWP_FRAMECHANGED`. Didn't fix the symptom. Notepad++'s rendering bug is deeper than what `WM_SIZE` triggers.

## Final state (what's actually in the codebase)

**`WindowSnappingFunctions.ahk`:**
- `_GetActiveWindowMonitorIndex()` and `_ActiveWindowOnPrimary()` helpers — center-of-rect lookup against `MonitorGet`.
- `SnapPrimaryMain` uses `_DFMovePrimary` → `Sleep(100)` → `_DFMaximize` (original behavior, just preserved).
- `SnapSecondaryTop/Bottom`: conditional `_DFMoveNext` (only if window is on primary), then `_DFSnapTop/Bottom`. No cross-monitor bounce when the window is already on secondary.
- `SnapSecondaryMain`: if on primary, `_DFMoveNextMaximize`; else `_DFMaximize`. No bounce in the in-place case.
- `SnapByAlias` unchanged.

**Test harness — kept and is real value:**
- [[TestHarness.ahk]] (`Helpers/TestHarness.ahk`)
- [[analyze_snap_traces.py]] (`Diagnostics/analyze_snap_traces.py`)
- See the testing section in `~/.claude/skills/ahk-functions/SKILL.md` for usage.

**`OpenAndPlace`, `AppHandling.ahk`, `MAINFUNCTIONS.ahk`:** back to pre-investigation state.

## Limitations / known issues

### Notepad++ multi-monitor rendering glitches (upstream, unfixable from AHK)
- Notepad++ has multiple open issues for multi-monitor scenarios with mixed scaling/aspect: [#6284](https://github.com/notepad-plus-plus/notepad-plus-plus/issues/6284), [#8660](https://github.com/notepad-plus-plus/notepad-plus-plus/issues/8660), [#13346](https://github.com/notepad-plus-plus/notepad-plus-plus/issues/13346), [#3457](https://github.com/notepad-plus-plus/notepad-plus-plus/issues/3457).
- Symptom: outer window is correctly sized + reports `min_max=1`, but the inner Scintilla view paints at a stale tiny size. User sees "only the title bar". `Win+Down` then `Win+Up` (manual restore→max) fixes it because that forces Scintilla to relayout.
- Programmatic `WinRestore + WinMaximize` from AHK fires the same Win32 sequence and DOES fix it, but has a visible flicker. We've decided that flicker is worse than the rare appearance of the bug, so we don't apply the fix automatically.

### Cursor-position dependency on `_DFMaximize`
- `_DFMaximize` (DisplayFusion's `WindowCurrentMonitorMaximize`) maxes on the monitor the *cursor* is on, not the active window's monitor.
- For `SnapPrimaryMain` and `SnapSecondaryMain`'s in-place branch, we rely on the cursor being on the target monitor.
- In practice: Jamie's cursor is almost always on monitor 1 (her main work area). For "open X main" calls, this works. For "open X twin" calls, the path goes through `_DFMoveNextMaximize` (DF's `MoveMonitorMaximize`) which is window-based, not cursor-based, so no issue.
- If we ever hit a case where the cursor is on the wrong monitor at snap time, the fix is to `MouseMove` to target monitor center first, fire `_DFMaximize`, then optionally restore the cursor. We didn't do this because the visual cursor jump is also ugly.

### Test harness can't see inner-paint glitches
- `WinGetPos` reports the outer window rect. If Notepad++'s inner Scintilla painted at a wrong size while the outer rect is correct, the harness sees "everything OK" even though the user sees a broken window.
- This is why the harness greenlit our fixes that Jamie still saw as broken in actual use. The test infrastructure is real value, but Notepad++'s paint glitch is invisible to it. Future tests for paint-correctness would need UIA inspection or pixel-level screenshot diffing — overkill for this case.

## Where to pick this up if it bites again

1. **First**: re-read this file before changing snap code. Most plausible "fixes" have been tried and have known costs.
2. **If the bug is the Notepad++ inner paint glitch specifically**: consider switching that file to a different editor (Obsidian, VS Code) for window-management. Don't try to fix Notepad++ from outside.
3. **If the bug is a real new regression**: run the harness sweep first (`MAINFUN.bat RunAllSnapTests`, then `py Diagnostics/analyze_snap_traces.py`) to see whether the snap functions land on the correct monitor/state. The harness catches monitor-misplacement reliably; it does NOT catch inner-paint issues.
4. **If you need to test the production OpenAndPlace path**: there's a `RunOpenAndPlaceTest` and `RunAllOpenAndPlaceTests` in the harness. Note that this uses Notepad++ with `-multiInst -nosession` so it's *not* the exact same code path as voice-fired "open X main" (which uses an existing instance). The mismatch is documented in the harness header.
