# AHK Function Testing — Tooling for Claude

## Why this exists
Claude struggles to *test* AHK functions after writing them. The dispatch
path (Caster -> Python -> MAINFUN.bat -> MAINFUNCTIONS.ahk -> helper) has
several places where a call can silently no-op, and Claude has no
canonical recipe for firing a function one-shot from the shell and
confirming it actually did something.

Symptoms seen in real sessions:
- Running `cmd /c MAINFUN.bat OpenFlux` from the wrong working directory
  (MAINFUN.bat is not on PATH), so the call silently does nothing.
- `BringApp` against a tray-only app (f.lux) appears to succeed because
  the process is running, but no window surfaces — Claude has no
  after-state check to notice.
- Claude guessing flags on verify.ps1 that don't exist (`ahk-check`) and
  burning turns on trial-and-error.

The fix is a two-layer tooling plan.

---

## Layer 1 — baseline, every AHK task
Add an `ahk-run` subcommand to `C:\Users\jamie\.claude\helpers\verify.ps1`.

Shape:
    verify.ps1 ahk-run <FunctionName> [arg1 arg2 ...]

Behavior:
1. cd to the AutoHotkey dir (wherever MAINFUN.bat actually lives — resolve
   it once, cache the path in the script).
2. Snapshot the tail of `mainfun_calls.log` before firing.
3. Invoke MAINFUN.bat with the function name + args, CREATE_NO_WINDOW.
4. Wait briefly, then diff `mainfun_calls.log` to confirm a new dispatch
   line appeared for this function.
5. Return structured output: `OK|ahk-run|<FunctionName>|dispatched` or
   `ERROR|ahk-run|<FunctionName>|no-dispatch`.

This alone would have caught the MAINFUN.bat cwd bug immediately instead
of silently passing.

Claude should use `verify.ps1 ahk-run` as the default way to invoke an
AHK function for testing — not raw `cmd /c MAINFUN.bat`.

Reference memory to update after building this:
`~/.claude/projects/.../memory/reference_ahk_functions.md` — add the
`ahk-run` invocation as the canonical test command.

---

## Layer 2 — per-function verification skill (`ahk-test`)
A skill that reads the target AHK function's source, classifies what
*should* change when it runs, and runs the matching post-check.

Classification + check table:

| Function type | Example | Post-check |
|---|---|---|
| App opener (tray-capable) | `OpenFlux` | Poll `Get-Process <exe>`; assert `MainWindowHandle -ne 0` within timeout |
| App opener (normal) | `OpenAndFocusVSCodeWithClaude` | Assert target exe is foreground window + optional UIA panel check |
| Window snapper | `SnapWindowLeft`, `SnapCenterHalf` | Capture active window rect before/after; assert new rect matches expected monitor zone |
| Process restart | `RestartStreamDeck`, `RestartDragon` | Assert old PID is gone and a new PID exists for the target exe |
| Stream Deck control | `QuitStreamDeck` | Assert StreamDeck.exe process state changed as expected |
| Text insertion | `TextInsertion*` | Focus a scratch buffer (Notepad), fire, read buffer contents back, assert match |
| Pure keystroke wrapper | thin `Send("...")` helpers | Trust Layer 1 dispatch — no extra check, too noisy |
| Chrome tab opener | `OpenChromeTabs`, `OpenGmail*` | Poll Chrome for a new tab matching the expected URL pattern |
| URL opener with layout | `OpenMyCurrentStuff` | Same as above + assert window count/positions |

Skill workflow:
1. Take a function name as input.
2. Read the function body from `AutoHotkey/Helpers/*.ahk`.
3. Classify (grep for `Run`, `Send`, `WinActivate`, `WinMove`, process
   control patterns, etc.).
4. Run Layer 1 (`verify.ps1 ahk-run`) to fire it.
5. Execute the matching post-check.
6. Report PASS/FAIL with the specific assertion that ran.

Scope boundary: the skill is **opt-in per task**, not automatic. Pure
keystroke-wrapper functions don't benefit from it and would just add
noise. Use it when a function has observable side effects that matter.

---

## Order of operations
1. Build Layer 1 first — it's a single subcommand addition to verify.ps1
   and every future AHK task benefits immediately. Update
   `reference_ahk_functions.md` memory to point at it.
2. Scope Layer 2 as a follow-up once Layer 1 is in place and a few real
   AHK tasks have exercised it. The skill's classification table should
   be seeded from whatever functions Claude has actually needed to test
   by then, not designed in the abstract.

---

## Related context
- `_DEBUGGING_GUIDE.py` in the caster rules dir — existing Caster<->AHK
  debug doc (different scope: Caster-side chain + log files + rdescript).
  `ahk-test` is the missing **post-dispatch verification** layer that
  guide doesn't cover.
- `mainfun_calls.log` — already written by the dispatcher; Layer 1 just
  reads it.
- `reference_ahk_functions.md` memory — already lists Claude-callable
  AHK functions and verify.ps1 check tools. Update when Layer 1 ships.
