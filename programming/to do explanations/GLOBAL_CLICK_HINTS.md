---
tags: [todo, ahk, uia, caster, voice, accessibility, hints, design-doc, overlay, numpad]
related: ["[[GRAB_TEXT_ENGINE]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[VOICE_COMMAND_SYSTEM]]", "[[ANNAS_ARCHIVE_PIPELINE]]"]
status: BUILT + in use 2026-07-27. Jamie confirms it works well (Signal "perfectly"). v1.1 adds prefix-dimming, Backspace-quit, and the Plex box fix (unconfirmed on Plex)
created: 2026-07-27
---

# Global Click Hints — "make click"

Say **"make click"** (or hit a hotkey). Every clickable thing on screen gets a number
badge. Type the number — no Enter — and it clicks, then re-scans and re-labels. Loops
until Escape. Works in every application, not just the browser.

This is [[GRAB_TEXT_ENGINE]] with the unit source swapped from OCR words to UI
Automation elements, and the overlay unbound from a single window.

---

## Why build it rather than install something

Nothing off the shelf does all four of *global + element-accurate + numbered + loops*.

| Tool | Verdict |
|---|---|
| **Windows 11 Voice Access "show numbers"** | Literally this feature, and genuinely global. But voice-only, no scripting API of any kind, overlay dismisses after each action, and it would sit alongside Dragon competing for the same utterances. |
| **Fluent Search** (Screen Search, Ctrl+M) | Strongest shipped product. UIA engine *plus* a computer-vision fallback engine, whole-screen or focused-window. But letters not numbers, closed source, loop mode unconfirmed, takes no voice command. |
| **hunt-and-peck** | The canonical open-source prior art (C#). Letters, active window only, Invoke-pattern elements only. Its own README calls it sporadically maintained. |
| **win-vind** EasyClick | Actively maintained, scans Win32 + UWP, has an async cache claimed 30× faster. Letters, vim-modal, heavier than we want. |
| **Talon + Rango** | Excellent, browser-only. Talon's Windows accessibility API is experimental and is a scripting surface, not a hint overlay. |
| **warpd** | Not Windows, and its hints are a blind grid rather than element-derived. |

**What changed recently and makes this viable now:** Chrome 138+ ships **native UI
Automation enabled by default on Windows**, replacing the lossy, slow MSAA→UIA emulation
bridge. Electron apps inherit it as they rebase. UIA on Chromium now returns a rich,
fast tree.

### Vimium stays

Not being replaced. The two systems are independently usable and overlap in Chrome by
design.

⚠️ **A correction worth preserving, because it was a reasoning error, not a typo.** An
earlier draft argued the global system was needed in Chrome because a Google Voice dump
had 981 UIA elements but "exactly 6 of type `link`", concluding that was why Vimium finds
so little there. **That conclusion is wrong and the premise doesn't support it.** Vimium
hints the DOM — `a`, `button`, `[role=button]`, `[onclick]`, `[tabindex]` and friends —
so a UIA *control-type* histogram says nothing about its coverage. Jamie confirmed Vimium
in fact works well in Google Voice. Never infer one tool's coverage from another tool's
tree.

What the global system genuinely adds in Chrome is **browser chrome** — tab strip,
toolbar, extension icons, menus — which a page-scoped extension can never reach, and
**every non-browser application**.

**Planned diagnostic:** when a page hints poorly here, Jamie will send a dump captured
with Vimium's hints visible, so we can diff what Vimium labelled against what our filter
kept and find the actual cause. That comparison — not a control-type histogram — is the
right instrument.

---

## Locked decisions

- **Labels are prefix-free, minimum two digits, and never need Enter.** Typing completes
  the moment the label is unambiguous. If `13` exists then `1` cannot be a label.
- **Leading digit 1–9 → 90 two-digit labels (10–99).** `0` is never a label prefix, so it
  is free as a control key.
- **Three-digit badges are permitted** when a window genuinely needs them, but the
  filtering should keep that uncommon. (See the measurement below — this needs the scope
  ladder to actually hold.)
- **Reading order**, banded into rows top-to-bottom then left-to-right within a row.
- **Scope: foreground window + its popups by default**, with a key to widen.
- **Visible action legend** on the overlay. No hidden compound codes.
- **Auto-suspend when a click lands in a text field**, so digits reach the field.
- **Global digit and Escape capture is accepted**, guarded by a dead-man timer.
- **Voice selection**: saying "thirteen" while hints are up clicks 13.
- **Gaze ordering is deferred.** The Tobii is disabled most of the time. Revisit later —
  numbering by distance from gaze would make the intended target almost always single-
  digit, and the hardware is already wired up via `WarpToGaze`.
- **OCR / computer-vision fallback is parked**, to be decided when we hit an app where
  UIA returns nothing.
- **`docs/gui-conventions.md` numpad rules do NOT bind this system.** Explicit exemption
  — this is a bare click-through canvas, not one of the seven blessed GUI shapes, and its
  no-Enter prefix-free input is deliberately different from the `N`/`N.M`/Enter
  convention. Do not "fix" it to match.

---

## What shipped (2026-07-27)

| Piece | File |
|---|---|
| Engine — scan, filter, order, label, overlay, modal loop | `Helpers\ClickHints\ClickHintsEngine.ahk` |
| Own-process host | `Scripts\ClickHintsHost.ahk` |
| Thin launchers (in MAINFUNCTIONS) | `Helpers\ClickHintsLaunch.ahk` |
| Per-app filter profiles | `INIDATA\click_hints_profiles.json` |
| Voice — start the overlay | `caster\rules\click_hints_commands.py` |
| Voice — act while it's up (overlay-scoped) | `caster\rules\click_hints_active_commands.py` |
| Filter tuning harness (Python, reads `uia.txt` dumps) | `Diagnostics\hint_filter_proto.py` |

**Voice:** "make click" (content) · "make click window" (+ app chrome) · "make click all"
(every window). While it's up: say a number to click it, "right 13" / "double 13" /
"hover 13" / "middle 13", "click wider", "click again", "click stop".

**Keyboard:** type the label, no Enter. `00` widen · `01` right · `02` double ·
`03` hover · `04` middle · `05` rescan · Escape quit.

### Two label namespaces: numbers AND letters

Every hint carries both, rendered as **`15.AG`** — one box, monospace, letters uppercase.
Either activates it; the first character typed decides which namespace is in play, and
since digits and letters can never collide the two sets stay independent. `label_mode`
drops either (`"number"` gives narrower badges and is what voice uses).

Both are shaped around Jamie's **split keyboard** so a whole label is typed one-handed:

- **Digits are hand-grouped**, 1-5 left and 6-0 right. All 25 left pairs are issued
  first, then the 20 right pairs. Past 45, labels grow into same-hand **triples** rather
  than reaching for a two-handed mixed pair — three keystrokes on one hand beats two
  across both. Each expansion costs one label and yields five, net +4, and runs from the
  end of the ordering so the extra keystroke lands on the least-likely targets. **Nothing
  two-handed appears until 225 hints.** Verified offline: prefix-free, no duplicates, zero
  two-handed labels at n = 12/45/80/130/225.
- **Letters come from the left half only** — `asdfg` / `qwert` / `zxcvb`, home row first.
  15 × 15 = 225 two-character labels, all the same length so the set is prefix-free for
  free. Note the consequence: on a dense page numbers go to three digits while letters
  stay at two, so **letters become the faster path exactly when it matters**.

⚠️ **The badge font must stay monospace.** The badge is four controls at fixed
character-cell widths; with a proportional font the `.` is narrower than a digit and the
reserved cell leaves visible slack, rendering `15. AG`. Consolas advance is 0.5498em, so
the cell is `fs * 0.733`.

**Backspace is a single escalating undo** (Jamie's request, 2026-07-27): with digits in
the buffer it removes one and the dimmed badges come straight back; **on an empty buffer
it quits the whole overlay.**

**Partial input narrows the field, Vimium-style.** Type `2` and the `2x` labels stay gold
while everything else recedes to grey; *within* each surviving badge the digits already
typed are dimmed and the digits still to come stay solid. Backspace restores the field.

Each badge is therefore **two adjacent Text controls** sharing one background — consumed
prefix, then remainder.

> ⚠️ **Flicker took three separate fixes.** Jamie caught each one.
> 1. **Don't rebuild the Gui.** The first attempt destroyed and recreated the badge
>    window per keystroke — every badge flashes, reading as "the whole overlay
>    refreshed" rather than "the field narrowed".
> 2. **Don't invalidate the whole window either.** The second attempt mutated controls
>    in place but finished with one `WinRedraw(overlay.Hwnd)`, which repaints *every*
>    badge including unchanged ones — still a visible full-screen flash. Invalidate only
>    the controls that changed, via `GuiCtrl.Redraw()`.
> 3. **Double-buffer the window** — `WS_EX_COMPOSITED` (`E0x02000000`) alongside the
>    existing layered/transparent bits, so a repaint composites off-screen instead of
>    clear-then-draw. Verified this does not break rendering on a layered TransColor
>    window (it can conflict on some Windows versions — it doesn't here).
>
> Geometry is also only `Move()`d when the prefix split point actually shifts, so a
> badge that merely changes colour never triggers a window-region recalculation.

**The pop-in flash is a separate bug and took three goes.** Showing the window and calling
`WinSetTransColor` afterwards paints one fully opaque near-black frame — the "whole screen
flickers as the overlay appears" report. Fixes, in order of how much they mattered:

1. Both windows are built with `Hide`, styled, and only then shown via
   `ShowWindow(hwnd, SW_SHOWNA)`. This alone left it flashing ~60% of the time.
2. `WinSetTransColor` now runs **immediately after `Gui()`**, before a single control is
   added, so the background is keyed out before the window can ever paint.
3. **`WS_EX_COMPOSITED` was removed.** It had been added for double-buffering, but on a
   window this large it forces a DWM recomposite when the window appears — itself a
   full-screen flash. Per-control invalidation is what actually fixes keystroke flicker;
   the composited style was only ever belt-and-braces. *If per-keystroke flicker returns,
   this is the thing to put back — but shrink the window first.*
4. **The overlay is no longer full-screen.** It used to span the whole 3640×1920 virtual
   desktop. Badges only ever cover the target window, so pass 1 places them in screen
   coords and takes the union, and the window is sized to that bounding box. Far less
   surface to bring up.

**The status bar is click-through and translucent.** It sits over the bottom-left of the
screen where there are usually real hints; as a solid window it swallowed those clicks and
blocked the view. It now carries the same `WS_EX_TRANSPARENT` as the badge layer plus
`WinSetTransparent(170)`, so hints underneath it are both visible and clickable.

A digit that can't lead anywhere is rejected rather than poisoning the buffer.

**Badge placement is size-dependent** (`badge_position`, default `auto`). Centre on
targets at least `badge_center_min_w` × `badge_center_min_h` (200 × 56), corner on
anything smaller.

Neither alone works. Corner placement makes big targets ambiguous — on a Plex result card
the card's badge and the poster's badge end up squashed together in the same corner with
no way to tell which is which. But centring a badge on a one-word link buries it in the
text. The split costs two integer comparisons per hint, so it is free. `"center"` /
`"corner"` force one everywhere.

### Click history — `%TEMP%\click_hints_history.jsonl`

One JSON line per click: timestamp, action, label, control type, Name, AutomationId,
rect, click point, focusable flag, scope, exe, window title.

Immediate purpose is debugging — when a click lands somewhere unexpected, this says
exactly which element the filter handed over without needing to reproduce it live.

**Planned: fold this into the macro recorder** (Jamie, 2026-07-27 — explicitly *not* now).
A recorded click-hints session becomes a list of concrete, re-findable UIA selectors
rather than bare coordinates, which is the programmatic form a macro actually wants. The
record shape is deliberately kept close to `make record`'s recordings
(`INIDATA\UIARecordings\*.json`, see `references/uia-clicking.md`) so the two can merge,
and the intended UI is the same shape as `open dumps` — a Miller listing recent sessions
with actions on each. The longer-term goal is that acting through the recorder captures
as many elements as possible automatically, so macros can be written programmatically
instead of by coordinate.

Voice works by sending the digits as real keystrokes into the host's global hotkeys, so
voice and keyboard share one input path. It must use dragonfly `Key()`, not `Text()` —
`Text()` sends unicode packets that AHK hotkeys never see.

### `probe` — the diagnostics mode you'll actually use

```
AutoHotkey64.exe Scripts\ClickHintsHost.ahk probe <scope> <hwnd>
```

Scans, filters, labels, writes `%TEMP%\click_hints_probe.txt`, exits. **Draws nothing and
arms no hotkeys**, so it's safe to run any time — this is how to tune a profile without
the overlay seizing the keyboard.

### Measured live, after the perf fix

| Window | raw elements | hints (content) | scan |
|---|---:|---:|---:|
| Chrome — YouTube video page | 1274 | 87 (all two-digit) | **141 ms** |
| Chrome — same, window scope | 1274 | 122 (36 three-digit) | 168 ms |
| VS Code | 1418 | 46 | **203 ms** |
| Signal | 149 | 42 | **32 ms** |
| Kindle for PC | **21** | 2 | 16 ms |

Content scope holds under 90 (two-digit) on every real window tried. The live numbers
track the dump-corpus predictions closely.

### ⚠️ The performance trap (cost us 11-14×)

The obvious way to batch UIA property reads is wrong:

```ahk
els := root.FindElements({}, 5, 0, 0, cacheRequest)   ; DON'T
```

That overload loops `BuildUpdatedCache` **per element** (`UIA.ahk:3081`) — one COM
round-trip each. Use a single subtree cache instead:

```ahk
els := root.BuildUpdatedCache(cacheRequest).GetCachedChildren(5)   ; DO
```

One cross-process call for the whole tree, then every `.CachedX` read is in-process.
Measured on the 1274-element YouTube tab: **1594 ms → 141 ms**. VS Code 2875 → 203 ms.
This is what made the click-rescan loop viable at all; a two-second rescan after every
click would have killed the interaction.

### The Plex "big box" problem — and why it was a general bug (fixed 2026-07-27)

Jamie's `ListNavClickNth` profiles exist partly because hint-style systems kept surfacing
the little links *inside* a Plex show card and never the card itself, so nothing actually
opened the show. v1 reproduced that exact failure. The tree explains why:

```
[link]   (NO Name, NO AutomationId)  [332,310 883x113]   <- the box you must click
  [heading] "Steven Universe"        [440,333 120x25]
  [button]  "steelflicks"            [440,377 120x24]
  [button]  "Play"                   [1107,343 48x48]
  [button]  "More Actions"           [1155,343 48x48]
  [button]  "Select Steven Universe" [332,310  32x113]
```

The card **is** in the tree, as an unnamed `[link]`. It was being killed twice over:

1. `require_name` dropped it for having neither Name nor AutomationId.
2. `nesting: leaf` would have dropped it anyway for containing other candidates.

Both filters were too blunt, and both fixes are **generic, not Plex special-cases**:

- **`require_name` now spares keyboard-focusable elements.** An unnamed `<a>` box is a
  real target; unnamed *non*-focusable filler still goes.
- **`nesting: "smart"` is the new default** (see the correction under Finding 4). The card
  is focusable so it survives; Play and More Actions are focusable so they survive too —
  correct, because opening the show and playing it are different actions. A Signal chat
  bubble is *not* focusable, so it still yields to its Options button.
- **Nameless survivors borrow a label** from the heading inside them, so probe output and
  logs stay readable.

Regression-checked live: Chrome 58 hints, VS Code 49 — no explosion in count, and 10
nameless-but-focusable elements (the Plex-box class) now survive on a Chrome page.

**Not yet confirmed on Plex itself** — the window wasn't open when this was written. The
check is `probe 1 <plex-hwnd>`: the card should appear as a `[link] FOC` roughly 883×113
carrying the show's name.

### Kindle for PC exposes almost nothing

21 UIA elements total, of which exactly 2 are clickable (the page-turn arrows). Not a
filter failure — there is nothing else in the tree. **This is the concrete case that
decides the parked OCR-fallback question**, and it's an app Jamie uses constantly. The
existing `Lib\OCR.ahk` + the GrabText engine already number text on screen in this exact
window, so the fallback has a working precedent to borrow.

## Measured: what filtering actually yields

A prototype filter was run over **12 real `uia.txt` dumps** (9 Chrome, 2 Signal, 1 VS
Code) captured from ordinary use. Pipeline: control-type allowlist → drop degenerate
rects → drop rects >40% of the window → drop offscreen → dedup exact rects → dedup
near-identical rects (4px) → prefer leaves over wrappers → drop nameless *and* aidless.

| raw elements | hints (all) | hints (page only) | window |
|---:|---:|---:|---|
| 1156 | 171 | 131 | Spotify Search (Chrome) |
| 969 | 103 | 59 | Google Voice (Chrome) |
| 865 | 56 | 56 | Mokwheel product page (Chrome) |
| 676 | 151 | 114 | Spotify album (Chrome) |
| 535 | 99 | 57 | Google search results |
| 247 | 72 | 35 | Bandcamp checkout |
| 242 | 111 | 69 | Steven Universe (streaming) |
| 190 | 101 | 64 | INVINCIBLE (streaming) |
| 150 | 47 | 42 | Signal |
| 134 | 46 | 46 | VS Code |
| 103 | 31 | 26 | Signal |

**Median 100 hints at full scope; 58 with browser chrome excluded.** Over-90 cases fall
from **7 of 12** to **3 of 12**.

### Finding 1 — 90 labels is not enough at full scope

This contradicts the assumption that good filtering alone keeps three-digit badges rare.
It does not. Native apps are fine (Signal 31–47, VS Code 46), but Chromium apps land at
100–171 and those counts are *legitimate* — those pages really do have that many
clickable things. Spot-checked: Spotify Search's 169 hints are 100 buttons, 42 links, 10
check boxes, and the repeated 17×32 elements are per-track row controls. Not filter junk.

### Finding 2 — browser chrome is a flat ~40 hint tax

Every Chrome window spends **~37–44 hints** on the tab strip, toolbar, and extension
icons before the page contributes anything. Two dumps had 19 and 21 elements of exactly
34×34 — the extension button row.

**This is the fix for Finding 1: a scope ladder rather than paging.**

1. Page/content region of the foreground window — median **58**
2. \+ application chrome (tabs, toolbar, menu bar) — median **100**
3. \+ all visible windows / whole desktop

`0` is already reserved as a non-label digit, so **`0` widens one rung and re-scans**.
That preserves "no Enter", keeps two-digit labels in the common case, keeps browser
chrome genuinely reachable (a thing Vimium can never offer), and is discoverable from
the action legend.

### Finding 3 — offscreen rejection is the single biggest lever, and we can't see it yet

On Google Voice the allowlist cut 969→378, then offscreen rejection cut 378→142. That
one stage removes **62%** of surviving candidates — the scrolled-out conversation rows.

The prototype approximates it by testing whether an element's centre falls inside the
window rectangle. The real signal is UIA's **`IsOffscreen`** property, which handles
clipping by any scrolling ancestor, not just the window edge. `Diagnostics/UIADump.ahk`
does not capture it today. **Adding `IsOffscreen` is the highest-value change to the
element source.**

### Finding 4 — wrapper-vs-leaf cannot be resolved geometrically

Two cases with identical geometry and opposite correct answers:

- A Google Voice **chat bubble** (688×45) wrapping a 40×41 "Options" button. The bubble
  is not a target; the button is. → **keep the leaf**.
- A Spotify **album tile** wrapping a play button. The tile opens the album and is the
  thing you want; the play button is secondary. → **keep the wrapper**.

Size ratio does not separate them. What separates them is whether the wrapper is *itself
an independent target*.

> **⚠️ Correction to this doc's original recommendation.** It said to use
> `IsInvokePatternAvailable`. **Measured on live Chrome: nothing reports it — not one
> element on a real page.** Chromium exposes activation through
> `LegacyIAccessible.DoDefaultAction` instead, so an Invoke-based rule silently
> classifies every Chromium element as "not a target". **Use `IsKeyboardFocusable`**,
> which does populate and is arguably the better signal anyway: real interactive targets
> take keyboard focus, decorative containers don't.

This is what the shipped `nesting: "smart"` mode does — drop a wrapper only when it is
**not** keyboard focusable. A focusable wrapper survives alongside its children, because
they genuinely do different things.

⚠️ `Helpers/UIAActions.ahk` carries a hard rule: **never touch a ValuePattern property**
(`el.Value` / `el.Dump()`) — it builds a live `IUIAutomationValuePattern` that crashes in
`__Delete` on edit controls. Any pattern probe added here must be guarded and tested
against edit controls specifically.

### Finding 5 — do not round-trip through the text file

`uia.txt` is a lossy rendering. Element names contain embedded newlines, so records
cannot be split on line boundaries (several dumps have name fragments that parse as
bogus elements). The live system must work on in-process UIA element objects. This also
removes the `AutoHotkey64.exe` spawn + `RunWait`, which dominates the current 1–2 s dump
time.

---

## Per-context filter profiles

Anticipated from the start: Signal will want different rules than VS Code than Spotify.
The prototype's counts already show why — "prefer leaves" is correct in Signal and wrong
in Spotify.

**Built on day one**, as planned, so nothing needs retrofitting: `INIDATA/click_hints_profiles.json`,
keyed by lowercased exe, with a `default` block merged first and the exe block merged over
it. Tunable per app: `types`, `nesting` (leaf/wrapper/both), `min_size`,
`max_window_fraction`, `require_name`, `near_dedup_px`, `row_band_px`, `settle_ms`,
`badge_font_size`, `exclude_name`.

Deliberately a **standalone file rather than a `hints` block inside `INIDATA/Contexts/`**,
which is where ListNav's `selectors` live. Reasoning: this is filter *tuning data*, not a
context *definition*, and keeping it separate means a bad edit can't destabilise the
context/binding/Stream-Deck registry. If it later wants context inheritance (parent-chain
fallback, URL matching), migrating into Contexts is the obvious move.

Seeded with `spotify.exe → nesting: wrapper` and `signal.exe → nesting: leaf` to make the
knob's purpose legible from the file itself.

---

## Reuse map — what already exists

AHK root = `~/Desktop/Important/AutoHotkey/`.

| Need | Already built |
|---|---|
| Click-through numbered badge overlay | `_GTShowOverlay()` / `_GTBar()`, `Helpers/GrabText/GrabTextEngine.ahk` — `+E0x80020` (layered + transparent) with `WinSetTransColor("010101")`, gold `FFE24D` badges |
| Numpad modal capture loop | `GTShowGrab()` + `_GTNumKey` / `_GTNumEnter` / `_GTNumBack` / `_GTNumCancel`, same file — including a 2-minute auto-exit dead-man timer |
| Own-process host skeleton | `Scripts/GrabTextHost.ahk` — `#SingleInstance Force` means relaunching kills the previous overlay for free |
| Launcher that captures the pre-launch hwnd | `_GTHostRun()` in `Helpers/GrabTextLaunch.ahk` (hidden launcher does not steal focus) |
| UIA element enumeration with screen rects | `Diagnostics/UIADump.ahk` — `UIA.ElementFromHandle(hwnd).FindElements({}, 5)` |
| Click a screen point with modifiers / multi-click | `_LN_ClickScreen()` in `Helpers/ListNavFunctions.ahk` |
| Guarded UIA click | `UiaClickElement` / `_UiaClickElementGuarded`, `Helpers/UIAActions.ahk` |
| Voice mode gating | `window_exists(title=...)` in `caster/rules/window_context_helpers.py` — raw `FindWindowW`, cheap enough for a `function_context` predicate |
| Per-context profiles precedent | `INIDATA/Contexts/*.json` + `Helpers/ListNavFunctions.ahk` |

Descolada's **UIA-v2 is already vendored** at `UIA-v2-main/Lib/UIA.ahk` and is already
`#Include`d by `MAINFUNCTIONS.ahk` ahead of the Helpers — a new hint function gets `UIA`
for free with no include work.

Caster's bundled Legion grid is enabled and is the closest prior art already installed,
but it finds hints by **pixel analysis** (PIL `ImageGrab` in a separate tkinter process),
so it hits text and misses icon-only buttons. Not the foundation we want.

---

## How the shipped v1 resolved the open questions

| Question | Decision in v1 |
|---|---|
| Re-scan trigger | Auto-rescan after every click. Settle = poll the cheap foreground identity (hwnd + title) every 60 ms until stable for two consecutive polls (max ~480 ms), then a fixed `settle_ms` tail for in-page rendering. Deliberately avoids walking the tree just to decide whether to walk the tree. Jamie left this to me and expects iteration. |
| Which window after a click | Re-acquires the foreground window every scan; never reuses the launch hwnd. Scope 1/2 also sweep other visible top-level windows of the **same PID**, so menus, popups and dialogs get hinted. |
| Action encoding | `0` opens a control namespace: `00` widen · `01` right · `02` double · `03` hover · `04` middle · `05` rescan. All shown in the legend. Voice uses the **same prefix form** ("right thirteen"), per Jamie's preference, rather than a suffix. |
| Click method | Real mouse click at the rect centre. Jamie confirmed she doesn't use a mouse and doesn't care where the cursor lands, which removes the only argument for `Invoke()` — and `Invoke()` silently no-ops on Chromium/Electron anyway. `GetClickablePoint()` is the obvious refinement if centre-clicks ever hit an occluder. |
| DPI awareness | Host calls `SetThreadDpiAwarenessContext(-4)` (per-monitor v2) before anything else, so AHK's coordinates match UIA's physical pixels. |
| Badge collision | Nudge-down-on-overlap, up to 6 attempts. Badge width scales with label length so three-digit badges don't clip; height and width both derive from `badge_font_size`. |
| Multi-monitor | One layered click-through window spanning the **whole virtual screen**, not per-window client area. |

### Still genuinely open

1. **OCR fallback (Jamie's Q9, parked for discussion).** Kindle for PC is the forcing
   case — 21 UIA elements, 2 clickable. Options: automatic fallback when the hint count
   is below a threshold, or a separate explicit command. `Lib\OCR.ahk` and the GrabText
   engine already do exactly this numbering in that exact window.
2. **Confirm the Plex box fix on Plex.** Logic verified generically; the actual page
   wasn't open. `probe 1 <plex-hwnd>` settles it.
3. **Whether `smart` over-hints anywhere.** Keeping focusable wrappers *and* their
   focusable children is right for Plex cards but adds badges. If some app doubles up
   pointlessly, that app gets `nesting: "leaf"`.
4. **Settle timing.** May prove too eager on slow SPA route changes, or too slow on
   snappy native apps. `settle_ms` is per-app for exactly this reason.
5. **Badge legibility.** `badge_font_size` defaults to 9. Raise it in the `default`
   profile if it reads small.
6. **Redraw cost on dense pages.** The badge layer is rebuilt on every keystroke to do
   the dimming. Imperceptible at ~60 badges; if it lags at 150+, switch to mutating
   control colours in place instead of destroy-and-recreate.
7. **Whether global digit capture is annoying in practice.** The text-field auto-suspend
   and the 3-minute dead-man are the guards; real use decides if they're enough.

### The "it just shuts out for no reason" bug (fixed 2026-07-27)

Clicking certain links quit the entire overlay; others re-scanned fine. The cause was the
text-field auto-suspend: `_CHFocusedIsTextField` counted `document` as a text field, and
**after any browser navigation focus lands on the page's RootWebArea, which reports
LocalizedType `document`.** So every click that changed pages tripped the suspend — which
is exactly why it looked random: only *navigating* clicks did it.

Now only `edit` counts. `document`, `combo box` and `search` were all removed, and there
is a comment in the code saying never to add `document` back.

### The stale-scan bug on SPA navigation (fixed 2026-07-27)

After navigating in Plex, the sidebar and bottom bar were re-hinted correctly but **none
of the new page's elements ever appeared** — the system behaved as though the page had
already refreshed.

`_CHSettle` polls `hwnd + window title` and waits for two stable readings. **A
single-page app changes neither on navigation**, so it reported "settled" on the first
poll, the re-scan hit the old DOM, and the parts of the page that genuinely don't change
(chrome, sidebar) made the result look plausible.

Fixed by making the re-scan *content*-aware rather than window-aware. `_CHSignature`
fingerprints a hint set (count + each element's position and name — positions alone are
insufficient, since an SPA route swap can reuse the exact same layout). `_CHActivate`
captures the signature before clicking and passes it to `CHRescan`.

**Waiting for the signature to merely DIFFER was not enough** — that was the first attempt
and it still failed. A page mid-load re-renders its **chrome first**: Plex's left nav and
bottom bar come back before the content does, so "it differs now" fired while the page was
still loading and the scan froze on a half-built DOM.

The working test is **stability**: re-scan until two consecutive scans are identical *and*
different from the pre-click signature, up to 8 attempts at 120 ms. An `A_Index >= 4`
escape hatch covers a click that legitimately changes nothing (a toggle), which would
otherwise never satisfy "different".

`_CHSettle` was correspondingly gutted to a plain sleep — its hwnd+title poll was
worthless for SPAs and cost up to 480 ms doing it.

### Findings worth not relearning

- **`IsInvokePatternAvailable` is useless on Chromium** — reports for nothing. Use
  `IsKeyboardFocusable`. (Cost: one wrong rule, caught by probing before shipping it.)
- **Never batch UIA reads via `FindElements(..., cacheRequest)`** — per-element COM calls.
  `BuildUpdatedCache(cr).GetCachedChildren(5)` is 11-14× faster.
- **An unnamed element is not automatically filler.** If it takes keyboard focus it is a
  real target — this is the entire Plex-card bug.
- **Don't infer one tool's coverage from another tool's tree** (the Vimium error above).
- **`document` is not a text field.** It's what a browser focuses after every navigation.
- **Don't rebuild an overlay to restyle it.** Mutate controls in place; a rebuild flashes.
  And don't `WinRedraw` the window either — invalidate only the controls that changed.
- **A single-page app changes neither hwnd nor title on navigation.** Any "has the page
  settled?" check based on window identity is worthless there; fingerprint the content.
  And "the content differs now" is *still* not settled — chrome re-renders before content.
  Wait for two consecutive identical scans.
- **Read `ahk_event.log` first.** A `Format()` call left one `{}` unfilled, so every badge
  control's options string ended `Background c202020` with no colour and `Gui.Add` threw
  "Invalid option" — silently, because `OnError` returns -1. Five minutes of theorising
  versus one `grep -a ClickHints ahk_event.log`. This is Jamie's standing rule and it was
  right again.
- **Don't inject synthetic keystrokes to test while Jamie is at the keyboard.** A test
  `SendKeys` landed in her chat box. Use `preview` mode (draws the overlay for N seconds
  and registers NO hotkeys — safe any time), `probe` mode, or the `ClickHints/key` log.
- **In AHK v2, assigning to a name a function didn't declare `global` creates a LOCAL.**
  `CHRescan` populated `CH_ALPHA` without declaring it, so the real map stayed empty:
  letters rendered on every badge but no keystroke ever matched, the buffer reset on each
  press, and it looked like the hotkeys weren't firing at all. The `ClickHints/key` log
  line settled it in one read — `key=a buf=a ns=alpha hit=0` followed by `key=v buf=v`
  (buffer reset, not accumulating) points straight at an empty lookup table.

---

## Testing status

Verified by me: AHK validates, include closure resolves, JSON parses, both Caster rules
compile, `MAINFUN.bat ClickHintsStart` returns `OK` and brings up the host, the overlay
renders correctly at 2 hints and at 122 hints (reading order, three-digit badges,
collision nudging, legend all confirmed by screenshot), and scan timings were measured
live across five applications.

**Not yet verified — needs Jamie at the keyboard:** actually typing a number and having
the right thing get clicked; the click → settle → rescan loop across a real navigation;
Escape and the dead-man timer under real conditions; voice number recognition into the
overlay; and whether the text-field auto-suspend fires when it should.

The `probe` mode is the safe way to iterate on filtering without any of that.
