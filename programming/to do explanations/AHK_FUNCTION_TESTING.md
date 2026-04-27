---
tags: [autohotkey, testing, claude-tooling]
---

# AHK Function Testing — Status

**Status: partly built, partly still TODO.** Originally drafted as a Layer 1 + Layer 2 plan. Layer 1 is now real (different shape than originally proposed). Layer 2 is not built and probably doesn't need to be.

## What exists today (built April 2026)

A pure-AHK test harness for window-changing functions. Lives in:
- `~/AutoHotkey/Helpers/TestHarness.ahk` — runner + scenarios + dispatch entry points
- `~/AutoHotkey/Diagnostics/analyze_snap_traces.py` — JSONL trace analyzer
- `~/AutoHotkey/Diagnostics/test_results/` — JSONL traces written here per run

The harness:
1. Spawns a disposable Notepad++ instance (`-multiInst -nosession`) as the test target.
2. Stages it into a known scenario (primary normal, secondary maxed, etc.).
3. Fires the function under test.
4. Samples window state every 50ms for ~2s, capturing in-flight bouncing.
5. Writes a JSONL trace.
6. Cleans up.

MAINFUN-callable entry points (so Claude can fire from PowerShell):
- `RunSnapTest(scenario, fnName)` — single test
- `RunAllSnapTests` — sweep all 8 snap fns × 5 scenarios = 40 traces
- `RunOpenAndPlaceTest(snapPos)` — end-to-end OpenAndPlace flow against one alias
- `RunAllOpenAndPlaceTests` — sweep all 6 aliases through OpenAndPlace
- `DescribeForegroundWindow` / `DescribeForegroundWindowAfter(secs)` — one-shot diagnostic, captures whatever window is foreground
- `ListSnapScenarios` — print available scenario names

Analyzer is `py Diagnostics/analyze_snap_traces.py`, prints a table of `start | final | bounce | transit_ms` per run plus a bounce summary.

Full usage docs are inside the `ahk-functions` skill at `~/.claude/skills/ahk-functions/SKILL.md` (Testing section).

## What this catches

- Window monitor placement (start vs. final monitor)
- Min/Max state (`min_max=1`/`0`/`-1`)
- Outer window rect (x, y, w, h)
- Bouncing through intermediate monitors mid-snap
- Whether a function lands on the correct monitor across diverse starting states

## What this does NOT catch

- **Inner-paint glitches** — e.g. Notepad++'s Scintilla view rendered at the wrong size while the outer window rect is correct. `WinGetPos` only sees the outer rect.
- **Production-path differences** — `RunOpenAndPlaceTest` uses `-multiInst -nosession` Notepad++, while voice-fired `OpenNotepadFileAt` uses the existing instance. The voice path has different timing characteristics. See `df_snap_investigation.md`.
- **Anything cursor-position-dependent** — DisplayFusion's `WindowCurrentMonitorMaximize` uses cursor monitor, but the harness doesn't manage cursor position between runs. Tests can pass while production fails for this reason.

## Layer 2 (per-function classifier) — not built

The original plan proposed a `ahk-test` skill that classifies functions by side-effect type and runs appropriate post-checks per category. It hasn't been built and probably shouldn't be:

- Window-changing functions are well-served by the existing harness.
- App-opener verification is covered by `verify.ps1 process|window|active` already.
- Process-restart verification is rare enough to do ad-hoc.
- The per-category post-checks proposed in the original plan ended up being trivial in practice — the manual inspection approach hasn't been a bottleneck.

If a future bug class emerges that genuinely needs per-function post-checks, *then* build it.

## Future expansion ideas

If the harness needs to cover more cases, the natural extensions:

1. **More scenarios** — staged starting states beyond the current five. Add a function in `_THStageScenario` and a name in `_THSnapScenarios()`. ~10 lines.
2. **Different test targets** — replace `_THSetupNotepadPP` with `_THSetupExplorer` / `_THSetupCalculator` etc. for testing functions that aren't N++-specific. The capture/log functions are app-agnostic.
3. **Open/close lifecycle tests** — capture process state too (already in `_THCaptureWindowState`), assert app exits cleanly when closer fires.
4. **Focus/detect tests** — pure-read functions like `CheckWindowExists`, `GetActiveWindowTitle`. Would just be wrappers around `_THCaptureWindowState` + assertion.
5. **Pixel-correctness checks** — would catch the Notepad++ Scintilla paint glitch but requires screenshot diffing. Big lift, narrow benefit, defer until a second case justifies it.
