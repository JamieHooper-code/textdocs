---
tags: [programming, design-doc, ahk, autohotkey, miller, gui, performance, architecture]
created: 2026-09-07
status: phase-1-shipped
owner: Jamie
---

# Miller GUI performance — the diagnostic system + the Show Fun / Macro Wizard fix

**This is the designated source** for "a Miller/GUI feels slow to open or slow to
search" — read this before guessing, and add to it (don't fork a sibling doc) the
next time a menu is slow. Skill pointer: `~/.claude/skills/ahk-functions/SKILL.md`
symptom table + `references/miller-authoring.md`.

## The problem that started this (2026-09-07)

Jamie: "Show Fun" (`OpenVoiceCommandEditor`) and the Macro Wizard (`_SetMacroWizard_RunMiller`)
were "really slow" to open, and typing into the filter box (e.g. "press left") took
5-10 seconds to show anything. Every other Miller in the codebase is instant. Ask
was for a **properly designed system**, not three hacky point-fixes: (1) a
timing/profiling recorder built into the template itself, (2) a lazy loader for the
menus that load hundreds/thousands of items, (3) possibly a fast-literal + slow-AI
two-tier search.

**What it actually was, once measured — none of the three guesses:**
- Not "too many items" needing a lazy loader — the item COUNT (~1,970 functions)
  wasn't the cost.
- Not AI/semantic search — there is none in this path; the engine's own filter is
  plain substring matching and was already fast.
- It was **one shared data loader calling one specific 620 KB JSON file's parse,
  unconditionally, on every single open of both menus** — 10.5 seconds, every time,
  confirmed reproducible across three separate runs. Everything else (function
  list, filter, rendering) was already within a few hundred ms.

The lesson generalizes: **measure before designing the fix.** All three of Jamie's
guesses were reasonable and none were the actual bottleneck. The instrumentation
built here (part 1 of the ask) is what turned "feels slow" into "here is the exact
line," and is now permanent so the next slow Miller doesn't need this whole
investigation repeated.

## Part 1 — the permanent diagnostic system (what shipped, and it stays)

Every Miller now logs timing automatically, no per-menu wiring required. Read it
with `grep "Mcp/perf" ahk_event.log` (human-readable) or the `.jsonl` sidecar for
the precise `dur_ms` field (the pipe log's own timestamps are NOT reliable for
subtracting durations — always read `dur_ms` from the jsonl, never eyeball
adjacent line timestamps; unrelated log lines from other systems can land between
two of yours).

**In the engine** (`Helpers/Gui/MillerColumnPickGui.ahk`, `_MillerPerfLog` /
`_MillerPerfThresholdMs` near the top of the file — generic, usable from ANY file
in the include closure):

| Phase (log category `Mcp/perf/<phase>`, `/slow` suffix over threshold) | What it measures | Fires |
|---|---|---|
| `first_render` | Root/first-level catalog build + initial fill | Once, the first `_doRender()` call |
| `render` | Every later drill/back/refresh render | Every subsequent `_doRender()` |
| `open_total` | Function entry to first render done (control setup + first_render) | Once per open |
| `search_index` | Building the cached recursive-search universe (walks every root node, pulls every `search_index` provider) | Once per window session — the FIRST non-empty filter keystroke pays this; every keystroke after is a cache hit |
| `filter` | Total keystroke-to-redraw latency for one filter change (includes a nested `search_index` cost on the keystroke that triggers the cache build) | Every filter keystroke (80ms debounced) |

**In the shared Show Fun / Macro Wizard data loader** (`Helpers/VoiceCommandEditorMenu.ahk`,
`_VceLoadData()` — called by BOTH `OpenVoiceCommandEditor` and `_MacroWizardMiller`):
`vce_regen` (conditional Python regen of the voice↔function join index),
`vce_parse_voice_by_fn`, `vce_parse_fn_index`, `vce_load_data_total`.

**In the Registry Editor's loader** (`Helpers/RegistryEditorMenu.ahk`, `_RegLoadData()`):
`reg_parse_dump` — logged **unconditionally**, not just when slow, specifically so
a future regrowth of `registries_dump.json` shows up in the numbers instead of
just being felt again. This is the line that caught the whole 10.5s bug.

**Process-spawn cost** (`GuiHost/launch` in `Helpers/CommonFunctions.ahk`'s
`_LaunchGuiHost`, paired against the existing `DISPATCH/in` line for the same
function name in the freshly spawned process): the "process spawn + parse the
~200-file `#Include` closure" cost. Measured ~240ms for Show Fun — never the
bottleneck here, but check it first for any FUTURE slow-Miller investigation
before assuming the problem is inside the menu's own code.

### How to use this the next time a Miller feels slow

1. **Fire the menu once, then grep the jsonl for `Mcp/perf` (and any menu-specific
   loader categories) for that open** — `Get-Content ahk_event.jsonl | Select-String
   'Mcp/perf'` (PowerShell) is enough; no code changes needed, the instrumentation
   is already there.
2. Read `dur_ms` per phase. `open_total`/`first_render` in the hundreds of ms is
   normal; anything in the **seconds** is the thing to chase.
3. If a MENU-SPECIFIC loader (like `_VceLoadData`) is the slow one, add the SAME
   pattern there: wrap each real phase (a file parse, a Python shell, a network
   call) in `_MillerPerfLog(phase, A_TickCount - t0, msg)` — it's importable from
   anywhere in the include closure, not just `MillerColumnPickGui.ahk`.
4. **Don't guess which JSON file or which loop is slow — isolate it.** The
   fastest way, proven below: copy the suspect file to a scratch `.ahk`, add a
   tick-count global counter per reader function (`_JsonReadObject`/`Array`/
   `String`/`Number`), run it standalone against the real data file via
   `AutoHotkey64.exe` directly through **PowerShell** (`Start-Process`, NOT
   Git-Bash — see the gotcha below), and compare against a same-order-of-size
   file that's known-fast. A/B one change at a time.
5. If the fix is architectural (make something lazy), verify with the SAME probe
   script before/after — don't trust "it should be faster now," re-run and read
   the number.

### Reusable verification pattern (how this investigation was actually done)

A throwaway `.ahk` probe script that: launches the real menu via `MAINFUN.bat` (or
the direct GuiHost entry function), `WinWait`s for its title, `CoordMode("Mouse",
"Client")` + `Click` into the filter box at its known layout coordinates (the
filter Edit is built at client `x10 y58 w<leftW> h22` — auto-focus is deliberately
SKIPPED under a detected Claude/test launch, so a click is required), `SendText`
into it, sleeps to let the debounce settle, then `SendLevel(1)` + `Send("{Esc}")`
to close (cross-process key delivery to a Hotkey()-bound key needs the SendLevel
bump; plain text into a focused Edit control does not). Read `ahk_event.jsonl`
after. This is a legitimate, reversible, brief-focus-steal verification action —
same tradeoff `references/gui-testing.md` already documents for "driving
(key-sends) still briefly steals focus."

**Gotcha that cost real time this session: run AHK scripts via PowerShell, not
Git-Bash, for anything beyond `ahk.py validate`/`run` (fire-and-forget).**
`ahk.py run` without `--wait` does not surface a launch failure — a wrong path
silently no-ops with zero output and zero error. `cmd.exe /c "..."` invoked FROM
Git-Bash gets its quoting mangled (the same MSYS translation class of bug as the
documented `/validate` gotcha) and can silently fail to dispatch at all with no
error and no log line — indistinguishable from "it's just slow" until you check
for a lingering process and find none. When in doubt, `Start-Process
<exe-or-bat> -ArgumentList ... -WindowStyle Hidden` from PowerShell, `Start-Sleep`,
then read the result — the same pattern CLAUDE.md already prescribes for
`MAINFUN.bat`, just as true for a raw `AutoHotkey64.exe` scratch probe.

**Another gotcha, also cost real time: AHK v2's `>`/`<`/`>=`/`<=` are numeric-only
between values that don't both look like plain numbers, and a mis-shaped
comparison throws** — an attempt to replace a per-character regex
(`c ~= "[0-9.eE+\-]"`) with `(c >= "0" && c <= "9") || ...` threw on every
non-digit character and, with no `OnError` handler in the scratch script, hung
forever on an invisible (WindowStyle Hidden) MsgBox — a process that looks "just
slow" but is actually dead-blocked. Use `InStr("0123456789.eE+-", c)` (membership,
not ordering) for this kind of char-class check instead.

## Part 2 — the Show Fun / Macro Wizard fix (case study, 2026-09-07)

### Root cause

`OpenVoiceCommandEditor` (show fun) and `_MacroWizardMiller` (the set-macro
wizard) both call the SAME loader, `_VceLoadData()` in
`Helpers/VoiceCommandEditorMenu.ahk`. That loader ALSO unconditionally called
`_RegLoadData()` (`Helpers/RegistryEditorMenu.ahk`) — the Registry Editor's own
data load — so show fun's bottom "Registries" branch could show a live count, and
so every registry entry could fold into show fun's recursive search (the
"unify search" feature from `VOICE_COMMAND_SYSTEM.md` §16.6).

`_RegLoadData()` reads a **cached** dump (`~/.claude/context/registries_dump.json`,
620 KB as of this writing: 53 registries, 1,523 flat entries) and parses it with
the codebase's own hand-rolled parser (`Helpers/JsonFunctions.ahk` `JsonParse`).
Measured, reproducibly, **10.5 seconds** for this one file — vs. 172-672ms for
two other files in the SAME open path that are a similar order of byte-size
(314 KB and 382 KB). That one call was **>90% of the total open time** for BOTH
menus — and the set-macro wizard **never even shows a Registries branch**, so
100% of its share of that 10.5s was pure waste.

The filter box itself was never the problem: engine-measured filter latency was
already 78-150ms per keystroke (see Part 1's `search_index`/`filter` categories)
— well under any perceptible threshold. Jamie's "typed and waited 5-10s" was
almost certainly the tail of the ~11.4s OPEN still finishing while she was
already typing, not the search itself being slow.

### What is (and isn't) understood about WHY that one file is slow

Bisected down to the string reader specifically (`_JsonReadString` in
`JsonFunctions.ahk`) via a counter-instrumented copy of the parser run against
the real file (see "reusable verification pattern" above):

- Ruled out: `FileRead` (0ms), `StrSplit`/char-array setup (32ms), `_JsonSkipWs`
  (93ms across 83,797 calls — trivial), `_JsonReadNumber`'s per-char regex
  (removed entirely in an A/B test — no change), escape density (only 1,509
  escape-loop iterations across 16,065 string reads — negligible).
- **Fixed and kept, but NOT the dominant cause for this file:** `_JsonReadString`
  had a genuine O(remaining-file-length) bug — `b := InStr(s, "\", true, state.i)`
  scanned for the next backslash ANYWHERE IN THE REST OF THE FILE rather than
  bounded to the string's own closing quote, so a file with sparse backslashes
  relative to its size could pay a scan-to-EOF cost per string. Fixed by bounding
  the search to a `SubStr` of just `[state.i, q)`. Verified against all 306 pure
  unit tests (incl. the dedicated `test_json_functions.ahk` suite covering escape
  edge cases) — safe, but re-measuring after the fix showed **no measurable
  change** for registries_dump.json specifically (still ~10.5s), so whatever this
  file's disproportionate per-character cost in the string reader actually is,
  this bug was not it, or not the majority of it.
- **Still unexplained:** per-character cost in `_JsonReadString`'s bulk-copy fast
  path measured **~59x slower** for registries_dump.json (10.9 μs/char) than for
  `fn_to_file_index.json` (0.185 μs/char) in the same process, same fixed parser,
  despite registries' strings being SHORTER on average (28 vs 46 chars) — ruling
  out "just longer strings, still linear." Object/array bookkeeping
  (`_JsonReadObject`/`_JsonReadArray`, `Map()`/`[]` construction, `Push`/key
  insertion) also accounts for a large, not-fully-isolated share once nested
  inclusive-timing double-counting is subtracted out. **This is an open
  question** — flagged for a future session if it recurs elsewhere (see "Related
  open items" below). The counter/timer harness in this doc's verification
  pattern is the starting point; don't re-derive it from scratch.

### The actual fix (architectural, not dependent on solving the mystery above)

Given the parser-internals cause wasn't fully pinned down, the fix that shipped
is **structural** and works regardless: **stop paying the cost eagerly.**

1. `_VceLoadData()` no longer calls `_RegLoadData()` at all. Show fun's "Registries"
   branch is mounted **unconditionally** (previously gated on an eager
   `VceHasRegistries` check — chicken/egg, since knowing whether registries exist
   required loading them) with a **generic detail line** (no live count) instead
   of the old `"<N> registries"`.
2. New guard `_RegEnsureLoaded()` in `RegistryEditorMenu.ahk` — a load-once-per-
   process flag around `_RegLoadData()`. The Registries branch's `children`
   builder calls it, so drilling in for the first time in a session pays the
   10.5s once (lazily), never again that session. `OpenRegistryEditor()` (the
   standalone "open registry" entry point) still calls `_RegLoadData()` DIRECTLY,
   unguarded — it intentionally wants a fresh read every open, and is UNCHANGED
   by this fix (see "Related open items" — it still pays the full 10.5s on every
   own use).
3. **The subtle one:** the Registries branch's `search_index` was REMOVED, not
   just made lazy. The Miller engine's recursive-search universe build
   (`_Mcp_GetSearchUniverse` / `_Mcp_WalkNode`) pulls EVERY mounted branch's
   `search_index` provider UNCONDITIONALLY the first time the search universe is
   built (the session's first non-empty filter keystroke) — regardless of what
   the user is searching for. Leaving `search_index` on the Registries branch
   while making its DATA lazy would only have moved the 10.5s stall from "at
   open" to "on the first keystroke you ever type into show fun's search box" —
   objectively worse for the reported complaint (typing lagged). The branch now
   carries `search_skip: true` instead (same pattern as every function branch),
   so it's a single searchable ROW ("registries registry sites docs contacts
   directories send slots entries") but its contents are never live-walked or
   index-pulled during a search. **Trade-off knowingly accepted:** a registry
   entry (e.g. a specific Google Doc) is no longer findable by typing its name
   into show fun's search box — only by drilling the "Registries" row manually,
   which is the one action that still pays the lazy 10.5s. Restore the
   `search_index` if the underlying parse cost ever gets fixed for real, or if
   the dump is restructured/shrunk (see the redundancy note below).
4. Kept the `_JsonReadString` backslash-bound fix (Part 2 above) — a real
   correctness-preserving perf improvement, low risk, may matter for some OTHER
   large/sparse-escape JSON file even though it wasn't decisive here.

### Verified results (re-run through the Part-1 instrumentation, not eyeballed)

| Phase | Before | After |
|---|---|---|
| `vce_load_data_total` (shared loader; BOTH menus pay this) | 11,391 ms | 672-688 ms (**~17x**) |
| `open_total` (Miller engine construction) | 1,172-1,250 ms | 719-735 ms |
| `search_index` (first keystroke, universe build) | 78-94 ms (registries WAS already counted in the walk before the fix broke it out separately) | 32 ms, and now structurally CANNOT regress to 10s+ since Registries is `search_skip` |
| `filter` (full keystroke-to-redraw) | 125-141 ms | 78 ms |
| Total time from process spawn to a usable, searchable window | ~11.5s+ | **~1.5s** |

Macro Wizard was not independently re-probed end-to-end (its GuiHost entry needs a
live device/slot context that's harder to fake safely), but it shares the
identical `_VceLoadData()` call with zero Registries-branch code of its own, so
the ~17x improvement in `vce_load_data_total` applies to it unconditionally, by
construction — it was never affected by the `search_index` change since it never
had a Registries branch to search through.

**Regression checks run:** `_run_unit_tests.ahk` (306/306 pass, both before AND
after each change), `MAINFUN.bat RunGuiTests miller` (18-19/19 pass across two
runs — the one intermittent failure was a different test each run with a
window-handle-timing error, a pre-existing GuiTestSendKeys flake unrelated to
these changes, not a logic assertion failure), full-repo `ahk.py validate --repo`,
`ahk_include_closure.py`, `ahk_warnings.py check` — all clean.

## Related open items (not fixed here — flag if they resurface)

- **`OpenRegistryEditor()` ("open registry") still pays the full ~10.5s on every
  own open.** Untouched by this fix on purpose (it wants fresh data every time,
  and touching its freshness contract was out of scope for a Show-Fun/Macro-
  Wizard ask). If Jamie reports "open registry is slow," this doc + the counter
  harness above is the starting point — the fix would need to actually solve the
  parser mystery (or split registries_dump.json's redundant triple-representation
  — see below) rather than just deferring, since this entry point has no "later"
  to defer to.
- **registries_dump.json stores the same ~1,523 entries in THREE overlapping
  shapes** in one file: a flat `registries` array, an `entries` Map keyed by
  registry id (each value an array of that registry's entries), and a `flat`
  array (literally all entries again, ungrouped). That's ~21,629 total JSON
  VALUES for what is conceptually ~1,523 records — worth asking whether the
  Python side (`Scripts/codebase_tools/registries dump generator — wherever
  `registries_dump.json` is written from`) actually needs all three, or whether
  the AHK consumers could derive `flat` from `entries` (or vice versa) in AHK
  instead of shipping it three times over the wire. This alone might solve both
  the mystery AND `OpenRegistryEditor`'s remaining slowness without ever finding
  the exact quadratic line.
- **Show Fun's `first_render` (~550-700ms) builds ~1,970 function branches
  eagerly** — real cost, much smaller than what was fixed, not chased further
  this session (diminishing returns per Jamie's own call 2026-09-07). If it's
  ever worth shaving: the pattern to copy is the SAME lazy/`search_skip` +
  cached-flat-`search_index` split already proven fast elsewhere (see "Menus
  that already do this right," below) — but function branches are cheap
  closures, not a data load, so the win would be smaller and the win-per-effort
  ratio may not be worth it.

## Menus that already do this right (the pattern to copy for anything new)

Cross-referenced during this investigation as the working contrast to what Show
Fun/Macro Wizard were doing wrong — copy these, don't reinvent:

- **`Helpers/RegistryEditorMenu.ahk`** itself (irony noted) — `_RegSearchIndex`
  just wraps the already-flat `RegFlat` array, zero recursion, zero Python in the
  filter path. The DATA load being slow is the bug; the SEARCH-INDEX pattern
  around it was always right.
- **`Helpers/SpotifyLibraryManager.ahk`** — lazy `children` (shells Python only
  when a level is actually drilled into) PLUS one explicitly-documented one-shot
  `search_index` ("Shelled ONCE per search session — the engine caches the
  universe"). The clearest statement in the codebase of the intended pattern.
- **`Helpers/MediaHubMenu.ahk`** — same `search_index` pattern, Python calls
  confined to leaf action handlers, never the search path.

Full mechanics of `search_index` / `search_skip` / lazy `children`:
`~/.claude/skills/ahk-functions/references/miller-authoring.md` §§ "Recursive
search" and "Refresh after an action."

## Related code

- Engine instrumentation: `Helpers/Gui/MillerColumnPickGui.ahk` (`_MillerPerfLog`,
  `_MillerPerfThresholdMs`, and the four call sites: `_doRender`,
  `_Mcp_GetSearchUniverse`, `_onFilterChange`, plus the `open_total` line after
  the initial `_doRender()` call)
- Shared loader: `Helpers/VoiceCommandEditorMenu.ahk` (`_VceLoadData`,
  `_VceAllFunctions`, `_VoiceCommandEditorRootNodes`'s Registries mount)
- Process-spawn timing: `Helpers/CommonFunctions.ahk` (`_LaunchGuiHost`)
- Registry loader + the lazy guard: `Helpers/RegistryEditorMenu.ahk`
  (`_RegLoadData`, `_RegEnsureLoaded`)
- The JSON parser fix: `Helpers/JsonFunctions.ahk` (`_JsonReadString`)
- Macro Wizard (unchanged, benefits for free): `Helpers/MacroWizardMiller.ahk`
- Related design docs: [[MILLER_LAYOUT_SYSTEM]] (the sibling machine-wide Miller
  system this pattern lives alongside), `AutoHotkey/docs/gui-conventions.md`
  (full template + opts reference)
