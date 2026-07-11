---
tags: [autohotkey, uia, recorder, codegen, macro-system, caster, context-manager, design]
created: 2026-07-09
updated: 2026-07-09
status: FULLY BUILT — recorder · primitives · normalizer · wrapper-reuse · codegen · auto-regen-hook
owner: Jamie
related: [KEY_ACTION_FUNCTIONS, ALWAYS_ON_CONTEXT_DETECTOR, STREAMDECK_WORKFLOW_OVERHAUL]
---

# UIA Interaction Recorder → Normalizer → Codegen

**Goal:** record a click/tap/type/voice-command walkthrough of any app, and turn it
into a robust, context-scoped AHK function — ideally **fully automatically for simple
click-throughs** (no Claude), and with rich enough context that when Claude *is*
needed, Jamie sends the recording + one line ("make this into a function for when I
open X") and Claude has everything it needs — no hand-written paragraphs of context
(the pain point from the Anki-fix session).

Legend: **[BUILT]** done & verified · **[NEXT]** agreed, not yet built · **[LATER]** designed-for, deferred.

---

## The pipeline (the whole architecture in one picture)

```
  RAW CAPTURE                NORMALIZER                 CONSUMERS
  (AHK host)                 (Python)                   
  ┌──────────┐   JSON   ┌──────────────────┐   IR    ┌─ human summary  (NOW: for Claude)
  │ recorder │ ───────► │ filter / dedup / │ ──────► ├─ generated key-action  (LATER: no Claude)
  └──────────┘          │ resolve-context/ │         │     └── steps call ──► UIAActions.ahk primitives
                        │ classify actions/│         └─ (full JSON stays on disk = deep-dive tier)
                        │ annotate-REUSE   │◄── uia_element_registry.json
                        └──────────────────┘         (derived from the wrapper fns)
```

**Key architectural decision (locked):** the enricher is a **normalizer that emits a
structured action IR**, and the human summary is just *one rendering* of that IR. A
future codegen script is a *second consumer* of the same IR — not a rewrite. Costs
nothing now because we're building the normalizer anyway; we just build it IR-first.

---

## Stage 1 — Recorder  [BUILT]

Own-process AHK host (`Scripts/UIAClickRecorderHost.ahk`), toggled by voice
**"make record"** → `Helpers/UIAClickRecorder.ahk` (`ToggleUIAClickRecord`, generic-store
command). KindleGrab-style own process (persistent hooks + live tooltip can't live on
the MAINFUN hot path).

Unified, timestamped timeline merged from **five** sources so input method never matters:

| kind | source | notes |
|---|---|---|
| `click` | `~LButton` hook + `ElementFromPoint` | physical / remote mouse |
| `focus` | `UIA.AddFocusChangedEventHandler` | **the touch + synthetic-macro solution** — a tap on a Chromium control or a synthetic coordinate-click emits no mouse event, but DOES move UIA focus, reported regardless of input source |
| `key` | non-suppressing `InputHook` | modified + special keys (F11, Ctrl+Shift+P, Esc) |
| `type` | same InputHook | plain typing aggregated into one entry |
| `ahk_fn` | merged from `ahk_event.log` `DISPATCH/in` at stop | voice/SD/Claude-triggered MAINFUN fns, interleaved by timestamp |

**Outputs:** full JSON `INIDATA/UIARecordings/<window>_<stamp>.json` (everything) +
minimal summary → clipboard + Claude paste slot (`INIDATA/CopyPasteSlots/claude.txt`).

**Hard-won gotchas baked in (do not regress):**
- **Never touch a ValuePattern in a capture loop.** Both `el.Value` AND `el.Dump()`
  (its cache request includes "Value") build a live `IUIAutomationValuePattern` that
  crashes in `__Delete` on edit controls ("Property ptr not found"). We read only
  plain, pattern-free props (type/name/automationId/className/helpText/rect).
- **`OnError` safety net** swallows + logs any stray UIA/COM error → no modal popup
  ever reaches Jamie mid-recording.
- **Reentrancy guard** (`REC_BUSY`): the focus handler and click capture must not
  reenter each other → COM reentrancy hang (bit us when a test created a window).
- **Re-resolve the window each poll**; a handle grabbed during a cold launch can be a
  dead transient window with a forever-empty UIA tree. (See [[uia-clicking-debugging]].)
- Webview links ignore UIA `Invoke` → use a real center-click. (See uia-clicking.md.)
- All AHK→Python handoff files written `UTF-8-RAW` (a BOM breaks `json.load`).

Verified working: mouse, remote mouse, **touch** (13-event touchscreen run matched the
16-event macro run), keyboard, and ahk_fn merge (MakeDump dispatches interleaved).

---

## Stage 2 — Normalizer / Enricher  [BUILT 2026-07-09]

**`Scripts/uia_recording_report.py`** — reads the raw JSON, emits (a) the
structured **action IR** (`--json`) and (b) the human summary rendered from it
(default stdout). The AHK host stays the dumb capture layer. Verified on a
synthetic multi-context recording (chrome → youtube → code): context blocks,
show-fun cards (via `ahk_search.load_ahk_entries`), transition collapse, context
segmentation, and wrapper-reuse annotation all render correctly.

**Gotcha found + fixed while building:** the recorder can't capture a URL, so
`url_contains` contexts are resolved by looking for the site label in the window
TITLE. Two traps: (1) use the **first** host label (`calendar`.google.com), not
the last — the last is often the generic `google`, which appears in every
"Google Chrome" title; (2) **strip the trailing " - Google Chrome" browser
suffix** before matching, or a `url_contains: "google.com"` context (hint
`google`) false-matches the word Google inside "Google **Chrome**". Both live in
`_url_hint` / `_strip_browser_suffix`.

### 2a. Filtering (Jamie's nuance: keep load-timing intel, cut churn)
- **Collapse** a run of `loading` / repeated `document` refocuses into a single
  **`⏳ page transition — may need wait/scan`** marker (keeps the "something loaded
  here → insert a wait" signal; that marker becomes a `UiaWaitForName`/`ClickThenWaitFor`
  in codegen).
- **Drop** only genuinely-dead dups (same element focused twice back-to-back with
  nothing between) and anonymous (no name + no aid) noise.
- Everything is preserved in the on-disk JSON regardless — filtering only shapes the IR/summary.

### 2b. Context resolution — PER EVENT (critical for multi-context)
Each event carries its window (title/exe/class/url). Resolve **each** event → a
**Context Manager** token (`INIDATA/Contexts/<token>.json`, matched on exe/class/
title_regex/url_contains — the same matching the always-on detector uses; see
[[ALWAYS_ON_CONTEXT_DETECTOR]]). Then:
- **segment** the timeline into contiguous same-context runs,
- **mark transitions** with the causing action (`AltTab()`, an open command, a taskbar
  click) — a recording that opens prog A, acts, closes it, opens B shows clean context
  boundaries. Jamie's constraint: **multiple contexts inside one recording is normal.**

### 2c. Enriched output (locked design, 0.00)
```
=== RECORDING: youtube · 16 events · 2026-07-09 12:07 ===

APP CONTEXT  (Context Manager: youtube)
  matching:   chrome.exe · url youtube.com/watch   parent: chrome
  program:    (open/launch fns if the context has them)
  voice:      scoped ON · ListNav site profile "youtube" (jump/next/focus)
  bindings:   5 macros · stream deck: "YouTube"
  files:      Helpers/YouTubeFunctions.ahk · INIDATA/Contexts/youtube.json

FUNCTIONS SEEN  (show-fun cards, reusing ahk_search.py's renderer)
  ListNavClickNth(n)  (Helpers/ListNavFunctions.ahk:380)
    summary: click the Nth item in the active list   aliases: jump <n> …   tags: listnav

TIMELINE  (filtered)
  [1] +0.0s  focus button "Account menu"     [chrome: youtube/watch]
  [2] +2.2s  ⏳ page transition — wait/scan here
  [3] +0.2s  focus group "Switch account"
  [6] +2.0s  AHK fn: ListNavClickNth("2")
```
- App-context block = **curated subset** (matching, program fns, voice/listnav/bindings/SD,
  file paths) — NOT the full 12-facet Context Manager dump.
- Relevant files = **paths + function `file:line`** (Claude opens what it needs), not full
  file contents.
- `show fun` cards for the **ahk_fn's in the timeline + the context's program/open fns**,
  rendered by reusing **`ahk_search.py`**'s card format.
- Two tiers: **clipboard = curated, bounded**; **full JSON on disk = everything**.

---

## Stage 3 — UIA primitives library  `Helpers/UIAActions.ahk`  [BUILT 2026-07-09]

The shared **runtime** every UIA function (hand-written or generated) calls — so we
never reinvent per-program. Generalized from the verified Anki helpers.
`#Include`d in MAINFUNCTIONS.ahk after OpeningAndClosingFunctions; full include
tree validates clean. Built:

- `UiaClickByName(type, name, timeout, winMatch)` — poll (re-resolving window) + real center-click.
- `UiaFindByName(...)` / `UiaWaitForName(...)` — the polling core / wait-for-present (no click).
- **`UiaClickThenWaitFor(clickType, clickName, expectType, expectName, timeout, retries, winMatch)`**
  — Jamie's retry idea: click X, poll for expected result Y (page loaded / panel opened),
  retry the click if Y doesn't appear (already-past-this-step short-circuit if Y is
  present and X is gone). **The** robust click-through primitive; folds in
  wait-for-load + verification + retry.
- `UiaFocusByName` (SetFocus, click fallback), `UiaTypeText`, `UiaTypeIntoByName`.
- `UiaResolveWindow(winMatch)` / `UiaActivateWindow` — largest-visible-window,
  re-resolved each poll (the cold-launch transient-handle guard).

Carries every gotcha: window re-resolution each poll, real-click (not Invoke),
activate-before-click, no ValuePattern access, mm:0/cs:0. `winMatch` defaults to
`"A"` (active window); pass an exe match (`"ahk_exe chrome.exe"`) for robustness
against cold-launch transients.

---

## Stage 2.5 — Generic wrappers + element registry  [BUILT 2026-07-09]

**Jamie's ask (2026-07-09):** stop pasting the same raw selector into nine
macros. Write ONE generic wrapper per commonly-clicked control (e.g.
`YouTubeClickProfileIcon()` in `Helpers/YouTubeFunctions.ahk`, calling
`UiaClickByName("Button", "Account menu", …)`); then when a NEW recording clicks
that same element, the system **recognizes it's already wrapped and wires the
wrapper in automatically** — so a UI change is fixed in ONE place, not nine.

**The key idea that makes it not-a-pipedream: the registry is DERIVED from the
wrappers, never hand-maintained.** The wrapper is the single source of truth.

- **`Scripts/uia_element_registry.py`** scans `Helpers/*.ahk`, brace-tracks each
  function body, and extracts the `(control-type, name)` first-args of every
  `UIAActions` primitive call (`UiaClickByName`, `UiaClickThenWaitFor`'s click
  pair, `UiaFindByName`, `UiaWaitForName`, `UiaFocusByName`, `UiaTypeIntoByName`).
  Emits `INIDATA/uia_element_registry.json`: `"type|name|context" → {fn, …}` plus
  an unscoped `"type|name"` fallback (only when a single wrapper owns that
  control — collisions are reported, not silently written). Context per wrapper
  comes from an optional `; @context: <token>` comment, else a filename→token
  heuristic (`YouTubeFunctions.ahk → youtube`, `VSCodeFunctions.ahk → code`).
- **The normalizer consumes it:** each recorded click/focus is looked up
  (`type|name|context` first, then `type|name`); a hit annotates the timeline
  step `= FnName()  <- existing wrapper` and sets `reuse_fn` in the IR. Codegen
  (Stage 4) then emits the wrapper CALL instead of a raw selector.

**Feasibility of the "auto-connect" loop:** exact `(type, name[, context])`
recognition is a deterministic lookup — fully automatable, works today. What
still needs Claude: fuzzy matches (the control's Name changed / localized),
deciding *which* new clicks deserve to become wrappers, and naming them. So the
practical loop is: recorder + registry surface "this is `YouTubeClickProfileIcon`"
automatically; Claude promotes genuinely-new repeated clicks into new wrappers on
request. Regenerate the registry whenever wrappers change (same cadence as the
`.ahk.meta.json` sidecars). Verified: extractor parses annotation + all primitive
shapes + filename inference; normalizer lights up the reuse tag on a seeded hit.

---

## Stage 4 — Codegen  [LATER]

**Target the Key Actions system — do NOT build a parallel generator** (see
[[KEY_ACTION_FUNCTIONS]]). It already:
- groups by **context** (Chrome, YouTube watch page, General) → that IS the per-context
  placement (answers "where do generated fns go" — no new `functions_file` field needed),
- compiles JSON step defs → real AHK functions with editor (`OpenKeyActionEditor`) +
  `show fun` + voice/SD wiring,
- **already supports a verbatim function-call step** (`Name(args)`, existence-guarded).

So a recorded UIA click **is** a key-action step: `UiaClickThenWaitFor("Button","Account menu","group","Switch account")`.
Codegen = normalize IR → emit key-action steps (`UiaClickThenWaitFor(...)`, `sleep(500)`,
`<ahk_fn>(...)`) into the right context group via `gen_key_actions.py add <Name> "<steps>" --group <context>`.
Recordings **become** editable key-actions in the Key Action Editor, scoped to their
context. Multi-context recording → multiple grouped key-actions, or one orchestrator with
context-switch steps between segments.

Escaping element names with quotes/specials into step strings is the main impl detail.

---

## Feasibility verdict

**Not a pipedream.** Simple deterministic click-throughs ("open app → click these
named controls → type → keys → done") are fully automatable: the recorder already
captures robust UIA selectors + timing + context; codegen emits `UiaClickThenWaitFor`
steps + waits at the transition markers + `ahk_fn` calls, into the context's Key Actions
group. **Still needs Claude:** conditional logic, retry/error strategy beyond the
primitive, ambiguous/dynamic selectors, "do X until Y." Minority of cases.

**#3 (guardrail):** silent auto-create-and-wire is acceptable — Jamie wires it, if it
works it works, if not she sends the recording to Claude to fix. No "show code first" gate.

---

## Build order (status)

1. ✅ **`Helpers/UIAActions.ahk` primitives** — BUILT + validated (include tree clean).
2. ✅ **`Scripts/uia_recording_report.py` normalizer** — BUILT + tested on a REAL
   recording (context block + show-fun cards + filtered/segmented timeline + reuse tags).
3. ✅ **`Scripts/uia_element_registry.py`** (Stage 2.5) — BUILT + tested.
4. ✅ **`Scripts/uia_make_wrapper.py`** — BUILT + tested. Turns a one-click "make
   record" recording into a named wrapper: picks the real target out of the tap's
   focus-churn (drops initial-focus, clicks > focuses, actionable types only),
   generates the `UiaClickByName(...)` wrapper with `; @context:`, `--write` appends
   it to the matching `*Functions.ahk` + regens the registry. **Proved the whole loop
   on real data:** recorded the YouTube avatar → `uia_make_wrapper.py
   YouTubeClickProfileIcon --write` created it in `YouTubeFunctions.ahk` → re-running
   the normalizer on the same recording now auto-tags that click
   `= YouTubeClickProfileIcon()`.
5. ✅ **Host wired** — `UIAClickRecorderHost._Rec_FinalizeCore` now runs the normalizer
   at finalize and puts the ENRICHED summary on the clipboard + Claude slot, with a
   bulletproof fallback to the built-in terse summary (make-record can't break on it).
6. ✅ **`Scripts/uia_codegen.py`** (Stage 4) — BUILT + tested end-to-end. Turns a
   multi-step recording into an editable **Key Action** in the recording's context
   group: reuse matches → the `WrapperFn()` call (not a raw selector), unmatched
   clicks → `UiaClickByName(...)`, keys → Caster combos (`Ctrl+Shift+P`→`cs-p`),
   typed runs → `text(...)`, `ahk_fn`s verbatim; churn/markers dropped (the polling
   primitives handle waits). `--write` writes a proper steps-array to
   `key_actions.json` + runs `gen_key_actions.generate()` → real compilable AHK
   (verified: the throwaway compiled with the wrapper call + SendText + KeyActionSend
   + FullSend + UiaClickByName). Multi-context recordings group by primary context.
7. ✅ **Registry auto-regen wired into the Stop hook** — `caster_ahk_verify`'s new
   `uia_registry` check regenerates `uia_element_registry.json` on any `.ahk` change
   (WARN-only; surfaces >1-wrapper conflicts). Wrappers self-register; no manual step.

**The whole pipeline is now built.** Remaining is refinement, not new stages: broaden
the actionable-type filter as real recordings expose gaps; smarter transition→wait
(vs relying on `UiaClickByName` polling); per-segment key-actions for multi-context
flows (currently one action grouped by primary context).

**Jamie's workflows now:**
- *One-click wrapper:* record → tap one control → stop → `py -3 Scripts/uia_make_wrapper.py <FnName> --latest --write`.
- *Whole flow → function:* record the click-through → `py -3 Scripts/uia_codegen.py <FnName> --latest --write` (dry-run first without `--write`).
- *Hand to Claude:* record → paste the enriched clipboard summary with "make this into a function for when I open X".

## Decisions locked (this session)
- Enricher via Python post-processor; IR-first; summary is a rendering of the IR.
- Curated context block (not full facet dump); file paths + `file:line` (not full contents).
- show-fun cards for timeline ahk_fn's + context program fns; reuse `ahk_search.py` renderer.
- Filter: collapse→annotate page transitions, drop dead dups only; full JSON keeps everything.
- Per-event context resolution + segmentation + transition markers (multi-context first-class).
- Codegen targets Key Actions (per-context groups); no `functions_file` field.
- UIA primitives library is the shared runtime incl. `UiaClickThenWaitFor` (retry/verify).
- Silent auto-wire OK; Jamie is the test/fix safety net.

---

## RETROSPECTIVE — the YouTube account-switch build (2026-07-09)

Building `YouTubeToggleAccount` (switch natehoop↔jamieeehooper, set Language
Reactor + captions per channel) took ~8 painful round trips. It worked in the
end, but the pain was mostly avoidable and the lessons are the point.

### Why it hurt
1. **I guessed selectors from the recording SUMMARY instead of reading the raw
   tree.** Every breakthrough came the instant Jamie sent a `make dump` and I read
   the *indented* `uia.txt`. Everything in between (positional click, "topmost
   name", "row above View all channels") was elaborate logic on an unconfirmed
   mental model — each wrong model = one full test-and-frustrate cycle for Jamie.
2. **The recorder's data was actively MISLEADING for this UI.** Touch → UIA focus
   landed on the enclosing **container group**, whose `Name` is *every descendant
   concatenated* ("Back Accounts Jamie Hooper … Jamie Maybe … Add account"). So the
   recording said "you touched the whole panel," not "you tapped Jamie Maybe." The
   real rows were **unnamed groups** with the label in a child `[text]`.
3. **The UI lies.** `"Jamie Hooper"` appears 3× (header + two Other-accounts); the
   caption control reads `"…captions unavailable"` *while captions display*;
   `ToggleState`/name are unreliable; the caption button is a `toggle button` type
   that `FindElements({Type:"Button"})` doesn't even return; Language Reactor
   injects its control async after reload. Nearly every property we'd normally
   trust was wrong.
4. **My discipline failure:** when Jamie corrected the model, I *patched* rather
   than fully re-grounding on the tree. Should have re-read the whole dump each
   correction.

### What finally worked (the durable techniques)
- **The `caption-window-1` element** (present only while captions render) was the
  reliable "captions on" signal when the button name/ToggleState lied.
- **Structure over names:** anchoring on the unique **email** link, then clicking
  the row group beneath it — not the colliding channel name.
- **Observable functions:** logging every signal (`CC/state | toggle=… showing=…`)
  and tooltipping every decision is the ONLY reason we could debug remotely.
- **Polling for async-injected controls** (LR mounts seconds after reload).

### Decision framework — when NOT to automate a UI
Count the red flags on the target BEFORE committing:
- repeated / ambiguous accessible names (can't target by name),
- clickable elements are unnamed containers, labels in separate children,
- current-relative / stateful structure (menu changes by state),
- lying properties (name/ToggleState don't reflect reality),
- async third-party injection.

**3+ red flags ⇒ expect a fragile, high-maintenance automation.** Then it's a
value call: a *frequent* action you care about is worth it (+ occasional
re-fixing); a rare one isn't — do it by hand. **And check for a non-UI path
FIRST** — keyboard shortcut, URL/deep-link, or app API beats clicking a webview
every time. We reached for UIA reflexively; that's a reflex to break.

### Codified improvements (roadmap — Jamie said do all 4, 2026-07-09)
1. **Dump-first gate + red-flag checklist** → `ahk-functions` UIA skill docs
   [DONE this session]. Rule: for any Chromium/webview menu, read the indented
   tree of the OPEN menu before writing a selector. No guessing from a summary.
2. **Recorder captures more, on disk, on-demand** [DONE 2026-07-09 — Jamie's
   "gather everything, access when necessary" idea]. Two parts, both built:
   - Per-event enrichment [DONE]:
     - The host (`UIAClickRecorderHost.ahk`) now records **`invokable` /
       `toggleable`** (IsInvoke/TogglePatternAvailable — safe boolean property
       reads, NOT a ValuePattern build) on the tapped element AND every subtree
       node. This is the one signal only the live tree has, and it's what
       identifies a **clickable UNNAMED group** — the exact webview trap (a
       control-type heuristic can't tell an account-row group from a layout
       group; the Invoke flag can). *Populates on the next `make record`;
       older recordings degrade gracefully (no flags → no clickability claims).*
     - The normalizer (`uia_recording_report.py`) surfaces control **type**
       (PascalCase ctype) + **AutomationId** on every timeline line, tags
       `[invokable]`/`[toggleable]` and `[not clickable itself -> target a
       child]`, and detects a **concatenated-container** (`is_concat_container`:
       Name == ≥2 child names joined) → loud `!! NAME IS CONCATENATED CHILD
       TEXT` warning so that name is never used as a selector.
     - The inline subtree preview now leads with **clickable** nodes (named or
       not); an unnamed invokable row is labelled by its child text
       (`group "natehoop@gmail.com" [invokable] (via child text)`) — precisely
       the data that would have prevented the account-switch disaster.
   - **Per-interaction tree snapshots as ARTIFACTS** [DONE]. For each container
     interaction with a real subtree, the normalizer writes an indented
     `<recording-stem>_event<N>.txt` next to the recording (in the `make dump`
     `uia.txt` line format: `[type] Name="..." AID="..." <flags> [l,t WxH]`) and
     references its path in the summary. Two-tier: lean preview inline (≤8
     clickable/named leaves), full tree on disk, opened only when building that
     selector. Written by `write_event_artifacts()` from `main()`, so `--json`
     (codegen) sees the paths too. Capped by the host's subtree bound (depth 5 /
     70 nodes) so files stay small.
3. **"Observable by default"** [convention, DONE in docs]. Every UIA function
   logs the signals it reads + tooltips its decision, from line one — the first
   failure hands us diagnostic data, not "it didn't work."
4. **`make dump menu`** helper [DONE 2026-07-09]. A scoped dump that lists ONLY
   the actionable elements (Invoke/Toggle-pattern, on-screen, non-container) of
   an open menu, specificity-ranked (smallest first) in the exact format Claude
   needs — kills the "read 1000-line uia.txt" step for the common case.
   - `Diagnostics/UIAMenuDump.ahk` captures every element + pattern flags of the
     active window to JSON (safe boolean reads, like the recorder host).
   - `Scripts/uia_menu_dump.py` filters to clickable/on-screen/non-container,
     ranks by area, and labels unnamed clickable rows by the named element
     spatially inside them (`Group "natehoop@gmail.com" (via child text)`) —
     reuses the recorder normalizer's `_node_area`/`_node_flag`/`control_type_name`.
   - `MakeDumpMenu()` (ProgrammingDebuggingFunctions.ahk) runs both, dedups
     co-located wrapper/leaf stacks, and drops the list on the clipboard + Claude
     slot. Voice phrase: **"make dump menu"** — `MakeDump("menu")` intercepts the
     keyword and delegates (the natural phrase collides with `make dump
     [<textnv>]`, so the slot captures "menu" as a name; the delegation honours
     the phrase with no new command / no reboot). You can no longer name a normal
     dump literally "menu".
   - **Scoping = mouse-free DELTA (the hard part).** A webview overlay menu has
     NO window boundary — it's inline DOM — so a one-shot active-window capture
     returned the WHOLE page (512 actionable of 874 on the account menu). Jamie
     is voice-only (no mouse), so cursor-scoping is out, and focus doesn't
     reliably land in a menu the instant it opens. Solution: two-phase diff.
     Say "make dump menu" once (menu closed → baseline), open the menu, say it
     again → output is only the NEW elements = the menu. Named/aid'd elements are
     matched by IDENTITY (type+name+aid) so page scroll between captures doesn't
     create false positives; unnamed elements fall back to coarse position.
     Baseline auto-expires after 120s. Validated live: isolates the account menu
     (LR control, both emails, View all channels / Add account / Sign out) from a
     scrolled page perfectly (9 of 874 elements).
   - **Chaining:** after each delta the baseline ROLLS FORWARD to the current
     state, so clicking through nested menus and saying "make dump menu" at each
     shows just that step's newly-appeared elements (one baseline at the start,
     delta-since-last thereafter).
   - **Co-located dedup is shared:** `_dedup_colocated` (collapse wrapper/leaf
     stacks at one rect to the named leaf) now lives in the recorder normalizer
     and is used by BOTH the menu dump and the recorder's per-interaction subtree
     preview — the one piece of menu-dump logic that belonged in the recorder too.
   - **Run grouping + persistence [DONE].** All deltas of one session accumulate
     into a SINGLE grouped dump-store folder (`%TEMP%\dumps\menu-run-<ts>\`),
     auto-numbered "Menu 1, Menu 2, …" (no manual numbering, no per-app phrases —
     Jamie explicitly did NOT want either). Clipboard/Claude slot always hold the
     whole grouped run; revisitable via "open dumps"; the run is the baseline's
     120s life, so an unrelated menu later starts fresh. Empty deltas are skipped.
   - **`make dump here` + `make dump mouse` [DONE].** Two point-scoped inspectors
     for a no-mouse user, both delegated from `MakeDump(...)`, both OBSERVABLE (the
     tooltip + first output line name exactly what got scoped, e.g. "scoped to:
     group 'Account switcher' — via focused element (last tapped)").
     - **The empirical finding (2026-07-09):** Windows moves the mouse cursor on a
       touch tap for NATIVE chrome but **NOT inside a webview** — so cursor-scoping
       is dead for YouTube/webview content, exactly where Jamie needs it.
     - **`make dump here` = FOCUS-scoped** (`--focus`): roots at
       `UIA.GetFocusedElement` walked up to the enclosing panel. UIA focus DOES
       follow a touch tap into a webview (that's why the recorder works on Jamie's
       touch-driven YouTube recordings), so this is the webview-capable, no-mouse
       path. Tap the panel, say "make dump here".
     - **`make dump mouse` = POINT-scoped** (`--point x y`): wiggles the mouse,
       roots at `ElementFromPoint`. Only reliable in native chrome (where touch
       moves the cursor); kept for that case.
     - Shared core `_DumpScoped(mode)`; both use `UIAMenuDump.ahk` +
       `uia_menu_dump.py` (scope-area used as the container reference so the panel
       itself is dropped). For a menu that triggers on tap, the delta stays the
       right tool.
   - **The recorder remains the scoped path for macro-building** (it bounds to the
     tapped subtree automatically, no baseline dance); `make dump menu`/`here` are
     the standalone inspectors.

### The seamless workflow (both of us)
- **Dynamic UI → Jamie `make dump`s the OPEN state → I read the `uia.txt` FIRST.**
  The tree is unambiguous; screenshots + prose were not. Lead with the dump; I
  should *demand* it, not guess.
- **I build observable from the start**, so failure #1 produces data.
- Recorder improvement #2 folds the dump INTO the recording, so eventually one
  `make record` carries everything and the separate `make dump` step disappears.
- Generic wrappers are the single source of truth for a selector; the element
  registry is DERIVED from them (never hand-edited), so one edit fixes every macro.
- Reuse recognition is exact `(type, name[, context])` (automatable now); fuzzy
  matches + deciding what becomes a wrapper stay Claude-in-the-loop.
