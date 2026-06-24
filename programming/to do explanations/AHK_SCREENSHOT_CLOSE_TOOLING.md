---
tags: [todo, ahk, tooling, claude, gui-testing]
related: ["[[VOICE_COMMAND_SYSTEM]]", "[[AHK_FUNCTION_TESTING]]"]
status: partially-done
updated: 2026-06-23
---

# AHK screenshot + close tooling — make GUI inspection safe and clean

**The problem (recurring, bit Jamie repeatedly up to 2026-06-23):** when Claude
inspected an AHK GUI via `ahk.py show` and then wanted to clear a stray window, it
reached for `Get-Process AutoHotkey64 | Stop-Process -Force`. That image name is shared
by Jamie's ~5 **always-on** scripts (MAINFUNCTIONS + every persistent macro), so the
blanket kill **destroyed her entire macro system** and she had to restart everything.

## Already done (2026-06-23)

- **`ahk.py close "<title>"`** — new subcommand: posts `WM_CLOSE` to just the matching
  GUI hwnd(s), requires a title, never kills a process. The safe way to clear a stray
  test window. (`~/.claude/helpers/ahk.py`.)
- **Hard rule documented** in `~/.claude/skills/ahk-functions/references/gui-testing.md`:
  never `Stop-Process` AutoHotkey64; close the WINDOW (`show --close` or `close <title>`),
  verify with `check` (read-only), `sweep` only for #32770 popups.

## Done (2026-06-23, round 2)

- **`ahk.py close --tests`** — closes ONLY `ClaudeTestWindow`-tagged windows (everything
  `show` opens; tag = `GetPropW(hwnd,"ClaudeTestWindow")`). Verified: launched the Registry
  Editor, `close --tests` closed exactly it, persistent scripts intact. This is now the
  recommended "clear my mess" button.

## Still to build (the "better screenshot + closing function")

1. **Track the launched PID through `show`** and expose a `close --last` that closes the
   exact window THIS invocation spawned (by pid→hwnd) — niche now that `--tests` exists;
   only matters if you want to close one of several test windows. Lower priority.
3. **Auto-close stale test windows** at the start of the next `show` (or on a short TTL),
   so leftover inspection windows never accumulate and never tempt a process kill.
4. **`show` could return structured info** (pid, hwnd, png path) as JSON on `--verbose`
   so a caller can close precisely without re-enumerating.
5. **Screenshot ergonomics:** optional multi-pane / scrolled capture for long Miller
   lists (today a single PrintWindow grabs one viewport); a `--annotate` overlay of the
   focused row; dark-mode toggle baked into `show` for theme checks.

## Why it matters / how to apply

Claude's GUI-inspection loop must be *non-destructive by construction*. The `close`
subcommand + the hard rule close the acute hole; items 1–3 make it impossible to even
express the dangerous operation (you close tagged test windows, never "all AHK"). Build
these before the next big Miller/GUI iteration session. See also [[AHK_FUNCTION_TESTING]].
