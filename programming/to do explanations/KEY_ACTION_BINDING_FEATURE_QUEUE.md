---
tags: [autohotkey, key-action, binding, wizard, streamdeck, macro-system, context, design, queue]
created: 2026-07-04
status: design-queue
owner: Jamie
related: ["[[KEY_ACTION_FUNCTIONS]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[UNIFIED_MILLERS]]"]
---

# Key-Action & Binding — feature queue (2026-07-04)

Five requests Jamie raised while using the new key-action + binding + wizard system, captured pre-compact. Each item: **her ask** (verbatim intent), **approach / options**, **where** (code locations), **recommendation**. Build post-compact. Items 2 & 3 are really **binding-layer** features (apply to ANY function bound to a device key, not just key-actions) — note the wider scope.

Standing context recap (so a fresh session has it):
- Key-actions: store `INIDATA/key_actions.json` → `Scripts/gen_key_actions.py` compiles → real functions in `Helpers/Generated/KeyActions_<Group>.ahk` (+ `.meta.json` sidecars + `fn_to_file_index.json` entries). Runtime `Helpers/KeyActionFunctions.ahk` (`KeyActionSend`).
- Authoring is UNIFIED into **show fun** (`OpenVoiceCommandEditor`) via the portable `_KeyActionNodes(fn, reopenFn)` cluster (steps / group / delete), also mounted by the standalone `KeyActionEditorMenu.ahk`.
- Device binding: `bindings.json` + `BindingResolver.ahk` (`WriteBinding`, `ResolveBinding`, scope=global|context, layers). Set-macro-wizard picker `08` = bind a key-action (pick/create → `_VWPickScope` scope pick → `WriteBinding fn:<name>`, or `streamdeck.py replace-at` for SD). ContextPassthrough gates keyboard keys per-context.

---

## 1. Disable (not delete) a key-action  — temporary off switch

**Ask:** "be able to disable a key… in either of the user interfaces an option to just temporarily disable it rather than delete it."

**Approach:**
- Add `"disabled": true` to the key_actions.json entry (default false/absent = enabled).
- **Generator (`gen_key_actions.py`):** when disabled, still EMIT the function (so bound/voice/SD callers still resolve — never "function not found"), but as a **no-op body** with a comment (`; DISABLED`). Keep it in `fn_to_file_index.json` + sidecar (so it stays visible in show fun) but tag the sidecar `"disabled": true` and add a badge to the summary. Rationale: a disabled key-action bound to a key should do NOTHING, not crash the dispatcher.
- **CLI:** `gen_key_actions.py disable <name>` / `enable <name>` (or reuse `set <name> disabled true`).
- **UI (portable cluster `_KeyActionNodes`):** an **Enable/Disable toggle row** (label reflects state, e.g. "Disabled — Enter to enable" / "Enabled — Enter to disable"). Shows in BOTH show fun + standalone automatically (one cluster). Mark disabled actions in the list detail (e.g. a `⊘` prefix / "(disabled)").

**Where:** `Scripts/gen_key_actions.py` (emit no-op when disabled; index/sidecar tag), `Helpers/KeyActionEditorMenu.ahk` `_KeyActionNodes` (+ `_KaNode*` handlers, mirror `_KaNodeSetGroup`), `_KaStepsStr`/list detail for the badge.

**Recommend:** no-op-body approach (safest for callers) + toggle row in the cluster + list badge.

---

## 2. Hold-to-repeat RATE for a device-bound key-action  — the YouTube right-arrow → `L` case

**Ask:** rebinding **right arrow → press `L`** on YouTube (forward 10 s in the video). Wants the **repeat rate to match the original** — "currently it is much slower… we might have to just build like a little thing that certain keys use."

**Investigate FIRST (READ_THE_LOGS rule):** measure WHY it's slow before building. Candidates:
- The binding fires once per physical keydown; held-key repeat depends on OS auto-repeat generating repeated keydowns that the `#HotIf` gate re-fires. If AHK's hotkey isn't seeing auto-repeats (or `#MaxThreadsPerHotkey`/`~`/`*` modifiers swallow them), it fires slowly.
- Per-fire overhead: does binding-dispatch send `l` with delay (SetKeyDelay) or spawn anything? A key-action bound as `fn:Name` runs in the always-on process (no MAINFUN spawn), so it should be fast — confirm.
- The interception itself (ContextPassthrough) may debounce or not pass repeats.

**Approach (the "little thing certain keys use"):** there is already `Helpers/HoldRepeatKeys.ahk` (foot-pedal hold-repeat) — the existing mechanism. Give a **binding (or key-action) an optional `repeat` config** `{initial_delay_ms, interval_ms}`; when the bound key is HELD, fire on a `SetTimer` loop at `interval_ms` until release (like HoldRepeatKeys), instead of relying on OS auto-repeat. Rate becomes configurable per key.

**Where:** `Helpers/HoldRepeatKeys.ahk` (reuse/extend), the keyboard binding path (`Helpers/KeyboardDispatcher.ahk` / `ContextPassthrough.ahk`), `bindings.json` record (add `repeat`), and the binding-edit UI (item 3's per-key Miller is the natural home for a "Repeat: off / rate…" option). Also relevant: `VideoWatching.ahk` (existing video-seek helpers) — the `L` press may have a dedicated function already.

**Recommend:** measure current slowness first; then a `repeat` option on the binding, powered by a HoldRepeatKeys-style timer loop, surfaced in the per-key Miller.

**UPDATE (2026-07-04) — root cause found + fixed (part a).** The slowness was NOT the repeat mechanism — it was that a key-action bound to a keyboard key spawned a fresh `MAINFUN.bat` process on EVERY held-key repeat. Confirmed in `ahk_event.jsonl`: `DISPATCH/err "function not found" YoutubeSkipForwardTen` → `Keyboard/Fallback: spawning MAINFUN` → a new pid per repeat (31888→4052→24256→46944). The always-on device processes (via `_CuratedClosure.ahk`) didn't include the generated key-actions or `KeyActionFunctions.ahk`, so `Keyboard_RunAction`'s in-process `Dispatch()` missed and fell back to process-spawn (~hundreds of ms) instead of the 15ms in-process call. **Fix:** added `KeyActionFunctions.ahk` + `#Include *i Generated/_KeyActionsAll.ahk` to `_CuratedClosure.ahk`, and made the aggregator's group-file includes `%A_LineFile%`-relative (not `%A_ScriptDir%`, which is the entry-script dir = `AlwaysOn\` for those processes). Now key-actions fire in-process on every device → near-native repeat. Deployed via ReloadWithNotice. **Part (b) — the configurable rate system (presets + custom) — still to build** (Jamie wants per-key rates regardless, to go faster/slower than native).

---

## 3. "Only when NOT in a text box"  — per-binding guard, an option in every key's Miller

**Ask:** "set certain hotkeys like the right arrow one to only be active when I am NOT in a text box… an extra option in the Miller of every single key."

**Approach:** a per-binding **guard condition** evaluated at press time. Detect "in a text box":
- AHK: focused control class check (`Edit`, `RichEdit20*`, `Scintilla`, browser content-editable is harder) via `ControlGetFocus` / `ControlGetClassNN`; OR
- UIA: is a control with the **Text/Value pattern + keyboard-focusable + editable** focused (more robust for web inputs). There may be reusable logic in the **Dragon typing-gate** path (`ContextPassthrough.ahk` already distinguishes physical vs synthetic + Dragon-safe typing — check `Keyboard_DispatchPhysicalOnly` and any focus/edit detection there).
- Cache the check (it runs per press) — keep it cheap.

Model it as a binding **guard flag** (e.g. `"guard": "not_in_textbox"`), checked in `ResolveBinding`/the dispatcher before firing (miss if guard fails, so the key passes through natively into the text box). Surface as a toggle row in the per-key binding Miller: "Active: everywhere / not in a text box."

**Where:** `Helpers/BindingResolver.ahk` (guard eval in resolve, or in each dispatcher post-resolve), `Helpers/ContextPassthrough.ahk` (focus/edit detection, Dragon-safe reuse), `bindings.json` (add `guard`), per-key binding Miller (item 5 / the wizard's per-slot options).

**Recommend:** a `guard: not_in_textbox` binding flag + a cheap focused-control/UIA edit check (reuse Dragon-gate detection if present); toggle row in the per-key Miller.

**UPDATE (2026-07-04) — BUILT.** Reused the site-stepper's UIA focus check by extracting it into a shared **`IsFocusInTextField()`** (CommonFunctions.ahk: `UIA.GetFocusedElement().Type` is Edit/ComboBox — works for web inputs too); `_SiteStepInTextField` now delegates to it. Binding record gains an optional **`guard: "not_in_textbox"`**; `ResolveBinding` Pass 1 skips a guarded record while focus is in a text field (computed at most once per call, only when a guarded record is hit), so the key falls through to a less-specific binding or misses → keyboard's miss = native passthrough (the char types normally). **`GetBindingGuard`/`SetBindingGuard`** in BindingResolver.ahk. Surfaced as a per-hotkey toggle in the wizard Miller's current-binding options ("Active: everywhere ↔ only when NOT in a text box"), non-SD only. Deployed. **Needs Jamie's live test:** guard the YouTube right-arrow→L, then focus the YouTube search box and press Right — it should move the cursor in the box, not skip the video.

---

## 4. BUG: Stream-Deck create-then-bind makes the key-action but never PLACES the button

**Ask:** "the stream deck button creating is still not working — manages to create the [key-action] but never actually adds the button, so I have to go find it in show fun to actually assign it."

**Symptom:** wizard on a blank SD button → `08` → "+ New key-action, then bind" (`_KaCreateThenBind` → `_KaBindNow` SD branch → `streamdeck.py replace-at deck page pos <fn>`). The key-action gets created (store + regen) but the **SD button is not updated**.

**Likely causes (investigate — read the log at failure):**
- `replace-at` **modifies an EXISTING button in place**. A **blank/passthrough** slot may have no real button to replace, or its `Settings.path` is the passthrough (`SetMacroWizardSdForSlot`), so replace-at either no-ops or targets wrong. → For a blank slot, use **`streamdeck.py add … --force`** (which we built) instead of `replace-at`, OR make `replace-at` upsert.
- Page/pos parsing from the SD slot string (`deck|page|pos[#sub]`) may be off (the `#sub` handling), so replace-at targets a nonexistent action.
- `RunWait replace-at` may be failing silently (exit code not checked in `_KaBindNow`).

**Fix plan:** reproduce with `ahk_event.jsonl` tailing at the moment; log the exact `replace-at` command + its exit; if the slot is blank/passthrough, branch to `add --force`; check exit code and tooltip on failure. Compare with the WORKING SD path `_Smw_SaveSdFromPicker` (it does button-info + replace-at + icon picker + `--restart`) — the create-then-bind SD path is a stripped copy and likely missing a step (e.g. `--restart`, or the button-must-exist assumption).

**Where:** `Helpers/KeyActionEditorMenu.ahk` `_KaBindNow` (SD branch), compare `Helpers/SetMacroWizardFunctions.ahk` `_Smw_SaveSdFromPicker`, `~/.claude/helpers/streamdeck.py` `replace-at` vs `add --force`.

**Recommend:** for blank/passthrough SD slots use `add --force`; check + surface the exit code; add `--restart`. Verify live.

---

## 5. Unify the wizard's function-finder with show fun's backend

**Ask:** "inside the wizard it should use the exact same backend for finding functions as show fun does. Currently it only shows 10 and there is no search beyond that. Show fun has search and everything built-in. Ideally a unified backend so I can change them both by changing one."

**Current state:** the set-macro-wizard picker is `_PersistentLoopPickGui` fed by `_Smw_RecentDispatches` (≤10 recent dispatches, no full search). Show fun is `_MillerColumnPickGui` fed by `fn_to_file_index.json` via `_VceAllFunctions()` + recursive Miller search (every function, typeable).

**Approach (per UNIFIED_MILLERS):** the wizard's "pick a function to bind this key" should reuse show fun's function browser/backend. Options:
- **(a) Shared provider:** extract the function-list + search source (`_VceAllFunctions` / index read + recents merge) into ONE provider both call. The wizard keeps its picker engine but pulls the full catalog + search from the shared provider. Smaller change; unifies DATA not UI.
- **(b) Reuse the Miller:** the wizard opens show fun (or a show-fun-derived Miller) in a "pick a function to bind <device/slot>" mode — Enter binds `fn:<name>` at the chosen scope (`_VWPickScope`) instead of assigning voice. Deletes the wizard's bespoke catalog entirely; full search for free; ONE backend. This is the UNIFIED_MILLERS-correct answer (the wizard becomes an entry point into the function Miller). Mirrors how `08` already routes key-action binding into a Miller.
- Recents stay pinned on top (both already have a recents concept).

**Where:** `Helpers/SetMacroWizardFunctions.ahk` (`_SetMacroWizard_RunPicker`, catalog build ~line 470-500), `Helpers/VoiceCommandEditorMenu.ahk` (`_VceAllFunctions`, root builder, recents merge), `Helpers/BindingResolver.ahk` (`WriteBinding`), `Helpers/Gui/VoiceWizardAssign.ahk` (`_VWPickScope`).

**Recommend:** **(b)** — route the wizard's function-pick through the show-fun function Miller (bind mode: pick fn → `_VWPickScope` → `WriteBinding`/SD replace-at). One backend, full search, unified by construction. Bigger but it's the right architecture and kills the "only 10, no search" gap permanently.

---

## Build order (suggested, post-compact)
1. ✅ **DONE (2026-07-04) — #4 SD create-then-bind bug.** Root cause was NOT `replace-at` vs `add` — it was a dropped arg: `_Smw_BindKeyActionFromPicker`'s hand-rolled `Run(Format('"{}" …8 slots…', …9 values…))` silently dropped the trailing `sdTarget`, so GuiHost got 5 args, `sdTarget` defaulted to `"none"`, and `_KaBindNow` took the **non-SD** branch → wrote a `bindings.json` scope=global entry instead of running `replace-at` on the SD manifest (confirmed in `ahk_event.jsonl`: `KeyActionEditor/bind` not `/bind-sd`). Fix: build the GuiHost command line variadically (mirrors `_RunInGuiHost`) so the count can't drift; and `_KaBindNow`'s SD branch now captures stdout+stderr and CHECKS the exit code (was fire-and-forget, always reported success). Deployed via `ReloadWithNotice`. **Needs Jamie's live confirmation** (press SD button in set-macro mode → 08 → "+ New key-action, then bind").
   - **Follow-ups (2026-07-04, same session):** (a) the SD create-then-bind was routing through a bespoke stripped-down `replace-at` that skipped the icon picker. Extracted the canonical SD-assign flow into **`_Smw_AssignSdSlot(slot, fn, sourceSrc)`** (button-info → icon picker → `replace-at --icon --restart` → exit-check → history → tooltip); BOTH `_Smw_SaveSdFromPicker` (normal fn assign) and `_KaBindNow` (key-action bind) now call it, so binding a key-action to an SD button uses the EXACT same UI as any other function (icon picker included). `_KaBindNow` closes the Miller first, then delegates. (b) The create form now auto-converts a typed name to PascalCase (`_KaToPascal`: "open new tab" → `OpenNewTab`, collapses ALL-CAPS, preserves intentional mixed case, prefixes `Ka` if it would start with a digit) with a live "will create: …()" preview — no more "invalid function name" failures.
2. ✅ **DONE (2026-07-04) — #1 Disable toggle.** `disabled` flag in the store; `gen_key_actions.py` emits a **no-op body** when disabled (callers still resolve) + badges the sidecar summary/tags + keeps the fn-index entry; new `enable`/`disable` CLI subcommands; portable cluster `_KeyActionNodes` gained an Enable/Disable toggle row (shows in BOTH show fun + standalone) + `[DISABLED]` markers in list details. Fully live (Python + GuiHost-fresh). No reload needed.
3. ✅ **DONE (2026-07-04) — #5, via a full wizard→Miller conversion (bigger than the original #5).** Jamie decided the set-macro wizard should just *be* show fun with a couple extra rows, killing the `_PersistentLoopPickGui` picker's "only 10, no search" limit AND the redundant numpad-scope wall in one move. Built the **hybrid** architecture (shared data, thin per-mode layer):
   - **Shared function list:** extracted `_VceFunctionBranches(makeOptions)` in VoiceCommandEditorMenu.ahk — the ONE builder both show fun and the wizard use (recents-pinned + every function, tags, search). Can't drift.
   - **The wizard Miller** (`Helpers/MacroWizardMiller.ahk`, GuiHost-only): Row 1 `This slot` (current+previous bindings → remove/revert, reusing `_Smw_RightPaneRows`/`_Smw_RemoveBindingFromPicker`), Row 2 `Wizard tools` (modes/trace/add-context/create-key-action), Rows 3+ every function → **pick fn → `_VWPickScope` (global/current/parent, like voice commands) → bind**. SD slots route through `_Smw_AssignSdSlot` (icon picker). Kills the `2.03`-to-pick-context clunk.
   - **Split for the include graph:** only the tiny launcher `_SetMacroWizard_RunMiller` (shells GuiHost) stays in SetMacroWizardFunctions.ahk (always-on closure); all builders live in MacroWizardMiller.ahk (MAINFUNCTIONS/GuiHost only). Intercept (`SetMacroWizard_HandleIntercept`) now calls the launcher. Old `_PersistentLoopPickGui` picker kept **dormant** for instant revert — remove once proven.
   - Also fixed a latent crash: `GuiHotkeys.ahk`/`GuiLayout.ahk` hard-depended on `Cfg` (unassigned in standalone GUI hosts like SoundLibraryPicker) → now derive their base path from `A_LineFile` with a `Cfg` fast-path.
   - **Deferred:** an advanced "bind with specific layers" per-function option (Miller currently binds with press-time filtered layers); add-context degrades if the press-time window is gone. **Needs Jamie's live test** of the drill-in flows.
4. **#3 not-in-textbox guard** + **#2 hold-repeat rate** — the two binding-layer features; do together since both add per-binding options to the same per-key Miller (which #5/#3 imply building). Measure #2's slowness first. **UI-touching — confirm approach with Jamie first.**
