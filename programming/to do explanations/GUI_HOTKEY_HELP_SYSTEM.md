---
tags: [gui, hotkeys, help, autohotkey, miller, data-driven, design]
created: 2026-06-18
status: built
owner: Jamie
---

# GUI Hotkey Help + Rebind System — the `?` overlay

Press **`?`** in any AutoHotkey GUI → a panel that SHOWS and EDITS every hotkey
active in that GUI, **generated from the live bindings** so it is never wrong and
never hand-maintained. Data-centric: keys are **rebindable through the panel**
(Browser), from day one.

## UPDATE (2026-06-18, same day) — now UNIVERSAL across all templates, 66/66 green

The `?` overlay + standardized hotkeys are no longer Miller-only. Built on top of
the as-built below:
- **Universal action catalog** `_GuiUniversalActionCatalog` (GuiPrimitives):
  `gui.cancel` [End, Numpad0], `gui.accept` [Numpad1, Home], `gui.help` [?].
  Home is an accept key by default; templates with a native Home meaning exclude
  just that key. Single source of truth for cross-GUI basics.
- **Cascade**: user override > instance `_GuiBindAction` > template defaults >
  universal catalog, all keyed by hierarchical actionId. Rebinding `gui.cancel`
  changes cancel everywhere; `miller.sort.cycle` changes only that.
- **`_GuiStandardActions(g, handlers, opts?)`** — new high-level entry: bind the
  universal set + `?` in one call (callbacks only, never keys). **`_GuiBindHelp`**
  = just `?`. **`_GuiAttachStandardHotkeys` rewritten** onto this, so it now
  registry-records its keys AND auto-adds `?` (Confirm/Form/SingleField/ThumbGrid
  inherited the overlay with zero changes).
- **Overlay is a decoupled plugin**: `GuiHelpOverlay.ahk` registers its opener via
  `_GuiHelpOpener(fn)` at load; GuiPrimitives never names it. GuiPrimitives
  **self-includes** the overlay, so `?` works in EVERY GUI process automatically —
  a brand-new GUI inherits it with no include to remember.
- **`_GuiBindAction` is idempotent per (hwnd,actionId) + self-prunes dead windows**
  — universal + explicit binds compose safely; the registry never leaks.
- **Rolled out**: Reader / Picker (PersistentLoop) / Browser register their keys
  (display_only where the binding has passthrough/digit nuance) + inherit `?`.
  Browser passes `no_help` for the overlay's own window (no recursion).
- **Tests**: `?`-opens verified on Miller + Confirm + Reader + Browser; full suite
  66/66.

New GUI now = `_GuiStandardActions(g, Map("on_cancel", closeFn))` → cancel + `?` +
overlay, done. New instance of a template = zero hotkey code. Full how-to:
`AutoHotkey/docs/gui-conventions.md` (the cascade section).

## AS BUILT (2026-06-18) — phases 1-6 complete, 63/63 GUI suite green

Files (all shipped):
- **`Scripts/gui_hotkeys/hotkeys.py`** — override store (`{actionId: [keys]}`),
  atomic JSON at `INIDATA/Hotkeys/gui_hotkeys.json`. CLI: `list / get / resolve /
  set / add-key / remove-key / clear / selftest`. Pure `normalize_keys` +
  `find_conflicts`. `selftest` green.
- **`Helpers/Gui/GuiHotkeys.ahk`** — thin shim (mirrors GuiLayout.ahk). Key change
  from the design: the cache is **primed in ONE `list` call** (`GuiHotkeyPrime`)
  rather than one `get` per action — binding ~16 actions was otherwise ~16
  subprocesses (seconds of startup + a keypress-before-bind race). The Miller
  primes *before* showing so every post-show `_GuiBindAction` is an in-memory hit.
- **`Helpers/Gui/GuiPrimitives.ahk`** — `_GuiBindAction(hwnd, actionId, label,
  defaultKeys, scope, category, fn, opts?)` + per-window action registry +
  `_GuiActionAddKey / RemoveKey / SetKeys(…, persist?) / KeyOwner / UnregisterAll`
  + `_GuiCaptureHotkey` (live "press the combo" capture via InputHook). `opts`:
  `locked` (structural, read-only) and `display_only` (record for help WITHOUT
  binding — used for the digit buffer). Self-includes GuiHotkeys.
- **`Helpers/Gui/GuiHelpOverlay.ahk`** — `_GuiHelpOverlay(parentHwnd)`: a
  `_BrowserGui`, ONE ROW PER ACTION (all its keys, pretty-printed `^s`→`Ctrl+s`),
  GROUPED BY CATEGORY (Navigation/Sort/Layout-Edit/View/Help), scope as a "this
  page / this GUI" tag, `(fixed)` on locked rows, `*` on overridden ones. Row
  actions `2=Add key 3=Remove 4=Replace 5=Revert`. Self-includes BrowserGui.
- **`Helpers/Gui/MillerColumnPickGui.ahk`** — every binding migrated to
  `_GuiBindAction` with action-ids (`miller.*`), labels, scope, category, locked
  flags; `?` bound to `gui.help.overlay`. Self-includes GuiHelpOverlay.
- **Tests** — `hotkeys.py selftest` + `Helpers/Tests/Gui/test_hotkeys.ahk` (pure
  normalize/pretty/group/order/row-cells + a live bind/add-alias/remove/conflict/
  revert round-trip + an end-to-end "`?` opens the overlay" fixture test).
- Explicit includes added to `AlwaysOn/_CuratedClosure.ahk` (Footpedals/Q0Max
  hosts) so the include-drift checker sees GuiHotkeys + GuiHelpOverlay.

Gotcha banked: the include-drift checker's def-parser is **not string-aware** —
a `()` inside a default-param string literal makes it read the def as a call.
Keep parens out of default-arg strings (cost me the `_GuiCaptureHotkey` default).

Everything below is the original design; it matches what shipped except the
one-call priming optimization noted above.

---

## Terminology (Jamie 2026-06-18 — this is the whole point)
- **Global** = global *within this GUI* — works on every page/level of it (e.g.
  across the whole Spotify library). **NOT** system-wide / OS hotkeys.
- **Local** = specific to the *current page/level* (e.g. this level's `N.M` row
  actions).
- System-wide / macro-binding-system hotkeys (`bindings.json`, modes) are **out of
  scope for v1** — maybe a third "System" group later, not important now.

## Core challenge
AHK v2 has **no API to enumerate active hotkeys**. "Generated, never wrong" can't
mean "ask the OS what's bound" — it means **capture each binding at registration
time** into a per-window registry the panel reads. The codebase already funnels
GUI hotkeys through `_GuiRegisterHotkey` / `_GuiAttachStandardHotkeys`
(`Helpers/Gui/GuiPrimitives.ahk`), so this is a natural extension of that layer.

Related (mirror these patterns): the layout system — [[MILLER_LAYOUT_SYSTEM]]
(`Scripts/gui_layouts/layouts.py` + `Helpers/Gui/GuiLayout.ahk` + the engine's
data-centric `sorts`/`order` stores). Hotkeys is the same shape: code = defaults,
data = overrides, UI = editor. GUI conventions: `AutoHotkey/docs/gui-conventions.md`.

---

## Architecture

### 1. Data-centric bindings (the key idea)
Today: `_GuiRegisterHotkey(hwnd, "^s", (*) => _cycleSectionSort())` — the KEY is
hardcoded next to the ACTION. Separate the stable action from its **keys**:

```ahk
_GuiBindAction(hwnd, actionId, label, defaultKeys, scope, category, fn, opts?)
```
- `defaultKeys` is a **LIST** (a bare string is accepted as a 1-element list)
- resolves `keys = override-store(actionId)` else `defaultKeys`
- registers a hotkey for **each** key → the SAME fn (so an action can have many
  keys)
- records `{actionId, label, keys, scope, category, fn, handles, locked}` into a
  **per-window registry** keyed by hwnd.

The **action (closure) stays in code** (closures can't be serialized); only the
**key list** is data. Exactly the layout pattern: code declares the catalog of
action-ids + default keys; the JSON store overrides them; the `?` panel edits the
store.

**Many keys → one action is first-class** (Jamie 2026-06-18). `Esc` = cancel, and
if you later want `End` = cancel too, that's just appending `"End"` to `cancel`'s
key list — one store write + one live-register, no code edit. The reverse
(many-to-one) is what the panel surfaces: every action shows ALL its keys.

`scope` = `"global"` (whole-GUI) | `"local"` (current page/level).
`category` = a grouping label ("Navigation", "Sort", "Layout/Edit", "View",
"Help", …) so the panel can group **types together** (see §3).
`opts.locked = true` for structural keys that must not be rebindable.

**Why fully data-centric is advisable (the caveats, none are blockers):**
- Closures stay in code — we key by `actionId`, store only the key string.
- **Locked structural keys**: Esc, arrows, Enter/NumpadEnter, Pg/Home/End, digits
  0-9, `.`, Tab, `?` itself → shown but NOT editable (rebinding them could
  soft-brick the GUI — e.g. lose your cancel key). Discretionary Ctrl-chords are
  rebindable.
- **Conflict detection**: reject a key already bound in the same scope (+ message).
- **Stable action-ids**: a one-time labeling pass (like the layout catalog keys).

### 2. Override store
`Scripts/gui_hotkeys/hotkeys.py` + `INIDATA/Hotkeys/gui_hotkeys.json` (mirrors
`layouts.py`): `{ "<actionId>": ["<key>", ...] }` — each override is a **key
list** (a stored bare string is tolerated as a 1-element list). CLI = the edit
API (note `add-key`/`remove-key` make aliasing trivial):
`list / get <actionId> / set <actionId> --keys k1,k2 / add-key <actionId> --key K /
remove-key <actionId> --key K / clear <actionId> / selftest`. Thin AHK shim
`Helpers/Gui/GuiHotkeys.ahk`: `GuiHotkeyKeys(actionId)` (cached get → list),
`GuiHotkeyAddKey(actionId, key)`, `GuiHotkeyRemoveKey(actionId, key)`,
`GuiHotkeySet(actionId, keysArray)`, `GuiHotkeyClear(actionId)`.

Action-ids are **engine-level** (`miller.sort.cycle`, `miller.rearrange`, …), so a
rebind applies to ALL Millers — almost certainly desired ("Ctrl+S sorts" should be
uniform). Per-GUI scoping is possible later via more specific ids.

### 3. The `?` panel (Browser, editable day one)
- `?` bound in every GUI (universal, via GuiPrimitives). Jamie never types `?` into
  a filter, so capture it directly — no empty-filter guard. **No F1.**
- Opens a **Browser** (`_BrowserGui`), one row **per ACTION** (not per key), so the
  many-keys-to-one-action relationship is visible at a glance:
  `Keys | Action | scope` — e.g. `Esc, End | Cancel | global`. Seeing everything a
  thing does, and everything that does it, is the default view.
- **Grouped by `category`** (Navigation, Sort, Layout/Edit, View, Help, …) so types
  cluster together (Jamie 2026-06-18). `scope` (this-page vs whole-GUI) shows as a
  per-row tag/column rather than the primary split; the current level's `row_actions`
  (`N.2`/`N.3`…) + leaf are pulled LIVE and fall under a "This page" category.
- Row actions on a selected action:
  - **Add key** → prompt → validate (not locked, no conflict) → append to the
    action's key list → **live-register** → refresh. (This is the "Esc AND End =
    cancel" path — trivial.)
  - **Remove key** → drop one key from the action (can't remove the last key of a
    locked/essential action).
  - **Replace** → set the key list wholesale.
  Locked actions: add/remove disabled (shown, read-only).
- 100% generated from the registry + current level → never wrong, never written.

### 4. Introspection sources (what feeds the panel)
1. **Per-window action registry** (from `_GuiBindAction`) → whole-GUI keyed
   hotkeys + nav. Rebindable (unless `locked`).
2. **Current Miller level's `row_actions` Map + leaf** → this-page actions. These
   are positional (`N.<digit>`), so SHOWN but not key-rebindable (they're digit
   slots, not keys). The engine already has these in `state` + the level/node.

### 5. Live add / remove / rebind
The registry holds each action's `handles` (one per bound key) + `fn`. **Add key**
= register `newKey → fn`, push the handle, append the key, persist. **Remove key**
= unregister that key's handle, drop it. **Replace** = remove-all + add-all.
Applies immediately, no reopen. (Fallback: if live (de)register proves fiddly,
persist + toast "reopen to apply" — but live is the goal.)

---

## Scope / build phases
1. **`hotkeys.py` + `gui_hotkeys.json` + `GuiHotkeys.ahk`** (resolve + set + clear
   + selftest). Pure resolve is unit-tested.
2. **`_GuiBindAction` in GuiPrimitives** (resolver + per-window registry; wraps
   `_GuiRegisterHotkey`). Add `_GuiHotkeyRegistryFor(hwnd)` reader + a live-rebind
   helper. Mark locked keys.
3. **Migrate the Miller engine** (`MillerColumnPickGui.ahk`) — its `extraHandles`
   block + standard hotkeys become `_GuiBindAction` calls with action-ids, labels,
   scope, locked flags. (Primary target — Spotify library, projects, layouts
   browser all ride the Miller.)
4. **The `?` Browser overlay** — one row per action, **grouped by category**, each
   showing ALL its keys; this-page actions from the live level, whole-GUI from the
   registry; Add-key / Remove-key / Replace row actions (validate + live
   register/unregister + persist). Bind `?` universally in GuiPrimitives.
5. **Tests** — hotkeys.py selftest (override-beats-default, **multi-key lists**,
   add-key/remove-key, conflict, locked); fixture test that `?` opens the Browser,
   lists the expected keys grouped by category, and an **add-an-alias round-trip**
   (Esc+End both fire cancel).
6. **Other templates** (Picker/Reader/Form/Confirm/Browser) get `?` (mostly nav +
   their few actions). Smaller, after the Miller proves it.
7. **(Later, maybe) system-wide group** — enumerate the macro/binding system
   (`bindings.json`, current mode) as a third "System" section. Deferred.

## Locked vs rebindable (v1 Miller)
- **Locked** (shown, not editable): `Esc`, `Up/Down/Left/Right`, `Enter`/
  `NumpadEnter`, `PgUp`/`PgDn`/`Home`/`End`, `0`-`9` + `Numpad0-9`, `.`/`NumpadDot`,
  `Tab`, `?`.
- **Rebindable**: `Ctrl+S` (sort field), `Ctrl+E` (sort direction), `Ctrl+R`
  (rearrange), `Ctrl+D` (dark mode), `Del` (hide), `NumpadSub` (divider), + any
  future action keys.

## Open questions (decide during build)
- Separate `hotkeys.py` store vs folding into a shared "gui config" store with
  layouts. **Lean separate** (hotkeys ≠ layouts; cleaner).
- Conflict policy: reject-with-message vs swap-keys. **Lean reject.**
- Rebind scope: engine-level action-ids (uniform across all Millers) vs per-GUI
  overrides. **Lean engine-level**, per-GUI later if wanted.
- Live re-register vs reopen-to-apply. **Lean live.**
- Whether `?` should also work on non-template one-off GUIs (none remain per the
  2026-05-21 refactor, so universal-via-GuiPrimitives covers everything).

## Why this is the right shape
- **Never wrong / never maintained**: generated from the same registrations that
  actually bind the keys — the list IS the source of truth.
- **Consistent**: same data-driven store+resolver+editor pattern as the layout
  system; the `?` Browser is one more dogfood of the Browser template.
- **Editable day one**: the data-centric split makes rebinding a key a one-line
  store write + live re-register, not a code edit.
