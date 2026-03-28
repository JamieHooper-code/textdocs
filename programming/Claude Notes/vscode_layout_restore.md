# VS Code Layout Restore

## The Problem
When switching VS Code between the main monitor and the smaller touchscreen monitor and back, the sidebar width and editor group proportions shrink and don't restore. Voice command: "fix view" (was "code restore").

**Broken things:** sidebar width, editor group split proportions (NOT zoom/font size).
**Constraint:** prefers no VS Code restart; "Reload Window" (~2-3s) is acceptable reluctantly.

---

## Key Technical Findings

**`storage.json`** at `%APPDATA%\Code\User\globalStorage\storage.json` is plain JSON and stores `sideBarWidth` (currently 170px — already shrunken) and `auxiliaryBarWidth` (378px — secondary sidebar). This is the programmatic hook for sidebar width.

**No VS Code setting exists for sidebar width.** `window.sidebar.defaultPanelWidth` does NOT exist — that was AI-generated misinformation from codegenes.net. The GitHub issue for this feature (microsoft/vscode#158603) is still in backlog.

**`workbench.action.resetEditorGroupSizes`** — built-in command, works fine, resets editor splits. Bound to `Ctrl+Shift+Alt+R` in keybindings.json. This part works.

**`workbench.action.increaseViewSize` / `decreaseViewSize`** — only work when the PRIMARY sidebar (file explorer) is focused. Do NOT work when secondary sidebar (Claude Code panel, auxiliary bar) is focused. This caused confusion during testing.

**Double-click the resize border** — built-in VS Code trick, auto-sizes to content. Could be AHK-driven but imprecise.

---

## What Was Built (current state, partially working)

Files modified:
- `%APPDATA%\Code\User\keybindings.json` — added `Ctrl+Shift+Alt+R` (resetEditorGroupSizes), `Ctrl+Shift+Alt+P` (decreaseViewSize), `Ctrl+Shift+Alt+;` (increaseViewSize)
- `VSCodeFunctions.ahk` — `VSCodeRestoreLayout()`: focuses sidebar → collapses 35x → expands 14x → resets editor groups. Storage.json approach commented out below it.
- `vscode_commands.py` — "fix view" voice command added
- `%APPDATA%\VSCodeLayouts\VSCodeLayout.py` — Python helper for save/restore via storage.json (written but not in use)

**What doesn't work:** The Claude Code secondary sidebar never grows with the resize commands — only the primary sidebar responds. Result: running "fix view" resizes the file explorer sidebar but ignores the Claude Code panel width entirely.

---

## Approaches Tried / Evaluated

| Approach | Status | Notes |
|---|---|---|
| `increaseViewSize` loop after focusing sidebar | Partially works | Only affects primary sidebar, not secondary (Claude Code) |
| `storage.json` swap + `Developer: Reload Window` | Built but not tested | Requires timing the JSON write after VS Code flushes on reload. Python helper exists. |
| `storage.json` swap + full restart | Not tried | Clean but ~10s disruption |
| VS Code extension for layout save/restore | No good option found | "Layout Saver" extension only saves tab groups, not sidebar widths |
| `window.sidebar.defaultPanelWidth` setting | Doesn't exist | Misinformation, confirmed via official docs |

---

## FINAL WORKING SOLUTION ✅

**Two-phase approach via `VSCodeRestoreLayout()` in VSCodeFunctions.ahk. Voice command: "fix view".**

Key insight: `increaseViewSize`/`decreaseViewSize` only affect whatever two things are currently competing for space with the focused view. Claude Code (secondary sidebar) can be resized by isolating it with the editor first.

**Phase 1 — Claude Code width:**
1. `Ctrl+\` — hide primary sidebar (editor and Claude Code now split the space)
2. Focus editor (`Ctrl+1`)
3. `increaseViewSize` × 35 — maximize editor, push Claude Code to minimum
4. `decreaseViewSize` × 9 — shrink editor, Claude Code grows to target width

**Phase 2 — Primary sidebar width:**
5. `Ctrl+/` — re-show primary sidebar
6. Focus file explorer (`Ctrl+Shift+E`)
7. `decreaseViewSize` × 35 — collapse sidebar to minimum
8. `increaseViewSize` × 3 — grow to target width

**Keybindings still active in keybindings.json:**
- `Ctrl+Shift+Alt+P` → `decreaseViewSize`
- `Ctrl+Shift+Alt+;` → `increaseViewSize`

**Calibration values that worked:** 9 (Claude Code), 3 (primary sidebar). Adjust loops if monitor layout changes.

---

## Options Still To Try (priority order)

1. **Open Claude Code as a permanent second editor group** (Jamie's idea — may be the best UX)
   - Instead of using the secondary sidebar at all, open Claude Code in a split editor group
   - Editor group widths ARE controllable via `resetEditorGroupSizes`
   - This sidesteps the entire auxiliary bar problem
   - Try: drag Claude Code panel out of secondary sidebar into an editor group

2. **storage.json + Developer: Reload Window** (already built, just needs testing)
   - `VSCodeLayout.py` already written at `%APPDATA%\VSCodeLayouts\VSCodeLayout.py`
   - AHK commented-out code is in `VSCodeFunctions.ahk`
   - Need to verify timing: does writing storage.json AFTER WinWaitClose but BEFORE renderer restarts actually stick?
   - If it works, uncomment the storage.json approach and wire it up

3. **DisplayFusion auto-trigger** (automation option)
   - DisplayFusion can run a script when monitor profile changes
   - Could auto-call `VSCodeRestoreLayout()` whenever switching monitor configs
   - No manual trigger needed

4. **Write a minimal VS Code extension** (most precise, most work)
   - Extension could save/restore sidebar widths directly via VS Code's internal APIs
   - Probably 1-2 hours of work
   - Would be the cleanest long-term solution

---

## Why the Current Approach Partially Fails

VS Code has two sidebars: the **primary sidebar** (file explorer, source control, etc.) and the **auxiliary/secondary sidebar** (where Claude Code extension panel lives). `increaseViewSize`/`decreaseViewSize` only resize whichever sidebar currently has keyboard focus, and the keybindings only trigger when sidebar focus is active. The AHK function focuses the primary sidebar (`Ctrl+Shift+E`) and resizes that — but the Claude Code panel is in the secondary sidebar and is unaffected.
