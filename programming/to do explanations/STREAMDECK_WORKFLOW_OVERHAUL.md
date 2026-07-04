---
tags: [autohotkey, streamdeck, caster, macro-system, context, design, workflow]
created: 2026-06-30
status: design
owner: Jamie
related: [ALWAYS_ON_CONTEXT_DETECTOR]
---

# Stream Deck Workflow Overhaul — design

**Goal (Jamie's words):** make creating a new Stream Deck profile and wiring it into a context *much less work and much more automatic* — "very easily whip up a new profile for a context in 10–15 minutes and then it immediately pays off." The context auto-switch system ([[ALWAYS_ON_CONTEXT_DETECTOR]] Consumer 2) is the payoff engine; this doc is about making the **content** (profiles + buttons) cheap to produce.

Legend: **[NOW]** actively building / next · **[SOON]** high-value, do after NOW · **[FUTURE]** captured, not scheduled.

---

## 1. Auto-reclaiming switcher — **[DONE 2026-06-30]**

The VSD "PROFILE SWITCHER" is an **auto-managed set**, not a hand-curated palette. `streamdeck.py vsd-sync` rebuilds it to *exactly* the profiles referenced by contexts (`streamdeck_profile` / `streamdeck_profile_b`): adds missing (DeviceUUID = the profile's own deck), reclaims unreferenced, re-packs gap-free, reuses existing buttons. **No-op (no restart) when unchanged.** The Context Manager picker calls `vsd-sync` after every mapping edit, so the switcher stays minimal automatically; profiles auto-add back the moment a context maps to them. `vsd-gc` is an alias. (Decision: reclaim-on-every-change is free because it piggybacks on the write+restart adding already costs, and no-ops otherwise.)

---

## 2. Profile templates — **[SOON]**

Several **templates**, not one, so a new profile starts pre-populated for its *kind*:
- **Chrome-tab template** — the recurring Chrome-page layout (pinned switchers + a standard action zone).
- Others as patterns emerge (messaging/list-nav page, media page, …).

Mechanics (much already exists — `copy-profile` + the `TEMPLATE`/`GECK2 Home` sources):
- Generalize `copy-profile` into a **template picker**: choose a template, name the new profile, done. Templates **prioritized/pinned at the top** of the picker UI.
- Templates are just profiles tagged as templates (naming convention or a marker file), so making a new template = build a profile + mark it.
- Output: a new profile in seconds, ready to customize — then wire it to a context via the Stream Deck facet (§1).

Open Qs: where does the template list live (naming prefix vs a manifest)? Deck A only, or template-per-deck?

---

## 3. Button groups — **[SOON]**

Reusable **named groups of buttons** added/removed as a unit at an anchor position:
- **list-nav-8** — the 8-button bottom-right block on the Chrome page (the recurring list-navigation cluster).
- **scroll-pair** — the extra scroll buttons.
- …grows as clusters recur.

Mechanics:
- A group = a JSON spec (list of relative-position buttons: label/icon/action). Store in `INIDATA/sd_button_groups/*.json` (or one file).
- `streamdeck.py add-group <profile> <group> <anchor>` stamps the group's buttons at an anchor (respecting pinned slots, blocking conflicts).
- Groups compose with templates: a template can *declare* groups it includes, so "Chrome-tab template" = base + `list-nav-8` + `scroll-pair` auto-placed.

This is what makes §2 powerful — a template is mostly "base layout + a few groups."

---

## 4. JSON-defined key-combo actions — **[V1 BUILT 2026-06-30]**

> **Full design + as-built: [[KEY_ACTION_FUNCTIONS]]** — the dedicated doc (schema, Caster-syntax DSL, generator, runtime, future-proofing). V1 shipped: store `INIDATA/key_actions.json` + `py Scripts/gen_key_actions.py` (add/list/remove) → real functions in `Helpers/Generated/KeyActions_*.ahk`, runtime `Helpers/KeyActionFunctions.ahk`. Still [SOON]: bindings reconcile (the SD-button "bind to key-action" default + `send:` migration that closes out the broken capture flow). Summary below.


**Problem:** the current "add a keyboard-shortcut button" flow (set-macro-wizard *capture hotkey* → saves `send:<captured>` at scope=global) is **broken in practice** — key capture is unreliable, hand-typing the combo + Enter often does nothing, and dispatch is flaky. Jamie has never gotten it to work.

**Vision:** define a key combination as a **named AHK action purely in JSON**, reusable by:
- **Stream Deck buttons** (`MAINFUN.bat RunKeyAction chrome_new_tab`),
- **voice commands** (phrase → same `RunKeyAction("chrome_new_tab")`),
- the **binding system** (Q0/pedals/CapsLock/keyboard),

so **one definition, many callers** — change the combo once and it changes everywhere.

Design sketch:
- Registry `INIDATA/key_actions.json`: `{ "chrome_new_tab": {"keys": "^t", "desc": "New tab", "method": "send"} , ... }`. `method` picks the reliable send path (plain `SendInput`, or the DisplayFusion/Playback route for Win-key combos that `Send("#…")` silently drops — see the "Win key gotcha").
- Dispatcher `RunKeyAction(name)` in a new `Helpers/KeyActionFunctions.ahk`: look up → send via the right method → log. Fixes the reliability problem in ONE place.
- Authoring: a clean add-flow (GUI or `key-action add <name> <combo> <desc>`) — and critically a **reliable combo capture** (the current capture is the weak point; consider hand-type-first with a live-preview + validate, since capture-via-keydown fights Dragon/AHK hooks).
- Reconcile with existing `bindings.json` `send:` actions — either migrate them to named key-actions or make `send:<name>` resolve through this registry. The macro/binding system ([[ALWAYS_ON_CONTEXT_DETECTOR]], `references/macro-binding-system.md`) is the integration point.

**Why it's the big one:** it collapses "SD button that does Ctrl+T" + "voice command that does Ctrl+T" into one durable, editable thing — the exact "define once, reuse everywhere, change everywhere" Jamie called out. Most Chrome-page buttons are shortcut buttons, so this unblocks fast profile authoring.

Sub-item **[NOW-ish]**: the current capture/dispatch is actively broken — worth an early reliability fix even before the full registry, since it blocks everything else.

---

## 5. `streamdeck.py` refactor? — **assessment: [DEFER]**

~3000 lines, one file, ~27 subcommands. **Is it necessary? No. How big? Medium + risky.**
- It's *large* but *organized*: clear function sections (manifest IO, passthrough, button builders, VSD, icons, per-command `cmd_*`, `COMMANDS` dict). A well-sectioned single-file CLI at 3k lines is maintainable.
- A split into a package (`streamdeck/{manifest_io,buttons,vsd,icons,cli}.py`) is a few hours of careful work with **real regression risk** on a tool Jamie depends on daily (and which just had two live incidents). Payoff is developer-ergonomics, not user-facing.
- **The 15-minute-profile goal comes from the FEATURES (§2–4), not from refactoring.** So: don't refactor as a project. Do **opportunistic extraction** — when the VSD code (§1) or groups (§3) grows, lift that cluster into its own module. Let structure emerge from the feature work.

Recommendation: **skip the refactor**; revisit only if a feature is genuinely hard to add because of the file size (it isn't yet).

---

## 6. End-to-end target workflow ("new profile in 10–15 min")

1. **Create** — template picker (§2): pick "Chrome-tab template", name it "Instagram" → profile exists, base layout + declared groups (§3) placed.
2. **Customize** — add a couple of key-action buttons (§4) from the reusable registry; tweak icons.
3. **Wire** — Context Manager → the Instagram context → Stream Deck facet → pick "Instagram" (Deck A) → `vsd-sync` auto-adds the switcher button; done (§1 + [[ALWAYS_ON_CONTEXT_DETECTOR]]).
4. It **immediately pays off**: switching to an Instagram tab auto-switches the deck.

Each piece removes a manual step; together they hit the 15-minute target.

---

## Build order (proposed)
1. ~~Auto-reclaim switcher~~ **DONE.**
2. **Fix key-combo capture/dispatch reliability** (§4 sub-item) — it's actively broken and blocks button authoring.
3. **JSON key-action registry + `RunKeyAction`** (§4) — the multiplier; reconcile with bindings.
4. **Button groups** (§3) — encode the recurring clusters.
5. **Template picker** (§2) — tie it together for one-step profile creation.
6. Refactor: **deferred** (§5) — opportunistic only.
