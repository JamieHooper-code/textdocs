---
tags: [autohotkey, caster, claude, vscode, uia, migration, streamdeck, context, design]
created: 2026-08-22
status: in-progress
owner: Jamie
---

# Claude Code: VS Code extension → desktop app migration

Full cutover from the **VS Code Claude Code extension** to the **standalone Claude desktop app**. Driven by the VS Code UI, not by any capability gap — Jamie doesn't need the integrated editor and doesn't want the layout system.

**Strategy: additive, not destructive.** The old VS Code automation stays intact and working. The new system is a fresh, disjoint namespace built against a better UIA surface. Nothing is deleted until the new path has replaced it in daily use.

Related: [[ALWAYS_ON_CONTEXT_DETECTOR]] (the detector the new context plugs into) · [[CONTEXT_MANAGER]] (the context editor) · [[STREAMDECK_WORKFLOW_OVERHAUL]] (the deck buttons that need repointing) · `ahk-functions` skill `references/directory-program-registry.md`

---

## The app is not what you'd guess

| Fact | Value | Why it matters |
|---|---|---|
| Install type | **MSIX / Microsoft Store** | `C:\Program Files\WindowsApps` is ACL-locked. `Run()` on the exe path is **access-denied**. |
| Package | `Claude_1.34493.1.0_x64__pzs8sxrjxfjjc` | |
| PackageFamilyName | `Claude_pzs8sxrjxfjjc` | |
| **AUMID** | `Claude_pzs8sxrjxfjjc!Claude` | The only supported launch route: `explorer.exe "shell:AppsFolder\<AUMID>"` |
| Protocol handler | `claude://` registered in HKCR | Unexplored. Candidate for `open_fn` (open a specific folder) — see Open questions. |
| exe | `Claude.exe` | |
| class | `Chrome_WidgetWin_1` | **Shared with VS Code, Obsidian, Slack, Stream Deck.** Class alone can never discriminate — match on `exe`. |
| Window title | literally `"Claude"` | No project, no session, no state. **Every contextual fact must come from UIA.** No title-matching shortcuts, unlike VS Code's `AHK_caster_workspace - ...`. |

There is *also* a `claude.exe` shipped inside the VS Code extension
(`.vscode\extensions\anthropic.claude-code-*\resources\native-binary\claude.exe`) — that's the CLI, it owns no window. A naive `Get-Process claude | Select -First 1` grabs the wrong one. Filter on `MainWindowTitle`.

## Why the new surface is better

The VS Code panel automation was fragile because it was scraping an editor sidebar. The desktop app exposes semantic, named controls.

| VS Code side | Desktop side | Verdict |
|---|---|---|
| `_ClaudeSessionRows` reconstructing flyout geometry | `[button] Name="Idle Grab Album"` — every session is a **named button with status in the Name** | Straight win |
| `Scripts/claude_sessions.py` (361 lines) mirroring `extension.js` title precedence (`customTitle‖aiTitle‖lastPrompt‖summary`) + reading the sidebar's `sessionID` memento | The sidebar **states the live title directly** | **Whole layer collapses** |
| `ClaudeChatTracker.ahk` (327 lines) MRU stack for "swap chat" | Native `Back` / `Forward` buttons | **Replaced by an affordance** |
| `_ClaudeFocusMessageInput` — multi-strategy hunt | `[edit] Name="Prompt"` | One FindElement |
| `_ClaudeFocusPanel` sending `Ctrl+Shift+8` (an *extension* keybinding) | Real top-level window | Normal snapping applies |
| `IsClaudeAuxBarMaximized` / `EnsureClaudeAuxBarMaximized` | n/a — it's its own window | Delete |
| No status visibility at all | `Idle` / `Awaiting input` / `Needs input` per row, plus `Usage: context 0, plan 19%` | **New capability** |
| Projects invisible | Sidebar groups sessions by folder (`desktop-important`, `Downloads`, `user-caster`) with `New session in <folder>` | **New capability** |

### The anchoring rule

Match on **Name + control type**. The app's AutomationIds are React-generated (`_r_5p_`, `_r_17_`, `base-ui-_r_86_`) and will shift between releases — never make one load-bearing. Names are semantic and stable: `Prompt`, `Search`, `New`, `Send`, `Back`, `Forward`, `Artifacts`, `Customize`, `Collapse sidebar`, `Home`, `Code`.

### The `mm: 0` bug (inherited)

`UIA.MatchMode` is `{StartsWith:1, Substring:2, Exact:3, RegEx:"RegEx"}`. **`0` is not a valid value.** The old `VSCodeFunctions.ahk` helpers pass `mm: 0` in 19 places, wrapped in `try` — so those lookups likely threw and silently returned no match rather than matching exactly. This plausibly explains a chunk of the old panel's flakiness. The new code uses documented values only.

---

## What's built (2026-08-22)

### `Helpers/ClaudeDesktopFunctions.ahk` — new, namespace `ClaudeDesk*`

Disjoint from the old `Claude*` / `VSCode*` names so both systems coexist. Included from `MAINFUNCTIONS.ahk` directly after `VSCodeFunctions.ahk`.

**Primitives:** `CDHwnd` (cross-desktop), `CDRoot`, `CDFocusRoot`, `CDFind`, `CDClickEl` (Invoke → real center-click fallback; webviews accept Invoke silently without acting), `CDClick`.

**Session list:** `CDSessionRows()` parses every sidebar Button whose Name matches
`^(Idle|Awaiting input|Needs input|Working|Running|Thinking|Error)\s+(.+)$`,
then **sorts by screen Y**. Sorting on position rather than trusting tree order is load-bearing: **titles repeat** (two `Add Anthropic Skills to Plugin Marketplace` rows in the reference dump), so position is the only reliable identity for "the 3rd one".

**Verified live** — 15 rows parsed, order and duplicates exactly matching the sidebar.

| Function | Does |
|---|---|
| `OpenClaudeDesktop(snap)` | `launch_fn`. Focus if running, else launch by AUMID + wait. 600ms settle after window appears — the window exists before the web layer paints and the UIA tree is empty until then. |
| `ClaudeDeskJump(n)` / `Next` / `Prev` | Session by position |
| `ClaudeDeskOpenByName(words)` | Exact title, then substring fallback |
| `ClaudeDeskNew` / `Search` / `Filter` / `Back` / `Forward` / `Menu` / `Artifacts` / `Customize` / `Sidebar` | One-click sidebar controls |
| `ClaudeDeskModeCode` / `ModeHome` | Mode toggle |
| `ClaudeDeskPrompt` / `Send` / `Type(text, submit)` | Prompt box |
| `ClaudeDeskStatus` | Summary + names of anything not Idle |
| `ClaudeDeskUsage` / `ClaudeDeskModel` | Read the chips |
| `ClaudeDeskDumpRows` | **Diagnostic.** Writes parsed rows to `%TEMP%\claude_desk_rows.txt`. Reach for this first when a jump lands wrong or a new status word appears — it shows the parse, not a guess at it. |

### `INIDATA/Contexts/claude_desktop.json` — new context

Token `claude_desktop`, matched on `exe: Claude.exe`. Gets context detection, Stream Deck auto-switch, scroll config and typing box **for free, with no code**.

`open_fn` is deliberately **empty** (status: `launch-only`, same as Chrome) — opening an arbitrary folder programmatically is unsolved. See Open questions.

---

### Opening folders — the deep link, and what it can't do

`OpenFolderInClaudeDesktop(path, snap)` is the context's `open_fn`, so `open <directory>` routes here. `ClaudeDeskResume(uuid)` imports a CLI session.

**Two things bit us, both worth remembering:**

1. **A new folder raises a modal, not an error.** The first deep link into an untrusted path opens an `alert dialog` — *"Trust this workspace? Claude Code may read, write, or execute files in this directory."* with `Cancel` / `Trust Workspace`. It is one-time per folder, but while it is up the app ignores further deep links, which makes subsequent calls look like silent no-ops. If a link "does nothing", **look for the modal before debugging the URL.**

2. **The deep link is single-root.** The main process forwards every `folder` param (`searchParams.getAll('folder')`), but the new-task screen renders exactly ONE folder chip — passing five yields one. Verified by firing 5 roots, then 2, then 1, navigating away between attempts so no call could be dismissed as a repeat-navigation no-op. `claude://code/new?folder=A&folder=B` gives you A.

3. **`src=external` means the folder is PROPOSED, not committed — and that is deliberate.** The handler appends `src=external` whenever a deep link supplies a folder. The composer then *displays* the chip but does not *commit* it: a trust dialog fires, and even after accepting, sending the first message bounces back to the folder picker — while the folder already shows as selected. An outside program silently handing a code-executing agent a working directory is exactly what that gate exists to stop, so **do not auto-accept it**; that would be defeating a security control on every button press.

The way past it is to not need it. The app remembers its folder across restarts (Jamie: *"when I reopen Claude Code it does not give me the prompt again"*), so when you are already on the target folder there is nothing to switch and a plain focus suffices. `CDOpenFolderSmart` reads the folder chip and only deep-links on an **observed** mismatch:

- chip == target → focus, no prompt
- chip unreadable (a session is open, so the composer row is off screen) → **treat as fine, focus only.** Deep-linking on unknown would fire the gate every time a session happens to be open, which is the common case and the exact friction this avoids.
- chip != target → deep link; the prompt is the honest cost of a real switch, once.

`CDCurrentFolderName` reads the chip by anchoring on the **named** "Local" button and taking the next button to its right on the same baseline — positional, but keyed off a named sibling, so it survives window moves and resizes.

**The actual multi-root mechanism is `permissions.additionalDirectories`** in the PRIMARY root's `.claude/settings.json` — the same thing the VS Code extension was doing when it passed the workspace roots through as additional working directories. `caster/rules/.claude/settings.json` now declares the other four roots explicitly; that file, not the deep link, is the real replacement for the 5-folder `.code-workspace`.

So `ClaudeDeskWorkspace()` opens the **first** folder listed in the `.code-workspace` file and lets settings.json supply the rest. It still reads that file rather than hardcoding a path, so the two stay in sync while both systems coexist.

### `OpenAHKCasterWorkspace` — the name was moved, not changed

Jamie's most-used button (Deck A `2,2`). Nothing else called it — no voice rule, no binding — so the bare name was reassigned to the Claude Code path and the button followed with no edit. The VS Code body is preserved verbatim as `OpenAHKCasterWorkspaceInVSCode`. Verified end-to-end: fires, no trust prompt, folder chip reads `rules`.

### The Stream Deck "75 references" was wrong

Grepping button text for `Claude` counted the **send system**, not VS Code automation: those buttons are `SendClaudePrompt <name>` / `EditClaudePrompt <name>`, one per prompt. `ClaudeFunctions.ahk` has **zero `Code.exe` references** — sends paste into whatever is focused, with one special case (`IsForegroundContext("dragon")` → paste-only, no auto-Enter, for the Dictation Box). **All 69 send buttons work in the desktop app unchanged.**

The genuinely VS Code-bound deck buttons are only ~6, all on the "VS Code" profile: `ClaudeToggleChat`, `ClaudeNewChat`, `ClaudeBackChat`, `ClaudeSearchChats`, `ToggleAutoAccept`, `OpenAndFocusVsCodeWithClaude` — plus `ClaudeNextChat` at `[7,3]`, which is **pinned from the default page** and therefore global across every profile. Watch that one: repointing it changes every deck.

### ListNav — the sidebar as a navigable list

The sidebar is now a ListNav profile, so `jump N` / `next` / `prevy` / `focus me` work and the standard `list-navigate-*` deck groups apply.

**Getting the row selector right took two tries, and the first one was a real trap.** Session rows and the date headers (`Today`, `Yesterday`, `Aug 19`, `Older`) are the SAME control Type, and their sizes overlap:

| Element | Width | Height |
|---|---|---|
| `Today` / `Older` (carry a trailing button) | 236 | 24 |
| **`Yesterday` / `Aug 19`** (no trailing button) | **264** | 24–25 |
| Session rows | **263** | 26–27 |

So a width filter is not merely fragile here — it is *impossible*. `min_width: 250` let `Yesterday` (264) through and `jump 1` clicked the word "Yesterday". Height differs by a single pixel, which is no better.

The only reliable discriminator is that a session row's Name carries a **status prefix** (`Idle …`, `Awaiting input …`) and a date header never does. That is a pattern, not a substring — which ListNav could not express.

**Engine change:** `_LN_MatchMode(sel)` in `Helpers/ListNavFunctions.ahk` adds `"match_mode": "regex"` → UIA `mm: "RegEx"`, used at all three selector sites. Additive — `contains` still maps to 2 and everything else still maps to 0, so every existing profile's comparison is byte-identical. Result: `found=15 kept=15`, zero headers, and `jump 5` lands on Grab Album exactly as expected.

> **Latent bug worth a separate look:** `match_mode: "exact"` maps to `mm: 0`, and `0` is not a valid `UIA.MatchMode` (`{StartsWith:1, Substring:2, Exact:3, RegEx:"RegEx"}`) — UIA's comparison chain ends in `throw Error("Invalid MatchMode")` for it. The same `mm: 0` appears 19 times in `VSCodeFunctions.ahk`, always inside a `try`, which would make those lookups silently return no match. Most working profiles use `contains` or omit `Name`, which is likely why nothing has visibly broken. Deliberately NOT changed here — it touches every profile and deserves its own test pass.

### Deck link — done

`streamdeck_profile: "Claude"` on the context. A "Claude" Deck A profile already existed (Code / Co-work / Chat / Search / Stop / Settings hotkeys) but had no switcher button, so it was absent from `vsd_profile_positions.json`. `vsd-ensure "Claude"` registered it at `[3,1]`; verified live: `ContextDetector/change [] -> [claude_desktop]` → `Vsd/switch invoked button 3,1`.

**A context-registry edit needs `ReloadWithNotice`** — the always-on detector caches the profile set at boot, so a new `Contexts/*.json` is invisible until it reloads. This cost a confusing "detector says `[]`" round.

**Open decision:** attaching `list-navigate-numbers` (ListNavClickNth 1–8, at `[4,2]`–`[7,2]` and `[4,3]`–`[7,3]`) collides with three buttons already on the Claude profile — `Chat [6,2]`, `Chat [4,3]`, `Search [6,3]`. Not attached; Jamie's call.

## Still to do

1. **Deck buttons** — decide the `list-navigate-numbers` collision above; repoint the ~6 VS Code-bound chat buttons to their `ClaudeDesk*` equivalents (`ClaudeDeskNext`/`Prev`/`New`/`Search`/`Back`). Remember `ClaudeNextChat [7,3]` is pinned from the default page and is therefore global.
2. **Caster** — extend `claude_commands.py` scope (its docstring already names `claude.exe`); ListNav voice already works via the `_has_listnav_profile` predicate, but a **`reboot caster`** is needed once so the predicate re-derives its exe set and picks up `Claude.exe` (first non-Chrome/Signal app). Rules scoped `executable="Code"` that are Claude-specific need re-scoping to `Claude`.
3. **Typing box coords** — `code.json` carries VS Code's panel geometry (`x:1267, y:172, w:1352, h:1614`). The new context needs its own once she picks a window size.
4. **Footpedals** — explicitly deferred (Jamie, 2026-08-22).

### Probably DELETE rather than port

- **`Scripts/AutoAcceptLoop.ahk` (226 lines) + both accept pedals.** Her `~/.claude/settings.json` already sets `"defaultMode": "bypassPermissions"` and `"skipDangerousModePermissionPrompt": true`, and the app exposes a `Manual` mode button (`_r_4a_`). If the desktop app honors that shared settings file, there is nothing left to auto-accept. **Verify before deleting.**
- **`ClaudeChatTracker.ahk` (327 lines)** — superseded by native Back/Forward.
- **`Scripts/claude_sessions.py` (361 lines)** — superseded by the sidebar stating titles directly. *Caveat:* it also provided stable **UUID** identity across renames; the sidebar gives title only. If stable identity turns out to matter, keep the `~/.claude/projects/<slug>/<uuid>.jsonl` half and drop only the `extension.js` title-precedence half.
- **VS Code layouts** (`VSCodeOpenLayout` / `RestoreLayout` / `BuildUserDataArgs`, `--user-data-dir` based) — no equivalent, and Jamie doesn't want it.

## Open questions

- Does the desktop app read `~/.claude/settings.json`? (Decides whether AutoAcceptLoop dies.)
- Does `claude://` accept a path or session id?
- Does the app expose the active session in the tree at all? Currently `CDCurrentIndex()` tracks it in `%TEMP%\claude_desk_index.txt` because **no row is marked active** in the dump — worth re-checking on a version bump.
- Are the multi-root folders (`Local` / `Important` chips + `Add another folder`) per-session or global? Determines whether the `.code-workspace` 5-folder set needs recreating per session.

## The workspace question — resolved, it was never the problem

`AHK_caster_workspace.code-workspace` is 5 folder paths plus `powershell.cwd`. Nothing else. The VS Code extension passes those roots to Claude Code as **additional working directories** — the same mechanism as `permissions.additionalDirectories` in `settings.json`, which she already uses for `.claude`. No file access is lost in the move.

The one thing to be deliberate about: the desktop app takes **one folder as primary**, and that choice determines which `CLAUDE.md` loads as project instructions, which `.claude/settings.json` applies, and the session/memory slug (`c--Users-jamie-AppData-Local-caster`). Keeping `caster/rules` primary preserves existing history and project memory keys.
