---
tags: [autohotkey, context, macro-system, bindings, streamdeck, passthrough, always-on, design]
created: 2026-06-30
status: working
owner: Jamie
---

# Always-On Context Detector + Per-Context Keyboard Passthrough

A single always-on AutoHotkey process that **knows the current foreground context at all times** and **announces every change** to a list of observers. The first two observers:

1. **Stream Deck auto-switch** — swap the active deck profile when the context changes (the reason we're building the detector now).
2. **Per-context keyboard passthrough** — make any key (arrows first, eventually anything) *truly native* in contexts where it has no binding, and a clean macro override in contexts where it does — with **no reload** and **no Dragon typing jangle**.

The detector is generic; everything else shells out to it. Build the detector once, well; consumers are ~30 lines each.

Related: [[CONTEXT_MANAGER]] (the context editor) · [[PER_CONTEXT_MATERIALIZER]] (voice-command scoping, the other per-context consumer) · `AutoHotkey/docs/macro_system_design.md` (the binding/mode system this plugs into) · `ahk-functions` skill `references/directory-program-registry.md` (the `Contexts/*.json` registry + `DetectContextChain`).

---

## Why this exists

### The passthrough problem (the original motivation)

Today, the keyboard binding system decides **at load time, per slot, globally** whether a key is intercepted — [`AlwaysOn/GeneralMacros.ahk`](../../../../AutoHotkey/AlwaysOn/GeneralMacros.ahk):

```ahk
prefix := _GM_boundSlots.Has(slot) ? "" : "~"   ; bound ANYWHERE → no tilde → intercept everywhere
```

Consequence: the moment a key is bound in **one** context, it loses its `~` (native passthrough) **everywhere**. In every *other* context the dispatcher intercepts it, misses, and **re-emits a synthetic key** ([`Helpers/KeyboardDispatcher.ahk`](../../../../AutoHotkey/Helpers/KeyboardDispatcher.ahk) ~line 654, `Send("{Blind}" passthrough)`). That synthetic re-emit is exactly what races Dragon and scrambles dictation ("jangle"), and it's why the macro layer for letters is gated behind a toggle instead of being always available.

So "seamless when not locally bound" is **false** under the current model: bind a key in Chrome, and in VS Code it's intercept-and-re-emit, not the native key.

### The fix: know the context ahead of time

If an always-on detector already **knows** the current context (because it tracked the last change), then a key's hotkey can be gated by a **HotIf membership check against a pre-computed set** — no detection at press time:

```ahk
; registered ONCE per candidate key:
HotIf((s => (*) => _CtxBoundSlots.Has(s))(slot))
Hotkey(slot, Keyboard_Dispatch)
HotIf()
```

- Slot **bound in current context** → gate true → intercept + fire action.
- Slot **not bound here** → gate false → **the key isn't hooked for this press → true hardware-native passthrough.** No re-emit, no synthetic key, no jangle.

The gate callback reads **only** the in-memory `_CtxBoundSlots` map — an O(1) hash lookup, low microseconds. All the real work (detect context, rebuild the set) happens off the keypress path, on the change event.

**This is strictly better than today:**
- True native passthrough where unbound (fixes typing).
- **No reload after binding** — every candidate key is always registered with a live gate; binding a key just writes `bindings.json` and triggers a set recompute. (Today you must reboot AHK to flip the static tilde.)
- Authoring flow is byte-for-byte identical — `set wizard` → press key → pick action. Only the backend changes.

---

## Architecture: one detector, many observers

```
┌─────────────────────────────────────────────┐
│ AlwaysOn/ContextDetector.ahk  (always-on)    │
│                                              │
│  • gated poll @ ~120ms  ─┐                   │
│  • CtxDetector_Notify()  ─┼─→ change? ─→ recompute chain ─→ fan out
│    (self-announce)        │                   │
└───────────────────────────┼───────────────────┘
                            │ on change, call every registered observer:
                ┌───────────┴───────────┬──────────────────────┐
                ▼                       ▼                      ▼
      StreamDeck swap         _CtxBoundSlots recompute     (future observers:
      (deck profile)          (keyboard passthrough)        ListNav, voice
                                                            materializer, …)
```

**Observer registry** (the generic seam): `CtxDetector_OnChange(handlerFn)` registers a callback `(newChain, oldChain) => void`. The detector calls each on change. Consumers never poll or detect themselves — they react.

---

## Detection strategy (DECIDED)

Three possible signals, divided by what each is **blind to**:

| Signal | Catches | Blind to |
|---|---|---|
| **Self-announce** | Changes caused by a known function (SwitchTopDesktop, OpenChromeTabs family, GUI opens) | Manual switches (mouse click, alt-tab, manual Chrome tab click, notification focus-steal) |
| **WinEvent foreground hook** | Every window switch | **Same-hwnd changes** (Chrome tab switch, SPA nav like YouTube home→watch) |
| **Poll** | Everything, incl. same-hwnd in-Chrome | Nothing (just adds ≤1 interval of lag) |

**Verdict: gated poll + self-announce. No WinEvent hook.**

- The **poll is mandatory** — it's the *only* signal that catches same-hwnd in-Chrome context changes (YouTube home→watch is literally the same window). The WinEvent hook is blind to exactly that case, so it can't replace the poll; it would only be a latency optimization for window-switches the poll already covers — not worth the DllCall complexity at a negligible 120ms.
- **Self-announce is not overkill** — it's the cheapest, most reliable signal for the changes it covers (the function *knows* it changed context → zero lag, guaranteed correct). Worth wiring in.

### The cheap gate (makes the poll nearly free)

Each poll tick:

1. **Cheap check first:** read foreground `hwnd` + window `title`. Compare to last tick.
2. If **both unchanged** → do nothing (steady-state cost = two `WinGet` calls, microseconds).
3. If **either changed** → run the real `DetectContextChain()` (incl. lazy Chrome URL fetch, already 500ms-cached), diff against the last chain, and if the *context* changed, recompute `_CtxBoundSlots` + fan out to observers.

Because the gate is cheap, the interval barely affects cost — **120ms** is both responsive and free. The title-change part of the gate is what catches in-Chrome page changes (YouTube home→watch changes the title), so it pulls double duty.

### Self-announce wiring

A single `CtxDetector_Notify(reason := "")` entry point that runs the same gated recompute immediately (eager trigger, same code path as the poll). Drop it into the tail of known context-changers:

- `SwitchTopDesktop` (and the virtual-desktop nav family)
- The `*ChromeTab*` opener family (`OpenChromeTabs`, `OpenOrFocusChromeTab`, `OpenOrNavigateChromeTab`)
- GUI openers (any function that brings up an AHK GUI — they change the foreground context guaranteed)
- App launch/focus primitives (`BringToFront`, `OpenAndPlace`)

These give 0-lag, guaranteed-correct updates for the programmatic case; the poll backstops everything manual.

---

## Consumer 1 — Per-context keyboard passthrough

### Data

- `_CtxBoundSlots` — in-memory `Map(slot → true)` for slots bound in the **current** context chain. Rebuilt on every context change by filtering `bindings.json` (device=keyboard) against the new chain (deepest-first, same specificity walk `ResolveBinding` uses).
- Recompute is also triggered **immediately after a wizard save**, so a freshly-bound key works with no reload.

### Registration

Every candidate key registered once with the HotIf-membership gate (above). Candidate set starts with **arrows + `[` `]`** (the SiteStep keys), expands to Home/End/PgUp/etc., and *eventually* can include letters/digits/punctuation — at which point the macro-layer toggle for typing keys can be retired in favor of true per-context gating.

### Hard invariant

**The HotIf gate callback reads only `_CtxBoundSlots` (+ the one-line modal-flag `FileExist` on the unbound path, same as the macro-layer gate). Never UIA, never `DetectContextChain`, never a bindings.json read inside the gate.** The gate runs per keypress on the hook thread; anything slow there = input lag. All heavy work lives in the detector's change handler, off the hot path.

### Wizard-capture + GUI-navigation (the gate's three outcomes)

`_CtxGateCheck(slot)` resolves to:
1. **bound in this context** → claim (normal SiteStep / binding fire).
2. **unbound + set-macro-wizard modal armed** → claim, so the key can be **captured for rebinding** even where it isn't bound yet. (Without this, arming the wizard in a non-step context left arrows unhooked → the wizard never saw the press. Fixed 2026-06-30.)
3. **otherwise** → don't claim → native passthrough.

In cases 1–2 the gate **yields to a foreground AHK GUI** (`Keyboard_IsAhkGuiForeground()`) so arrows **navigate the wizard picker** (or any GUI) instead of being intercepted. Whether the picker itself acts on arrow keys is a separate GUI concern — the gate just stops stealing them.

### Why this fixes typing

In a normal text context, ~all letters are unbound → their gates are false → keys are never hooked → pure hardware native → no synthetic re-emit → no Dragon jangle. Only the handful of keys you deliberately bound *in that context* intercept. (Bound keys keep the existing `Keyboard_DispatchPhysicalOnly` synthetic-vs-physical gate so Dragon dictation of a bound letter still passes through.)

### SiteStep migration (the immediate win)

The weather/Instagram Left/Right nav ([`Helpers/SiteStepFunctions.ahk`](../../../../AutoHotkey/Helpers/SiteStepFunctions.ahk)) becomes context-scoped bindings → `KeyboardAction_SiteStepLeft/Right` (wrappers already exist in `KeyboardDispatcher.ahk`). Result: arrows show up in the wizard + in `recent functions`/logs as "via keyboard" (they currently don't, because SiteStep is called from a raw lambda that bypasses `Dispatch()`), and the per-context gate handles the IG-suppress vs elsewhere-native split natively — no more dual tilde/non-tilde hotkey variants.

Jamie is fine with **native suppressed ~99% of the time** and specifically fine with **weather suppressed**, so the migration doesn't need to preserve the old tilde "native-also-fires" behavior on weather.

---

## Consumer 2 — Stream Deck auto-switch (WORKING, name-based, 2026-06-30)

Switches the physical Deck A profile when the foreground context changes — **focus-preserving, cursor-still, cross-desktop**.

**The switch mechanism (the hard part).** Elgato exposes *no* focus-preserving way to switch to an arbitrary profile from the backend on Windows (SDK only reaches read-only manifest profiles; the trigger-exe steals foreground 500ms; MCP/plugin routes need a custom plugin + read-only profile copies — researched + rejected). The working answer: a **Virtual Stream Deck** (SD 7.0+ feature) in **Fixed/persistent** mode, holding **Switch-Profile buttons** (the "PROFILE SWITCHER" VSD profile), parked **off-screen** (`WinMove` to x=5000 — it sticks). Its Qt buttons expose the UIA **Invoke** pattern, so `element.Invoke()` fires the switch with **zero focus steal, zero cursor move, works while off-screen and while cloaked on another virtual desktop**. This is the app's *own UI* driving its *own* internal switch — sidesteps every external-API limitation.

**Code:** `Helpers/VsdSwitcher.ahk`
- `_VsdFindWindow()` — finds the VSD (`StreamDeck.exe` + class contains `ToolSaveBits`) using `DetectHiddenWindows` so it resolves **cross-desktop**.
- `_VsdButtonGrid(hwnd)` — clusters the live UIA button bounding-rects into a `(col,row) → element` map (robust to window position + the tiny 10%-opacity buttons).
- `VsdInvokeButton(col,row)` — UIA-Invokes that button.
- `_VsdLoadProfileMap()` — loads + mtime-caches the name→position map (below).
- `_VsdResolvePos(chain)` — deepest-first chain → `"col,row"` (resolution rules below).
- `VsdTick(chain, fgChanged)` — the `CtxDetector_OnTick` observer (dwell-debounce below).

### Mapping — profile NAME, not a raw position (both decks)

A context's JSON carries **`"streamdeck_profile": "<Deck A profile name>"`** and/or **`"streamdeck_profile_b": "<Deck B profile name>"`** (e.g. `youtube.json` → `"Youtube"`, root `chrome.json` → `"Chrome"`). A context can drive **Deck A, Deck B, or both** — the two fields resolve independently and both buttons fire on the same context change. The name is human-readable and **survives a switcher relayout** — the brittle raw `"streamdeck_vsd": "col,row"` is gone (legacy field still honored for the Deck A field only, for un-migrated contexts).

**Which physical deck a button switches is intrinsic to the button** (its `DeviceUUID`), so the AHK side stays deck-agnostic — it just invokes a position. That's why `streamdeck.py vsd-ensure` sets `DeviceUUID` from the *target profile's own deck* (a Deck B profile's button must carry the Deck B serial). The switcher is **auto-managed**, not a hand-curated palette: `vsd-ensure` appends a button on demand (row-major), and **`vsd-gc`** reclaims buttons no context references (`streamdeck_profile[_b]`) and re-packs gap-free — so it never accumulates cruft or wastes the ~31-slot page. (`vsd-gc` reads the disk truthfully: a profile only survives if some context references it; clearing a context's mapping makes its button collectable.)

Name→position lives in **`INIDATA/vsd_profile_positions.json`** (`{"Youtube":"0,0","Google Voice":"2,0","Chrome":"0,1",…}`), generated by **`streamdeck.py vsd-map`** by reading the PROFILE SWITCHER's Switch-Profile buttons (`ProfileUUID` → profile name → its grid slot). AHK reads this map at runtime (mtime-cached) — it never parses Stream Deck manifests itself.

**Resolution (`_VsdResolvePos`), deepest-first over the chain:**
- `streamdeck_profile` present → look up the name in the map → that position.
- **Map MISS** (name not yet wired) → log a `Vsd/err` pointing at `vsd-ensure`, then **fall through to the parent context** — so an unwired child still lands on an ancestor's deck.
- **Default coverage for free:** root `chrome` carries `"Chrome"`, so **every Chrome tab with no closer mapping inherits the Chrome deck** via the same parent-chain walk. No separate default/revert logic.

### Editing the mapping — inside the Context Manager

The context↔profile link is a first-class facet in the **Context Manager** ("open context" / `OpenContextEditor`): a new **Stream Deck** row (`_CtxStreamDeckNode` in `ContextEditorMenu.ahk`) shows the current profile and drills into a **picker of every Deck A profile + (none/inherit)**. Picking one:
1. writes `streamdeck_profile` to the context JSON (`WriteContextProfile`);
2. runs **`streamdeck.py vsd-ensure "<name>"`** — which adds a Switch-Profile button to the switcher if one doesn't exist yet (first empty slot, **row-major**), regenerates `vsd_profile_positions.json`, and restarts Stream Deck (only when a button was actually added). Already-wired profiles are a no-op (no SD kill).

So wiring a brand-new context to a deck is one pick in the GUI — no manual button placement, no manifest editing.

### CONTIGUITY INVARIANT (the one correctness gotcha)

AHK targets a VSD button by **clustering the live UIA button centers into dense col/row bands** — so a manifest position equals the AHK band index **only when the populated switcher slots have no fully-empty column/row between populated ones**. The existing block is gap-free (cols 0-3 row 0, cols 0-2 rows 1-2); `vsd-ensure` keeps it that way by filling **row-major**. Never hand-place a switcher button into a slot that leaves a fully-empty column/row between populated ones, or the runtime map mis-targets. (Documented in both `streamdeck.py` and `VsdSwitcher.ahk`.)

### Dwell-debounce (unchanged from the POC)

**Driven by a per-tick dwell-debounce, not the context-change event** (`CtxDetector_OnTick(fn)` → `fn(chain, fgChanged)` every ~120ms poll). `VsdTick`:
- on any **foreground change** (`fgChanged` — tab/window switch, *including tab-to-tab within the same context*) restarts a dwell timer for the current context's resolved position;
- **invokes only after the target is stable for `_VsdDwellMs` (~250ms)**, once per dwell.

Behaviors: **re-assert on every tab switch** (incl. same-context YouTube A→B — corrects a manual deck change; re-invoking the current profile is a no-op, no thrash) and **reject flash-through** (a context grazed for <~250ms never commits). The change-event observer would do neither, which is why this consumer uses the tick hook.

**Three correctness rules:**
1. **Per-tick dwell-debounce + re-assert on every foreground change** (above). `_VsdDwellMs` tunable; ~250ms ≈ "present 2+ poll samples."
2. **The detector force-fetches a fresh URL** (`DetectContextChain(true)` → `_Ctx_GetCachedUrl(force)`). Without it a **fast tab switch** reads the stale 500ms URL cache, misdetects, and (title won't change again) **sticks**. The cache stays for per-keypress callers; only the detector bypasses it.
3. The detector exposes **two observer hooks**: `OnChange` (context-change, deduped — keyboard passthrough) and `OnTick` (every poll + `fgChanged` — SD dwell-switch). Pick by whether the consumer debounces/acts on a timer.

### Scope + future use cases (DECIDED — Chrome + AHK GUIs only)

This consumer is intentionally only for **`chrome.exe` contexts and AutoHotkey GUIs** — apps where one exe hosts many "pages" that native Smart Profiles can't tell apart. The detector already matches on `url_contains`/`url_regex` (Chrome) and `title_regex`/`class` (everything), so the same `streamdeck_profile` field extends to:
- **AHK GUIs by title** — e.g. a future Spotify-library GUI vs other GUIs share `AutoHotkey64.exe`; give each its own context (distinct `title_regex`) + `streamdeck_profile`, and the deck follows. Needs no new mechanism — just context files.
- **VS Code workspaces** — per-project decks via `title_regex` on the window title (`… — project — Visual Studio Code`). Same pattern; only the contexts are new.

Both are **already supported by the design**; they're just not wired yet (no contexts created). Keep the field generic — never hardcode "chrome" anywhere in `VsdSwitcher`/the map.

**Remaining known gaps:** the switcher is single-page (the contiguity invariant + `vsd-ensure` assume one page; multi-page paging of the switcher is a future extension if >~31 profiles ever need decks); native Smart Profiles on `chrome.exe` can still tug against the context driver (strip `AppIdentifier` from any chrome-owned profile the context system drives).

---

## Performance summary

| Path | Cost |
|---|---|
| Poll tick, no change | 2× `WinGet` (hwnd+title compare) — microseconds |
| Context switch, non-Chrome | detect + `bindings.json` filter — **<2ms** |
| Context switch, Chrome (URL needed) | + UIA URL fetch (500ms-cached) — **~5–30ms**, once, off keypress path |
| Keypress (any gated key) | one `Map.Has()` — low microseconds |
| Idle | one 120ms timer doing near-nothing — negligible |

Feels instant; not a resource hog.

---

## Risks & edge cases

- **Stale set right after a switch** (the one real failure mode): for up to ~120ms after switching, the set reflects the old context, so a keypress in that window could do the old context's thing. Mitigated to near-zero by self-announce on programmatic switches; for manual switches the ≤120ms window is rarely hit (you don't press a bound key in the first 120ms of switching). Acceptable.
- **Title-gate miss:** a same-hwnd context change where the URL changes but the title does *not* would be missed until the next title change. Rare (most page nav changes the title); contexts usually align with title boundaries. Accept; revisit only if a real case appears.
- **Chrome URL detection flakiness:** the detector uses the same `DetectContextChain`/`ChromeActiveAddress` the whole system already relies on — not a new risk, the existing one.
- **HotIf closure capture:** capture `slot` per-iteration (the `(s => …)(slot)` IIFE pattern) or every key shares the last loop value — standard AHK gotcha.

---

## Build sequencing

1. ~~**Detector core**~~ — **DONE 2026-06-30.** Engine `Helpers/ContextDetector.ahk` (gated poll, `_CtxCurrentChain`, `CtxDetector_Notify`, `CtxDetector_OnChange` observer hub, `CtxDetector_CurrentChain/Context`); bootstrap `AlwaysOn/ContextDetector.ahk` (registers a verification observer + `CtxDetector_Start(120)`), `#Include`'d by `Parent.ahk` so it runs in the persistent process alongside the keyboard hotkeys. Also `#Include`'d by `MAINFUNCTIONS.ahk` so transient processes can call `CtxDetector_Notify` (cross-process via a TEMP flag file the poll honors). No `StartupOrder.bat` change needed — Parent already launches it; reload via `ReloadWithNotice`. Verified: boot prime, `winchange` gate detection, cross-process self-announce flag consumption, observer fan-out all fire in `ahk_event.log`.
2. **Self-announce wiring** — **DONE 2026-06-30 (surgical).** `CtxDetector_Notify` now fires from: **`WriteBinding`** (BindingResolver, both `replaced`/`added` paths) so a freshly-wizard-bound key goes live with **no reload**; and **`SwitchTopDesktop`** (synchronous desktop switch = instant context change). To make notify actually refresh consumers when the *context* didn't change (only the *bindings* did), `_CtxDetector_Recompute` gained a `force` param — the dirty/notify path fans out even on an unchanged chain (logs `ContextDetector/refresh`). NOT hooked into async navigations (`OpenOrNavigateChromeTab` etc.) — the page loads after the function returns, so `Notify` would fire before the context changes; the poll's title-gate is the right mechanism there. `CtxDetector_Notify` is callable from every process (added `ContextDetector.ahk` to `_CuratedClosure.ahk` + `MAINFUNCTIONS.ahk`; cross-process via the TEMP flag file).
3. ~~**Passthrough consumer**~~ — **DONE 2026-06-30.** Engine `Helpers/ContextPassthrough.ahk`: `_CtxBoundSlots` + `CtxPassthrough_Recompute` (detector observer, membership via real `ResolveBinding` per candidate) + `CtxPassthrough_RegisterHotkeys` (`#HotIf _CtxGateCheck.Bind(slot)` → `$Left`/`$Right` → `Keyboard_Dispatch`). Candidates = `["Left","Right"]` (`[`/`]` excluded — `]` is SwitchTopDesktop). Registered + observed from the `AlwaysOn/ContextDetector.ahk` bootstrap (before `CtxDetector_Start` so the boot prime fills the set). **SiteStep migrated**: 4 records in `bindings.json` (Left/Right × `google_weather`/`instagram_web`) → `KeyboardAction_SiteStepLeft/Right` (now pass `suppressedNative=true`); the hardcoded tilde/non-tilde arrow hotkeys removed from `GeneralMacros.ahk`. Also added **noise filtering** to the detector (skip fan-out when foreground is a `noise:true` context, so tooltips/popups don't flicker consumers). Verified: clean boot, gated hotkeys register, `bound=[]` on non-step contexts (code/google_voice → arrows native). **Pending behavioral test**: arrows actually stepping on `google_weather` + the IG feed (`instagram_web`).
4. ~~**Stream Deck consumer**~~ — **DONE 2026-06-30 (POC), name-based foundation 2026-06-30.** POC (raw `streamdeck_vsd` position + `CtxDetector_OnChange`) → reworked to the **name-based** system: `streamdeck_profile` field + `INIDATA/vsd_profile_positions.json` map (`streamdeck.py vsd-map`) + `_VsdResolvePos` name→pos with parent-chain fallback + dwell-debounce on `OnTick`. **GUI editing** via the Context Manager "Stream Deck" facet (`ContextEditorMenu.ahk`), which calls `streamdeck.py vsd-ensure` (auto-adds the switcher button row-major + regenerates the map + restarts SD on a real add). **Default coverage** = root `chrome → "Chrome"`. Migrated: `youtube → Youtube`, `google_voice → Google Voice`, `chrome → Chrome`. Verified live in `ahk_event.jsonl`: boot detected `[google_voice,chrome]` → `Vsd/switch invoked button 2,0`. Per-context wiring of more Chrome contexts (Instagram/Plex/etc.) deferred — each needs its own Deck A profile built first.
5. **Expand candidate keys** — **DONE 2026-06-30** in waves (all verified working, "so much better than the old system" per Jamie):
   - **Wave 1 (nav/special)** ✅ — Up/Down/Home/End/PgUp/PgDn/Insert/Delete moved off GeneralMacros' dynamic-tilde into the gate. Home/End now truly native outside Kindle; Up/Down newly bindable; PgUp/PgDn keep their global SiteScroll.
   - **Wave 2 (F-keys)** ⏭️ **SKIPPED deliberately** — F-keys are almost all global-scope, so the gate would behave identically to dynamic-tilde (no per-context native benefit), and they carry the special cases (`~*F15`, chord bypass, F13/14/17/18 hold-repeat). Already fully wizard-bindable + logged. Pure-refactor risk for zero gain → left on dynamic-tilde.
   - **Wave 3 (typing keys)** ✅ — letters a-z + digits 0-9 + punctuation, registered through `Keyboard_DispatchPhysicalOnly` (Dragon-safe) with **layer-aware membership** (a letter bound under `keyboard:macro_layer:on` is claimed only while the layer is on). The macro-layer toggle calls `CtxDetector_Notify` so membership tracks the layer exactly like a context change (the macro layer is just part of the binding environment). The old macro-layer block in `GeneralMacros.ahk` is **commented out, not deleted** (revert path). Net win: unbound letters are now *truly native* (no synthetic re-emit) → the Dragon typing-jangle is gone, and only the few letters you deliberately bound in a context intercept. `Space` and the `+$letter` caps-shim were dropped — they were re-emit-latency workarounds the per-context model makes unnecessary.

   Reusable optimization baked into `CtxPassthrough_Recompute`: `KeyboardBoundSlots()` pre-filters to slots that actually have a keyboard binding, so a context/layer change costs one bindings load + a few `ResolveBinding`s, not 55.

---

## Open decisions

- ~~Context→Stream-Deck-profile mapping: where does it live?~~ **DECIDED:** `"streamdeck_profile": "<name>"` on the `Contexts/*.json` profile; name→position via `INIDATA/vsd_profile_positions.json` (regenerated by `streamdeck.py vsd-map`/`vsd-ensure`); edited in the Context Manager "Stream Deck" facet.
- ~~Do letters ever move into this system?~~ **DECIDED:** yes — Wave 3 done (layer-gated, Dragon-safe).
- **Which Chrome contexts get their own deck?** Only `youtube`/`google_voice`/`chrome`(default) wired so far. Others (Instagram/Plex/Reddit/Gmail/…) each need a Deck A profile built first — wire per-context on demand.
- **Switcher paging** — single-page only today (contiguity invariant). Revisit only if >~31 profiles ever need decks.
- Poll interval: **120ms**; trivially tunable since the gate makes it cheap.
