---
tags: [autohotkey, streamdeck, caster, macro-system, bindings, keyboard, design, dsl]
created: 2026-06-30
status: v1-built
owner: Jamie
related: [STREAMDECK_WORKFLOW_OVERHAUL, ALWAYS_ON_CONTEXT_DETECTOR]
---

# Key-Action Functions — JSON-defined, Caster-syntax macros as first-class AHK functions

**Goal:** define a keyboard shortcut (or a short macro) as a **named action purely in JSON**, using Caster's `Key()` syntax (`c-t`, `cas-t`), and have it become a **real, first-class AHK function** — so ONE definition is reusable by Stream Deck buttons, voice commands, and the binding system, is searchable in `show fun` / `ahk_search`, shows in the function/recent-functions log, and is bindable globally or per-context. Edit the JSON, regenerate, and it changes **everywhere at once**. This replaces the broken set-macro-wizard "capture hotkey → `send:<captured>`" flow ([[STREAMDECK_WORKFLOW_OVERHAUL]] §4).

Legend: **[V1]** first build · **[FUTURE]** captured, designed-for, not yet built.

---

## The core decision: GENERATE real AHK functions (not a runtime interpreter)

A key-action must behave like every other function (searchable, `show fun`, recent-functions, MAINFUN-callable, bindable). The only way to get all of that for free is for it to **be** a real AHK function. So a **generator** compiles `key_actions.json` → real `.ahk` function files (mirroring how voice commands are materialized from a store). Consequences — all good, and the reason this future-proofs cleanly:

- **Function-call steps become native.** A step that is `SnapPrimaryLeft()` just emits that call verbatim — because the generated file is real AHK, calling existing functions is free. **This is why "include other real functions later" changes nothing structural.**
- **Indexing is automatic.** The Stop-hook indexer scans `.ahk` files, so generated actions land in `codebase_inventory.md` + the qmd `functions` collection + `.meta.json` sidecars with args — no special-casing. (Generator must emit into the indexed tree, e.g. `Helpers/Generated/`.)
- **Cost:** edits require a regenerate + reload (same as voice commands — acceptable).

---

## Store schema — `INIDATA/key_actions.json`

```jsonc
{
  "ChromeOpenNewTab": {
    "steps": ["c-t"],              // canonical form: an array of DSL steps
    "desc":  "Open a new Chrome tab",
    "group": "Chrome"             // -> generated file KeyActions_Chrome.ahk (namespacing)
  },
  "ChromeNewTabThenClose": {
    "steps": ["c-t", "sleep(50)", "c-w"],
    "desc":  "New tab, then close it",
    "group": "Chrome"
  }
}
```

- **`steps` is canonical**; `"keys": "c-t"` is accepted as sugar for `"steps": ["c-t"]`. Everything is uniformly a step list, so new step-types (function calls, text, params) slot in without a schema change. **[future-proofing #1]**
- **Reserved-for-future fields** (accepted/ignored in V1 so adding them never migrates the store): **[future-proofing #2]**
  - `"params": ["url"]` — a parameterized action; generator emits `Name(url) { ... }`.
  - `"scope"` — default binding scope hint (global / a context token), for the future auto-bind flows.
  - `"tags"` — extra search tags.

---

## The step DSL (Caster-syntax) — one grammar, extensible classifier

Each step is a string. A **single classifier** (in the generator; optionally a runtime twin) routes it by shape → emits AHK. Keep this dispatch a clean table so adding a step-type is additive. **[future-proofing #3]**

**[V1 BUILT] step types** (classifier checked in this order):
- **Sleep** — `sleep(N)` → `Sleep N`.
- **Text** — `text(hello world)` → `SendText("hello world")` (AHK-escaped).
- **Function call** — `FunctionName(arg1, "arg2")` → emitted **verbatim** as a native call. Generator validates the name exists (hand-written function index OR a sibling key-action); an **unknown** call skips the WHOLE action (never emit a call to a nonexistent function — that would be a load-time error breaking all of MAINFUNCTIONS). This is the "include real AHK functions" ask — free because output is real AHK.
- **Key combo** (default) — `<mods>-<key>`; mods ⊆ `{c=Ctrl, a=Alt, s=Shift, w=Win}`, e.g. `c-t`, `cas-t`, `w-e`. Bare key allowed (`enter`, `f5`, `t`).
  - Keys: letters/digits verbatim; **named specials** map to AHK: `enter`→`{Enter}`, `tab`, `escape`/`esc`, `space`, `backspace`, `del`/`delete`, `home`, `end`, `pgup`/`pgdown`, `up`/`down`/`left`/`right`, `f1`..`f24`, `insert`, punctuation names (`minus`, `equals`, …).
  - Emit: `KeyActionSend("cas-t")` (runtime parses + `SendInput`). Mods → `^!+#`.

**[FUTURE] step types** (design already supports — just add classifier branches):
- **Repeat** — Caster-style `t:3` (press t ×3), `c-t:2`.
- **Key down/up** — `shift:down` / `shift:up` for chords.
- **Conditionals / context guards** — later; would move compilation toward a small emitted block.

**Send reliability** (the thing that was broken): `SendInput` for all Ctrl/Alt/Shift combos (fast, atomic). **Win-key combos are the one shaky case** (`Send("#…")` silently drops some, e.g. snapping) — if a `w-` action misbehaves, route it through the existing Playback/DisplayFusion path the snap functions use. Centralized in `KeyActionSend`, so the fix lives in one place.

---

## Generator — `Scripts/gen_key_actions.py`

1. Read `key_actions.json`.
2. **Collision guard:** refuse/warn if an action name already exists as a real function in the index (`INIDATA/fn_to_file_index.json`) — never shadow a hand-written function. **[future-proofing #4]**
3. Group actions by `group`; emit one file per group: `Helpers/Generated/KeyActions_<Group>.ahk`, each action a PascalCase function with inlined step emissions (readable/greppable — the generated file shows the real keys).
4. Ensure MAINFUNCTIONS (and the always-on closure, if these must run in the persistent process) `#Include` the generated files — one glob-style include block for `Helpers/Generated/`.
5. Bump the AHK reload (`ReloadWithNotice` path) so new/changed functions go live; regenerate the function index.

Example emitted:
```ahk
; AUTO-GENERATED from INIDATA/key_actions.json — do not edit by hand.
ChromeOpenNewTab() {
    KeyActionSend("c-t")
}
ChromeNewTabThenClose() {
    KeyActionSend("c-t")
    Sleep 50
    KeyActionSend("c-w")
}
```

---

## Runtime — `Helpers/KeyActionFunctions.ahk` (hand-written, stable)

- `KeyActionSend(casterStr)` — parse `<mods>-<key>` (+ named-key table) → `SendInput` with the right modifier symbols; Win-combo fallback path. This is the ONE place send-reliability is solved.
- (Later) `KeyActionText`, repeat/down-up helpers as step-types land.

---

## Integration

- **Stream Deck button:** `MAINFUN.bat ChromeOpenNewTab` (the default when binding a button — "make a new named function"), OR bind a button to a raw combo directly (`send:c-t`) for one-offs.
- **Voice:** a rule maps a phrase → `mainfun_action("…", function_name="ChromeOpenNewTab")` — same function the button calls.
- **Bindings** (Q0 / pedals / CapsLock / keyboard / contexts): bind to the function name; globally or per-context.
- **Reconcile the old `send:` actions:** the existing `bindings.json` `send:<combo>` (from the broken capture flow) either migrate to named key-actions, or `send:` learns to resolve a key-action name. The macro/binding system (`references/macro-binding-system.md`) is the integration seam. **Migrating the capture UX to "create/pick a key-action" is what fixes the broken flow.**
- **Change-once-changes-everywhere:** because every caller references the function NAME, editing the JSON + regenerating updates all callers at once.

---

## Future-proofing baked in now (cheap, prevents migrations)

1. `steps` array is canonical (`keys` is sugar) → new step-types are additive.
2. Reserved `params` / `scope` / `tags` fields accepted-and-ignored in V1.
3. Step classifier is a clean dispatch table → function-call/text/repeat steps are new branches, not a rewrite.
4. Name-collision guard vs the real function index.
5. Output is real AHK in the indexed tree → searchable/loggable/bindable with zero extra plumbing, and native function-call steps.

**Does the "embed real functions later" goal change V1?** No — only the five items above, all cheap. Nothing structural.

---

## Future features (captured)

- **[FUTURE] Parameterized key-actions** — `params` + step interpolation (`text(${url})`); generator emits `Name(url)`.
- ~~**[FUTURE] Function-call steps**~~ **DONE 2026-06-30** — call any existing AHK function (or sibling key-action) as a step, with the existence guard. The headline "include other real functions" — shipped.
- **[FUTURE] Editor function-picker-insert** — press a hotkey in the editor → a `show fun`-style picker (reads the function index + `.meta.json` args) → pick a function + fill its arguments → **insert the call directly** (or type the name if known). Works for both hand-written and generated functions since both are indexed with arg metadata. This is a natural payoff of everything being a real, indexed function.
- **[FUTURE] GUI authoring** — add/edit a key-action from the macro wizard or a form (name + Caster steps + group), with a reliable step-capture (hand-type-with-live-preview beats keydown-capture, which fights Dragon/AHK hooks).

---

## Build order

1. ~~**[V1] Runtime** `KeyActionFunctions.ahk` (`KeyActionSend` + Caster parse + send reliability).~~ **DONE 2026-06-30.**
2. ~~**[V1] Store** `key_actions.json` + **generator** `gen_key_actions.py` (grouping, collision guard, include wiring).~~ **DONE.**
3. ~~**[V1] Verify end-to-end.**~~ **DONE** — dispatched a sleep-only `SmokeKeyAction` via `MAINFUN.bat` → `OK`; parser verified for c-t/cas-t/w-e/enter/f5/c-minus/as-right/s-tab/bare-letter.
4. ~~**[V1] `key-action add/list` CLI**~~ **DONE** — `add` / `list` / `remove` on `gen_key_actions.py`.
5. **[SOON] Bindings reconcile** — SD-button "bind to new key-action" as the default path; migrate/adapt `send:`.
6. ~~**[FUTURE] function-call + text steps**~~ **DONE 2026-06-30.** Remaining [FUTURE]: param steps, repeat (`t:3`), key down/up, GUI authoring, editor picker-insert.

---

## V1 — as built (2026-06-30)

- **Store:** `INIDATA/key_actions.json` — `{ Name: {steps[], desc, group} }`. `keys` sugar + reserved `params`/`scope`/`tags` accepted.
- **Generator:** `Scripts/gen_key_actions.py` — `py Scripts/gen_key_actions.py` regenerates; `list`; `add <Name> "<step> / <step> ..." [--desc][--group]`; `remove <Name>`. Groups by `group` → `Helpers/Generated/KeyActions_<Group>.ahk`; writes `Helpers/Generated/_KeyActionsAll.ahk` aggregator; **collision guard** scans `Helpers/**/*.ahk.meta.json` keys (excl. `Generated/`) and refuses shadowing a hand-written function; deletes stale `KeyActions_*.ahk` each run so renamed/emptied groups don't linger.
- **Runtime:** `Helpers/KeyActionFunctions.ahk` — `KeyActionSend` (Caster parse → `SendInput`; mods `c/a/s/w`→`^!+#`; named-key table; f-keys; win-combo flagged for the DisplayFusion fallback if a `w-` action ever misbehaves). Sends ONE combo per call; the generator unrolls multi-step actions into `KeyActionSend(...)` / `SendText(...)` / `Sleep N` / verbatim-function-call lines.
- **Step types built:** `sleep(N)`, `text(...)`→`SendText`, `Name(args)`→verbatim real-AHK call (existence-guarded against the function index + sibling key-actions; unknown call skips the whole action), and the default Caster key-combo. Classifier is a checked-in-order dispatch in `classify_step()` — new types are additive branches.
- **Wiring:** MAINFUNCTIONS.ahk `#Include`s the runtime + `#Include *i` the aggregator (optional-include so a missing file never breaks load). `%A_ScriptDir%` paths are stable inside included files. SD/voice/Claude calls (fresh MAINFUN.bat process) pick up new functions with no reload; the persistent always-on process (bindings path, [SOON]) would need its own include when reconcile lands.
- **Gotchas hit:** AHK v2 `FileDelete` throws when the file is absent (guard with `try`); a function referenced but undefined is a load-time error (so a stubbed `LogEvent` was needed to test the runtime in isolation).
