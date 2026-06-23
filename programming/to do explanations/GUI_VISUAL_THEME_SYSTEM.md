---
tags: [gui, theme, dark-mode, visual, autohotkey, miller, browser, data-driven, design]
created: 2026-06-18
status: design-locked
owner: Jamie
---

# GUI Visual / Theme System — shared look, inherited by every template

The visual sibling of the hotkey system ([[GUI_HOTKEY_HELP_SYSTEM]]). Today the
Miller looks polished (dark mode, zebra rows, clean palette) and **every other
template looks old** because that styling lives *inside the Miller* and nowhere
else. This makes a shared theme layer — store + apply primitive + universal
toggle + cascade — so dark mode and a consistent look are **inherited by all (or
select) templates**, and changing the look is one edit, not N.

Motivating case: the **`?` hotkey-editor overlay is a `_BrowserGui`**, and the
Browser template is a plain light Windows ListView — that's the "ugly / old"
surface Jamie wants to look like the Miller (single clean column, dark mode).
Reskinning the Browser → the overlay (and StackViewer, KindleBookManager, the
recent-functions viewer, …) all get the Miller look at once.

## The parallel to hotkeys (same architecture, one layer over)

| Hotkeys (built) | Theming (this doc) |
|---|---|
| `hotkeys.py` + `gui_hotkeys.json` store | `theme.py` + `gui_theme.json` store (mode, accent, font, palette overrides) |
| `GuiHotkeys.ahk` shim (primed once pre-show) | `GuiTheme.ahk` shim (primed once pre-show) |
| `_GuiBindAction` / `_GuiStandardActions` | `_GuiApplyTheme(g, opts)` — every template calls it |
| `gui.theme.toggle` (`^d`) is already a universal action | flips the store + re-applies → **dark mode free on every template** |
| cascade: user > instance > template > universal | same cascade for colors/fonts |
| overlay is a decoupled plugin | theme is pure data; no plugin needed |

The Miller's existing theming is the **reference implementation we extract
upward** — the Miller's look does not change; the others rise to meet it.

---

## What exists today (the Miller, to be generalized)

All in `Helpers/Gui/MillerColumnPickGui.ahk`:
- `_MillerThemePath()` → `INIDATA/miller_theme.txt` holding `"dark"`/`"light"`.
- `_MillerThemeIsDark()` / `_MillerThemeSetDark(dark)` — the store.
- `_MillerThemeColors(dark)` → palette Map: `bg, lvbg, text, row1, row2, sel,
  seltext, divider` (0xRRGGBB).
- `_applyTheme(dark)` — sets `g.BackColor`, control font colors, `+Background`
  on Edits/ListViews, strips the Explorer visual style
  (`uxtheme\SetWindowTheme … ""`) so custom-draw colors win, registers each LV in
  `_McpCDRegistry`, redraws.
- `_McpOnNotify` (WM_NOTIFY/NM_CUSTOMDRAW) — the **zebra-stripe engine**: only
  ListViews in `_McpCDRegistry` are painted; everything else draws normally.
- `_McpHex` / `_McpRgbToBgr` / `_McpThemeBgr` — color utils.
- `^d` → `miller.theme.toggle` (already a universal-style action id) → `_toggleTheme`.

This is ~clean and self-contained — extraction is mostly *moving + renaming*, not
rewriting.

---

## Architecture

### 1. Theme store — `theme.py` + `INIDATA/Theme/gui_theme.json` — BUILT (phase 1)
A **registry of named, switchable themes** (not a dark/light flag — Jamie wants
to grow + switch between multiple themes; 2026-06-18). Dark + Light are seeded
built-ins; the user adds more and cycles/picks. Mirrors `hotkeys.py` (atomic,
BOM-less, CLI = edit API, selftest green). Schema:
```json
{
  "version": 1,
  "current": "dark",
  "themes": {
    "dark":  {"label":"Dark","kind":"dark","font_name":"Segoe UI","font_size":10,
              "palette":{"bg":"0x1E1E1E", ...8 tokens...}},
    "light": { ... }
  },
  "template_overrides": { "browser": {"font_size":11} }
}
```
Tokens: `bg lvbg text row1 row2 sel seltext divider`. `kind` (dark/light) hints
dark-title-bar / Explorer-style. The Miller's two palettes are the seeded
defaults; **migrates from `miller_theme.txt`** on first read.
CLI: `list / get-current [--template T] / get <id> / set-current <id> /
next / prev / set-token / set-font / add <id> --from EXISTING / delete /
template-set / selftest`. `next`/`prev` cycle `current` (drive the `^d` action);
a future theme picker uses `list`. **Shim `GuiTheme.ahk` built** (primed once,
cached, palette hex → ints): `GuiThemeCurrent/Colors/Font/IsDark/List/SetCurrent/
Next/Prev`. Wired into GuiPrimitives' self-includes.

### 2. AHK shim — `Helpers/Gui/GuiTheme.ahk`
Mirrors `GuiHotkeys.ahk`: primed once per process (one `get` call), cached.
- `GuiThemeIsDark()` / `GuiThemeMode()`
- `GuiThemeColors(mode := "")` → palette Map (base + overrides merged)
- `GuiThemeFont()` → `{name, size}`
- `GuiThemeToggleMode()` → flips + persists, returns new mode
- `GuiThemePrime(force := false)` — called from `_GuiShowTestAware` (same hook
  that primes hotkeys), so it's one Python call pre-show, then in-memory.

### 3. The apply primitive — `_GuiApplyTheme(g, opts)` in GuiPrimitives
The visual analog of `_GuiStandardActions`. Generalizes Miller `_applyTheme`:
- reads `GuiThemeColors()` + `GuiThemeFont()`,
- sets `g.BackColor`, walks **all** the Gui's controls (or an opt-in list) and
  applies text color + `+Background` per control type (Text/Edit/ListView), and
  routes **Buttons** through the OS dark visual style
  (`SetWindowTheme(hwnd, "DarkMode_Explorer")` in dark, `"Explorer"` in light) —
  see the dark-button note under Status #6,
- strips Explorer style + registers each ListView into the shared custom-draw
  registry for zebra + themed selection,
- optional **dark title bar** via `DwmSetWindowAttribute(DWMWA_USE_IMMERSIVE_
  DARK_MODE)` (the Miller doesn't do this yet — a free win for all),
- `opts.controls` to theme only specific controls; `opts.zebra_lvs` to pick which
  ListViews get the stripe; `opts.template` to pull `template_overrides`.

Folded into `_GuiApplyStandardSetup` (called pre-show by ~every template) OR
called explicitly post-control-creation (theming needs the controls to exist, so
likely a `_GuiApplyTheme(g)` call right before `_GuiShowTestAware`).

### 4. The shared custom-draw zebra engine (move out of the Miller)
`_McpOnNotify` + `_McpCDRegistry` + `_McpThemeBgr` become
`_GuiThemeOnNotify` + `_GuiThemeCDRegistry` (GuiPrimitives), registered **once**
process-wide (`OnMessage(0x4E, …)`). Any template adds its ListView via
`_GuiThemeRegisterListView(lv, cols)` and gets Miller-grade zebra + selection
colors. The Miller switches to the shared registry (no behavior change).

### 5. Universal dark-mode toggle — `gui.theme.toggle`
Promote the Miller's `^d` to a **universal action** in the catalog
(`_GuiUniversalActionCatalog`), like `gui.help`. `_GuiStandardActions` /
`_GuiAttachStandardHotkeys` bind it (flips `GuiThemeToggleMode()` + re-applies to
the open window). Result: **every template gets `^d` dark-mode toggle for free**,
and toggling persists machine-wide so the next GUI opens in the chosen mode.
(Needs a re-apply hook: `_GuiApplyTheme(g)` is idempotent, so the toggle handler
just calls it again + `WinRedraw`.)

### 6. The cascade (same as hotkeys)
`user override (store) > instance opts > template defaults > base palette`.
Keyed by token (`sel`, `font_size`, …) + optional per-`template` overrides. So:
- changing the global look = edit the store once → every template;
- a template that wants its own accent = one `template_overrides` entry or
  `_GuiApplyTheme(g, Map("accent", …))`;
- the Miller's exact palette stays as the base default.

---

## Browser reskin (the immediate payoff)
`BrowserGui.ahk` calls `_GuiApplyTheme(g, Map("zebra_lvs", [itemsLV]))` after
building its ListView + Edit, and registers `itemsLV` for custom-draw. Result:
the Browser — and therefore the **`?` overlay**, StackViewer, KindleBookManager,
recent-functions viewer — render dark, zebra-striped, Miller-fonted: "a single
clean column like the Miller." No per-caller work; every Browser inherits it.

Optionally tighten the overlay's columns/spacing to read as one clean list
(Keys / Action / Where) rather than a spreadsheet.

---

## Companion: hotkey model refinement (small, do alongside)
Jamie's point — "bind globally always-on, locals override automatically" — is the
cleaner model. Change `_GuiBindAction` to **auto-relinquish**: when an action
binds a key already held by another action in the same window, remove that key
from the prior action (transfer it). Then:
- bind the universal `gui.cancel/accept/help/theme.toggle` always-on,
- a template that wants a universal key for its own action just binds it — the
  universal action silently yields that key,
- **drop the per-template `exclude` lists** (only native-control keys —
  ListView/Edit built-in nav — still need leaving-alone, since a hotkey always
  intercepts those before the control).
Overlay stays clean because each key ends up on exactly one action.

---

## Build phases
1. ✅ **`theme.py` + `gui_theme.json` + `GuiTheme.ahk`** — BUILT (multi-theme
   registry; selftest green; primed in `_GuiShowTestAware`).
2. ✅ **`_GuiApplyTheme` + shared custom-draw engine** in GuiPrimitives — BUILT.
   Extracted the Miller's `_McpOnNotify`/`_McpCDRegistry`/`_McpThemeBgr` as
   `_GuiThemeOnNotify` / `_GuiThemeCDRegistry` / `_GuiThemeBgr` (+ `_GuiHex` /
   `_GuiRgbToBgr` / `_GuiThemeRegisterListView`). Added `_GuiSetDarkTitleBar`
   (DWM) — a win the Miller didn't have. Lazy process-global WM_NOTIFY hook.
5. ✅ **Browser adopts `_GuiApplyTheme`** — BUILT + visually confirmed: the `?`
   overlay (and StackViewer etc.) render dark + zebra + dark title bar like the
   Miller. `^d` cycles themes in-place (`gui.theme.toggle` universal action).
3. ✅ **Miller deduped onto the shared engine** — its private palette / color utils
   / BGR converter / `_McpOnNotify` / `_McpCDRegistry` deleted; `_applyTheme` now
   reads `GuiThemeColors()` + registers via `_GuiThemeRegisterListView`; `^d` →
   `GuiThemeNext()`. **Visually pixel-identical**, suite 66/66 (its regression guard).
   One engine for every GUI now. (Bonus: the Miller got a dark column header too.)
4. ✅ **`gui.theme.toggle` everywhere** — bound in `_GuiAttachStandardHotkeys`
   (Confirm/Form/SingleField/ThumbGrid) + Reader/Picker/Browser. The re-apply is
   generic: `_GuiApplyTheme` records each window's opts (`_GuiThemeLastOpts`), and
   `_GuiThemeCycleThisWindow(g)` does `GuiThemeNext()` + re-apply with those opts —
   so any window that themed itself cycles correctly with zero extra wiring.
6. ✅ **Rolled out** `_GuiApplyTheme(g)` to Reader / Picker / Confirm / Form /
   SingleField / ThumbnailGrid — every template now OPENS in the current theme
   (dark by default). Picker uses `zebra_all` (both list panes striped like the
   Miller). The custom-draw registry self-prunes dead windows, so no template
   needs an explicit unregister. Reader + Confirm visually verified; suite 66/66.
   - ✅ **Dark buttons (2026-06-19)** — the original claim that "Windows 11
     dark-renders buttons once the window has the immersive attribute" was WRONG:
     Jamie's Finished/Reading dialogs showed bright-white buttons in dark mode.
     Fix: opt the PROCESS into dark mode via the private uxtheme ordinals
     (`_GuiEnableDarkModeForApp` → ordinal 135 `SetPreferredAppMode(AllowDark)` +
     136 `FlushMenuThemes`), called from `_GuiApplyStandardSetup` BEFORE controls
     exist (with a safety-net re-call + per-window ordinal 133
     `AllowDarkModeForWindow` at the top of `_GuiApplyTheme`), then per-button
     `SetWindowTheme("DarkMode_Explorer")`. The earlier attempt was abandoned only
     because it couldn't be visually verified on the asleep test monitor; with
     `ahk.py show` screenshot verification it's confirmed working (dark face, light
     text). The "before controls exist" ordering is what the post-construction
     attempt was missing.
7. ✅ **Hotkey auto-relinquish** — `_GuiBindAction` now steals a key from any other
   action in the window that holds it (`_GuiActionRelinquishKey`), so the universal
   actions bind always-on and a template's own binding overrides automatically.
   The overlay skips zero-key (fully-relinquished) actions. (Native CONTROL keys —
   ListView/Edit built-ins — are the only thing still left alone.)
8. ⏳ **Tests** — `theme.py` selftest done + the full GUI suite (66/66) covers
   behavior. A dedicated "opens dark / `^d` cycles + persists" fixture test is the
   one remaining nicety.

### Polish
- ✅ **No open flash** — theming runs PRE-show now. `g.Hwnd` is valid right after
  `Gui()` (the old "Hwnd is 0 before Show" note was wrong), and controls exist
  before show, so `_GuiShowTestAware` applies the full theme (bg + control colors
  + zebra + dark title bar) generically BEFORE the first paint. The window opens
  dark from frame one — no light-default flash. (MultiField/ThumbGrid theme just
  before their own `g.Show`.) Templates' post-show `_GuiApplyTheme(template:…)`
  calls remain as dark-over-dark refinements for per-template overrides.
- **ListView header** — reverted the `DarkMode_ItemsView` attempt: it darkened the
  header BACKGROUND but left BLACK text (dark-on-dark, illegible), because the app
  isn't registered dark-mode-aware, and the uxtheme `SetPreferredAppMode`/
  `AllowDarkModeForWindow` ordinals didn't reliably flip the text when called
  post-construction (and can't be verified on the asleep test monitor). Current
  state: the **`?` overlay HIDES its header** (LVS_NOCOLUMNHEADER `0x4000` via the
  new `hide_header` Browser opt — clean dark list; columns Keys·Action·Where are
  in the header text + self-evident). Other Browsers keep the default LIGHT header
  (legible). A fully-dark legible header would need **header NM_CUSTOMDRAW via a
  ListView subclass** — deferred (real complexity, blind-to-verify here).
- Minor: the thin scrollbar gutter corner is still light (cosmetic).

## Gotchas (learned from the Miller)
- **Strip the Explorer visual style** (`uxtheme\SetWindowTheme(lv, "", "")`) or
  the OS repaints its own selection highlight over the custom-draw colors.
- Custom-draw colors are **BGR**, not RGB (`_McpRgbToBgr`).
- Theming needs controls to **exist** → apply after `AddX`, before/at show.
- The WM_NOTIFY handler is **process-global**; gate strictly on the registry so a
  non-themed ListView elsewhere isn't repainted (the Miller already does this).
- Dark title bar (`DWMWA_USE_IMMERSIVE_DARK_MODE = 20`) needs a `WinRedraw` /
  re-show to take on some Windows builds.

## Why this is the right shape
- Same data-driven store+resolver+apply pattern as hotkeys + layouts — one more
  dogfood of a proven architecture.
- The Miller becomes the reference, not a special case; "templates too
  independent" is fixed structurally — look lives in ONE place.
- Inherited by default, overridable per-template, editable via the store (and a
  future `?`-style theme editor if wanted).
