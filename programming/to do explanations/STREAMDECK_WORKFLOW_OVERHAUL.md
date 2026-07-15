---
tags: [autohotkey, streamdeck, caster, macro-system, context, design, workflow, inheritance, ledger]
created: 2026-06-30
status: design
owner: Jamie
related: [ALWAYS_ON_CONTEXT_DETECTOR, KEY_ACTION_FUNCTIONS]
---

# Stream Deck Workflow Overhaul — design

**Goal (Jamie's words):** make creating a new Stream Deck profile and wiring it into a context *much less work and much more automatic* — "very easily whip up a new profile for a context in 10–15 minutes and then it immediately pays off." The context auto-switch system ([[ALWAYS_ON_CONTEXT_DETECTOR]] Consumer 2) is the payoff engine; this doc is about making the **content** (profiles + buttons) cheap to produce.

Legend: **[NOW]** actively building / next · **[SOON]** high-value, do after NOW · **[FUTURE]** captured, not scheduled.

---

## 1. Auto-reclaiming switcher — **[DONE 2026-06-30]**

The VSD "PROFILE SWITCHER" is an **auto-managed set**, not a hand-curated palette. `streamdeck.py vsd-sync` rebuilds it to *exactly* the profiles referenced by contexts (`streamdeck_profile` / `streamdeck_profile_b`): adds missing (DeviceUUID = the profile's own deck), reclaims unreferenced, re-packs gap-free, reuses existing buttons. **No-op (no restart) when unchanged.** The Context Manager picker calls `vsd-sync` after every mapping edit, so the switcher stays minimal automatically; profiles auto-add back the moment a context maps to them. `vsd-gc` is an alias. (Decision: reclaim-on-every-change is free because it piggybacks on the write+restart adding already costs, and no-ops otherwise.)

---

## 2. Profile templates — **[SOON]** · **reframed by §8**

> **Superseded framing:** with the ledger model (§8), a "template" is just a **node in the profile tree** whose children inherit its layout. "Copy the Chrome template" becomes "instantiate a child of the Chrome node," which auto-inherits its groups and enrolls the new profile in the Chrome family. Read §8 first; the picker below is still the entry UX.

Several **templates**, not one, so a new profile starts pre-populated for its *kind*:
- **Chrome-tab template** — the recurring Chrome-page layout (pinned switchers + a standard action zone).
- Others as patterns emerge (messaging/list-nav page, media page, …).

Mechanics (much already exists — `copy-profile` + the `TEMPLATE`/`GECK2 Home` sources):
- Generalize `copy-profile` into a **template picker**: choose a template, name the new profile, done. Templates **prioritized/pinned at the top** of the picker UI.
- Templates are just profiles tagged as templates (naming convention or a marker file), so making a new template = build a profile + mark it.
- Output: a new profile in seconds, ready to customize — then wire it to a context via the Stream Deck facet (§1).

Open Qs: where does the template list live (naming prefix vs a manifest)? Deck A only, or template-per-deck?

---

## 3. Button groups — **[SOON]** · **reframed by §8**

> **Superseded framing:** groups become **first-class ledger objects** with many-to-many membership (a button can belong to several groups — §8 solves the multi-group case a per-button marker couldn't). A group attaches to a **tree node** (inherited by its subtree) or a **flat tag** (cross-tree cohort). The relative-anchor mechanics below still hold for *anchored* clusters; *shared* sets use absolute preferred positions. Read §8 first.

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

## 7. Propagate an edit to a button's clones/relatives — **[DONE 2026-07-12]**

**Ask (Jamie):** "recognize identical buttons + identical key-logic buttons; when I replace one, offer to replace them all with what I just did — but not always, so it's just an option." E.g. the lockout Key-Logic button lives on many profiles across both decks; usually an edit should mirror to all of them.

**Reality check (from her real decks):** the 8 lockout buttons are **NOT identical — they've drifted.** By content they form 5 groups: minutes differ (1 vs 5), long-press differs (15 vs ResumeLockout), and the icon has 2 content-variants (with 3 different *filenames* — UI-set icons get hashed names, so **filename ≠ identity; content hash is**). Strict "identical" matching finds ~nothing. So the feature gathers by a **drift-tolerant** notion of "the same button."

**Design decided:**
- **Trigger = opt-in, not automatic.** Most edits should NOT propagate, so nothing pops up after an edit. It's one extra row in the macro-wizard's SD-button drill ("Propagate this button to matching buttons…"). The edit UI is otherwise unchanged.
- **Two tiers** (Jamie: tiered gather): **exact** = content-identical clones (pre-checked; re-push is a no-op) + **related** = same *primary function* (short-press fn name, args-agnostic) — the family key that survives drift (unchecked, opt-in). Toggle-list shows each candidate's current action summary so the drift is visible while choosing.
- **Whole-button converge** (Jamie: converge; the drift was accidental): propagate overwrites each checked button with the *entire* source button (icon/label/all KL slots). Intentional exceptions are protected by simply not checking them.
- **Content-based icon identity** (hash of the icon bytes) so the same picture counts as equal across its hashed/pack-named copies.

**Where it lives:**
- **Engine — `~/.claude/helpers/streamdeck.py`.** Per Jamie's "don't tack onto crappy code": the three ad-hoc button walkers (`_iter_hotkey_buttons`, `_walk_open_buttons`, and a new one) were **unified into one `ButtonView` value object + `iter_buttons`/`find_buttons`** — every finder (`find-hotkey`, `find-fn-icons`, and the new ones) now runs through it. New commands: `button-signature` (whole-button identity), `find-matching` (exact by signature), **`find-siblings`** (exact + related tiers in one call — what the wizard uses), `propagate` (deep-copy the source button onto target positions, fresh ActionIDs, `safe_write` re-stamps `--src`, one page-write per touched page, one restart; `--targets-file` for AHK handoff, `--dry-run` to preview the plan).
- **Wizard — `Helpers/MacroWizardMiller.ahk`** (`_MwPropagateNode` / `_MwPropagateChildren` / `_MwPropDoPropagate`): SD-only row → toggle-list (exact pre-checked, related opt-in, in-place ☑/☐ refresh) → `propagate --targets-file --restart`.
- **Verified:** engine dry-run end-to-end on the real lockout button (source → check all 7 related → plan overwrites exactly those 7 across 7 profiles, no writes); AHK validate + include-closure clean; toggle-list screenshotted (all 7 relatives with drift summaries).

**This partially delivers §3 (button groups)** — "primary-function family" is a *derived* group with zero bookkeeping. A future explicit/persistent group tag (survives even a fn rename) is still the heavier option if derived grouping proves too loose.

**GOTCHA burned in (2026-07-12): a whole-button copy MUST copy the icon FILE too.** A button's `States[0].Image` is a **relative** path (`Images/<name>.svg`) resolved inside *each profile's own* `Images/` folder. The first propagate copied only the button JSON, so all 7 targets pointed at a file that lived only in the source profile → **blank icons on 6 of them.** Fix: `propagate` now calls `_propagate_icon` (→ `embed_icon`) to copy the source icon into each target profile's `Images/` and rewrite the clone's `Image` to the new per-profile path (+ carries provenance). Same trap as the A→B copy's `shutil.copytree` (which copies Images/ wholesale — that's why it doesn't hit this). Any future "move/clone a button across profiles" path must copy the icon file, not just the manifest entry.

**Notes / possible follow-ups:** (a) propagate always `--restart`s (one per confirm) — fine for an occasional action; batch-on-close only if it grates. (b) Deck B is rebuilt by the A→B copy anyway, so propagating to B is a "do it now" convenience; the durable value is across Deck A's own profiles. (c) A standalone "sync this button" command outside the wizard was **declined for v1** (parked). (d) The `ButtonView` unification is exactly the "opportunistic extraction" §5 endorses — done in-file, no package split.

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

> **Note:** the build order above predates §8. The **§8 ledger model is now the spine** — templates (§2), button groups (§3), and propagate (§7) all become facets of it. See §8's own build order at the end.

---

## 8. Ledger model — profile tree, groups, render — **[DESIGN 2026-07-14]**

The big architectural decision, from a design conversation with Jamie (2026-07-14). Goal in her words: many near-identical Chrome sub-profiles that are ~75% shared + a few unique buttons, where she can **update the shared parts across a whole cohort at once**, unique buttons are protected, and **nothing she's already placed ever moves without her say-so**. Jamie chose to **refactor properly rather than bolt onto the existing copy/propagate paths.**

### 8.1 Why a ledger (not per-button markers)

Three requirements that emerged in discussion each break a per-button-marker scheme and each point at an external source of truth:

- **Multi-group membership** — a button (e.g. `scroll-up`) can belong to several groups. A scalar `_grp` marker can't express many-to-many, and "which group's preferred slot wins?" has no home on a leaf node. That's relational data; it belongs in a ledger.
- **Parent inheritance** — profiles form a tree (`chrome` → `chrome-nav` → `Instagram DMs`); content cascades down, child overrides win. That's a relation between profiles, not a property of a button.
- **No drift** — Jamie wants a ledger "so deeply integrated/derived it's impossible to drift."

**The anti-drift construction:** the ledger is the **single source of truth** for all *structure*; **manifests are a pure render of it.** You never read structure back out of a manifest, so there is nothing to drift *from*. The **only** thing read back from a manifest is "did Jamie manually override this slot," detected via an opaque **stable ID** (`_mid`) that render stamps into each managed button's `Settings`. The `_mid` is a *render artifact / breadcrumb*, **not** the source of truth — which is what keeps the §7 anti-drift lesson intact (identity comes from the authoritative ledger, not a fragile external filename map).

### 8.2 Core objects (all in the ledger — `INIDATA/sd_ledger.json`)

- **Profile tree** — **single parent** per node (v1). A node is a profile (or an abstract template node with no device of its own). Effective content of a node = its own groups/buttons **unioned up the parent chain**, child wins on conflict. Spans **both decks** (each node records which physical deck + device it renders to).
- **Family = a subtree.** No separate family registry — addressing a node addresses its whole descendant set. Push a button to `chrome` → lands on chrome and everything under it; push to `chrome-nav` → just that subtree. This is the cohort-scoping Jamie wanted, for free.
- **Flat tags** — orthogonal to the tree, for **cross-tree cohorts** (e.g. several `GECK2*` profiles under different parents that all want the chrome list-nav group). A group can attach to a **node** (inherited by subtree) *or* a **tag** (any tagged profile).
- **Groups** — named button sets, **many-to-many** with buttons. Each group-button has a *preferred position*. Anchored clusters (relative to an anchor) and shared sets (absolute preferred pos) are the same structure with/without an anchor param.
- **Per-profile resolved positions** — where each managed button actually landed on each profile, **frozen** (see 8.4).
- **Per-profile deprecations** — group-slots Jamie removed on a specific profile (see 8.4).

### 8.3 The three operations

- **`render`** (ledger → manifests): the only writer of managed content. Pure projection. Stamps `_mid` per managed button. Handles per-node device targeting, `AppIdentifier` strip for Deck B mirrors, fresh page UUIDs at node creation. **Additive:** never moves a frozen button; new buttons drop into free slots by the deterministic fill order; a placement that would require displacing an existing group **asks** rather than reflowing silently. Internally calls the §7 `propagate` write primitive (deep-copy, fresh ActionIDs, **the icon-file copy** — the gotcha §7 already burned in).
- **`reconcile`** (manifests → ledger): the drift-closer. Scans for manual overrides — a managed slot whose `_mid` is missing or whose content diverged → records a **deprecation**; a new hand-placed real button (no `_mid`) → a protected **unique**. Run **before every apply** so raw-Stream-Deck-UI edits are safe.
- **`apply`** = reconcile → render for a target node/tag (a cohort).

### 8.4 Placement rules (Jamie's muscle-memory constraints)

- **Deterministic first placement, then frozen.** First time a managed button lands, its slot is chosen by the fill order (existing zone priority: primary cols 4-7 rows 0-1 → nav-right → utility-left) into a free slot, then **written to the ledger and never recomputed.** Re-applying a group is purely additive — existing buttons do not move.
- **Group-level relocation is an explicit, prompted choice.** If a new group can't fit without displacing an existing group, the system **asks**; it never silently shuffles muscle memory.
- **Manual overwrite = deprecation, not eviction.** Jamie's list-nav case: she stamps 8 buttons, later overwrites positions 6/7/8 because she only wants 5. Result: those three group-slots are marked **deprecated on that profile** and **never re-placed**; the remaining 5 stay frozen where they are. The profile "accepts it has 5 of 8." **Re-adding the group intentionally clears that profile's deprecations** and re-places the missing ones (into gaps; asks if that needs to move anything).
- **Unique = a real button with no `_mid`.** No separate "preserved slots" list. Untagged real buttons are protected by definition, so existing hand-built profiles need zero migration — untagged = safe default.

### 8.5 Deck B, unified into the tree (retires the wholesale A→B copy)

- **Mirror profiles** (most of B) = **children of their A counterpart.** They inherit A's content and **cascade automatically when A updates.** Because render is incremental per-node, B children can carry **their own overrides that survive** — this **fixes the current copy's known bug** ("individual Deck B customizations are lost every copy").
- **`GECK2*` profiles** = **independent nodes**, no A parent (never overwritten), that opt into shared groups via **flat tags**.
- **The 8-step wholesale A→B copytree is retired.** Its mechanical steps (fresh page UUIDs, `AppIdentifier` strip, device retarget, switch-button remap) become **render finalizers** run at node creation, not on every bulk copy.
- **Migration:** one-time — enroll existing B mirrors as children of their A counterpart by name-match; tag existing managed buttons; capture groups from reference profiles.

### 8.6 What's kept vs reworked (per "refactor, don't bolt on")

- **Keep** `ButtonView` / `iter_buttons` / `find_buttons` (the good §7 foundation — render *and* reconcile sit on it).
- **Keep** `propagate`'s write internals, but **demote** it to render's private write primitive (not a user verb).
- **Rework** `copy-profile` → "instantiate a child node" (inherits parent chain + enrolls in tree/tags).
- **New extracted module** (`streamdeck/families.py` or `ledger.py`) for the ledger/render/reconcile core — this is finally the "opportunistic extraction" §5 was waiting for, scoped to just this feature.

### 8.7 Open risks / to verify before coding

1. **`_mid` persistence — VERIFIED 2026-07-14. ✅** An unknown key injected into a button's `Settings` **survives** Stream Deck's graceful-exit flush. Test: force-kill SD → inject `_mid_persist_test` into a real button on disk (VLC profile, `0,0`) → launch SD (loads into cache) → `StreamDeck.exe --quit` (graceful flush; manifest mtime confirmed changed, i.e. SD *did* rewrite the file) → read back: key present. So render can stamp `_mid` into `Settings` and reconcile can read it after SD's own rewrite — **no sidecar fallback needed.** (Test script parked in the session scratchpad; manifest was snapshotted + restored byte-for-byte.)
2. **Reconcile correctness** for raw-UI edits (hashed icon names, content-compare thresholds).
3. **Single vs multi parent** — chose **single** for v1; multi-inheritance (diamond conflicts) deferred.
4. **Scope** — page 0 per profile for v1.

### 8.8 Revised build order (supersedes §"Build order (proposed)")

1. **Verify `_mid` survives an SD rewrite** (8.7 #1) — gates everything.
2. **Ledger schema + `render`** for a single node (no inheritance yet) — prove ledger → manifest with frozen positions + `_mid` stamping.
3. **`reconcile`** — override/deprecation detection; makes edits safe.
4. **Parent inheritance + subtree cohorts** — union-up-the-chain, `apply` to a node's descendants.
5. **Groups as ledger objects** (many-to-many) + **flat tags** — `group capture` from a reference profile doubles as the tag-migration.
6. **Deck B as children of A** + retire the wholesale copy; migrate existing mirrors.
7. **Template picker** (§2) as the create-a-child UX; wire to context (§1).
8. Extract the module (8.6) as it grows — don't front-load the split.

### 8.9 As-built scaffold — **[2026-07-14]**

**Refactor sequencing decided: "during, incremental" (not before/after).** A big-bang
refactor of the 3000-line `streamdeck.py` as its own project is the §5 trap (regression
risk on a daily driver with two live incidents, zero user-facing payoff, reshapes code
the ledger will re-seam anyway). "After" entrenches the monolith as a dependency and the
refactor never happens. So: **build the ledger as a proper package now; extract from the
monolith only the primitives render/reconcile need (manifest IO → buttons → process),
one verified step at a time.** The monolith shrinks opportunistically afterward, no rush.

Two contracts preserved permanently: the CLI stays `py -3 streamdeck.py <cmd>`, and
`import streamdeck as sd` keeps working (the two `.claude/helpers` Python callers).

Package now on disk at `AutoHotkey/Scripts/StreamDeck/`:
- `sdlib/__init__.py` — package doc + extraction strategy.
- `sdlib/paths.py` — first extracted foundation: `AHK_DIR`, `PROFILES_DIR`, `INIDATA_DIR`,
  `LEDGER_PATH` (`INIDATA/sd_ledger.json`), and `FILL_ORDER` (deterministic first-placement order).
- `sdlib/ledger.py` — the §8 data model (`Node`/`Group`/`GroupButton`/`Tag`/`Ledger` dataclasses),
  `load_ledger`/`save_ledger`, and the **real** pure helpers (`parent_chain`, `children`,
  `descendants` = cohort, `effective_group_names` = inherited+tag groups child-wins). `render`
  and `reconcile` are **stubs** (raise until the manifest/buttons primitives are extracted).
- `streamdeck.py main()` delegates `ledger <sub>` to `sdlib.ledger.main` (one hook; monolith
  argparse untouched). Sub-CLI: `init｜list｜tree｜groups｜show｜effective｜render｜reconcile`.

Verified: `ruff` + `pytest` + `caster_ahk_verify` all PASS; back-compat CLI (`list`) and
`import streamdeck as sd` intact; `ledger tree/list/show/effective` work on an (empty) ledger.
`sd_ledger.json` not created yet — awaiting §8 review before populating the first real tree.

**Next (§8.8 step 2):** extract manifest IO (`_walk_profiles`/`_walk_pages`/`safe_write`) into
`sdlib/manifest.py`, then `ButtonView`/`iter_buttons` into `sdlib/buttons.py`, then implement
`render` for a single node on top of them.

### 8.10 Authoring & UX — how you actually create/edit — **[decided 2026-07-14]**

**Group source-of-truth = real (hidden) Stream Deck profiles.** Each group is an ordinary
Deck A/B profile holding ONLY its own buttons — no defaults, no chrome, nothing else — marked
as an authoring profile so it's excluded from context auto-switch. **The VSD is only the
switch mechanism, not storage.** To edit a group, the Miller has a macro invoke that group's
button on the VSD "PROFILE SWITCHER" — the same UIA-invoke that switches decks for everything
else (`vsd-ensure` adds the switcher button) — the physical deck flips to the group profile,
and Jamie edits it on the actual Stream Deck with the wizard: **the exact same backend as
editing any profile today.** She never hunts for these on a physical switcher by hand; the VSD
link is just how the Miller reaches them. Render reads buttons from the group profile like any
other and stamps them onto members (`propagate`'s cross-deck path already handles device-UUID +
icon-file copy).

**Groups are granular** (Jamie's split): `profile-switching-defaults` (top-left 4 buttons),
`other-defaults` (the 8 below/left), `navigate` (list-nav), `chrome`, … "Defaults" is not
special — just group(s). New profiles start with a `default` set checked; you can uncheck
even that.

**Three edit actions, all reusing existing tools:**
1. **A group's buttons** → Miller/wizard switches the VSD to that group profile; edit directly; re-render restamps members.
2. **A node's composition** (which groups / parent / tags) → ledger edit via the checklist or Miller.
3. **A node's unique buttons** → on the node's own profile; reconcile keeps them (no `_mid`).

**Two authoring UIs onto the one ledger:**
- **Wizard blank-slot menu (new).** Clicking a blank SD button offers, next to "add function",
  rows for **add group** (pick a group → stamp from this anchor) and **add/adopt template**
  (set parent → inherit its groups). Imperative "put this here", same muscle memory as adding
  functions today. (Screenshot of the current set-macro Miller is the surface being extended.)
- **Per-profile group checklist.** Reachable from "open context → add Stream Deck profile" AND
  re-openable for any profile anytime. Checkboxes of groups (and bundles/families). New profiles
  start with `default` pre-checked; auto-checks derive from the chosen parent/app (a chrome
  profile pre-checks default+chrome); unchecking an inherited group records a per-node
  **exclusion** (`Node.excluded`) so you can even drop the defaults. Toggle = attach/detach →
  re-render (explicit apply, so render can prompt on displacement).

**Target new-profile flow:** open context → "add Stream Deck profile" → checklist (default +
context auto-checked) → tick ~3 groups → render → hand-add 3–4 unique buttons → done. Ties into
the context auto-switch (§1).

**Model addition:** `Node.excluded` (inherited groups removed on this node). `effective_group_names`
now computes `(own + inherited + tag groups) − exclusions`, **nearest-node-wins** (a direct attach
nearer than an ancestor's exclusion wins, and vice-versa). Live in the scaffold.

**Initial groups to build once the engine works** (Jamie's list): `profile-switching-defaults`,
`other-defaults`, `navigate`, `chrome`, … — then most new profiles are "check 3 + add a few buttons."

**No new editing backend needed.** Editing a group happens on the real physical deck after the
VSD switches to it — identical to editing any profile now. (Corrected 2026-07-14 from an earlier
misread that wrongly put group *content* on the VSD device; the VSD only holds switch links.)

### 8.11 Build progress — **[2026-07-14, cont.]**

The "during" extraction (§8.8 steps 2–3) plus a first `render` are BUILT. Each extraction was
AST-sliced out of the monolith and re-imported (usage-filtered) so `streamdeck.py`'s namespace and
both public contracts (CLI + `import streamdeck as sd`) are unchanged; `ruff` + `pytest` +
`caster_ahk_verify` green after every step.

`streamdeck.py`: **3925 → 3210 lines.** New `sdlib/` package (~1100 lines):
- `paths.py` — locations + `FILL_ORDER`.  `constants.py` — plugin UUIDs.  `process.py` — kill/restart/exe.
- `manifest.py` — profile/page discovery, active/pinned readers, `--src` stamping, force-kill-guarded `safe_write`.
- `buttons.py` — `ButtonView` + `iter_buttons`/`find_buttons`/`read_button`, signatures, clone helpers (`_regen_action_ids`, `_propagate_icon` icon-file copy, `_set_button_image`).
- `ledger.py` — data model + pure tree/cohort/effective-groups (real, tested).  `Group` now carries `source_profile` (§8.10).
- **`render.py` — single-node render (§8.3/8.4):** reads each effective group's buttons from its source profile and places them honoring pinned / unique-anchor / FROZEN `resolved` / `deprecated` / `FILL_ORDER`; stamps `_mid`; copies icon files. `streamdeck.py ledger render <node> [--write]` is wired to it.

**Verified:** freeze path (a resolved `_mid` returns FROZEN at its slot) against real profiles; the
placement decision `_first_free` (preferred → flow → skip-claimed → full-grid=None); "no free slot"
warning (all Deck A profiles are currently full — 0 free slots anywhere). **The write path
(safe_write + clone + icon copy + `_mid`) is implemented but NOT yet live-tested** — reuses the
proven §7 propagate primitives, but needs a supervised run against a scratch profile.

**Next:** `reconcile` (manifest → ledger override/deprecation detection, §8.3), then a supervised
live write-test (create a scratch profile with free slots → `render --write` → inspect → delete),
then the authoring UIs (§8.10). Not committed yet.
