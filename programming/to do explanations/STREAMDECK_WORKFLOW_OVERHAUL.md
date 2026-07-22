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

### 8.12 Two findings + the history system — **[2026-07-15]**

**FIXED: the passthrough flag was a lie — 2787 free slots were invisible.** `is_passthrough_button`
identified a blank slot by `Settings.passthrough is True`. A scan of all **4673** live buttons found
that flag on exactly **1** of **2788** real passthroughs. Cause: `populate_empty_in_page` only fills
slots holding *no button at all*, so a slot already holding a flag-less passthrough is skipped
forever and never upgraded. Effect: `dump` / `empty` / `add` / `render` counted 2787 free slots as
occupied real buttons — **every profile looked full** (that's why render could never place anything).
Fix: identify a passthrough **by the fn it calls** (`SetMacroWizardSdForSlot` + legacy names, now in
`sdlib/constants.py` as `PASSTHROUGH_FNS`), flag kept only as a secondary signal. Same lesson as §7's
icon identity: **identify by what the button DOES, not by a fragile flag kept in lockstep.**
After the fix VLC reads 25 real + 7 free (was 32 + 0); system-wide 1885 real + 2788 free; and render's
placement path works — a dry-run onto Anki now plans 13 real placements.

**`_mid` persistence re-verified per button TYPE. ✅** The first test (§8.7) only proved survival on a
`profile.rotate` button — a false-positive risk, since render stamps `system.open` / key-logic buttons.
Retested all four types in one graceful `--quit` flush (manifest rewrite confirmed): **open, key-logic,
hotkey and rotate ALL kept the unknown key.** SD does not strip unknown `Settings` keys. §8's `_mid`
breadcrumb is safe; no sidecar needed.

**History / revert — BUILT (`sdlib/history.py`).** Jamie's correction: the Stream Deck dir is
**already a git repo** (`C:/Users/jamie/AppData/Roaming/Elgato/StreamDeck`, 87 commits, remote
`streamdeck-backup`, all 2246 ProfilesV3 files tracked, sensible .gitignore). So we do **not** create a
second repo — we commit into that one. Whole state is only 15 MB, so this is cheap.
- **Scope:** stage only `ProfilesV3/` (the repo also tracks Plugins/ etc.). **Never push** — GitPushAll owns that.
- **Attribution:** before every mutating command, any pre-existing dirt is committed as `Actor: external`
  (SD UI edit / hand edit / SD's own flush) so someone else's change is never folded into ours — this is
  what makes "was it me or the tool?" answerable at every point.
- **Log = git log.** Trailers (`Actor:`, `Kind:`, `Op:`, `Scope:`) make it machine-readable with no
  sidecar file that could drift out of sync (same anti-drift rule as §8). `Kind` = automatic | manual |
  bulk (the "recursive" multi-profile ops) | revert.
- **Hook** is at the CLI entry (`main()`), not `safe_write` — `copy-profile`/`populate`/A→B write directly
  and would slip past a `safe_write` hook. try/finally so a command that writes then exits still records.
  Read-only commands skip entirely (overhead 0.18s).
- **CLI:** `streamdeck.py history status | log [-n] | snapshot | revert <sha>`. Revert force-kills SD first
  (else its flush overwrites the restore), removes files added since the target, and commits the revert as
  its own revert point. New `--actor` flag (`streamdeck.py --actor wizard add …`) so callers self-identify.

### 8.13 reconcile BUILT + the live write-test passed — **[2026-07-15]**

**`sdlib/reconcile.py` (§8.3) — the drift-closer.** The ONLY place manifests are read for meaning, and it
reads exactly one thing: the `_mid` breadcrumb. `mid_of`/`stamp_mid` now live in `sdlib/buttons.py` (button
identity's proper home) with `MID_KEY` in `constants.py`. Detects:
- **MOVED** — a `_mid` now sits at a different slot (she dragged it) → `resolved` updated; wherever she put
  it IS its home now. Explicitly *not* a deprecation.
- **DEPRECATED** — a `_mid` is nowhere on the page → recorded in `deprecated`, dropped from `resolved`,
  never re-placed.
- **ADOPTED** — a `_mid` on the page the ledger lost track of (or a deprecated one she re-added by hand) →
  re-adopted at its real slot + un-deprecated.
- **UNIQUE** — real buttons with no `_mid`; nothing to record (unique IS the absence of a breadcrumb).

New CLI: `ledger reconcile <node> [--write]` and **`ledger apply <node> [--write]`** (= reconcile → render,
the normal way to push a node).

**LIVE WRITE-TEST — PASSED end-to-end** (scratch profile, then fully reverted; VLC and every real profile
provably untouched — post-revert `git diff 46e3919 HEAD -- ProfilesV3` was empty):
1. `copy-profile TEMPLATE "ZZ RenderTest"` → history auto-committed it `[cli/automatic]`.
2. `ledger apply zztest --write` → **21 buttons written**; both `placed` (preferred slot free) and `flowed`
   (preferred taken → FILL_ORDER) paths exercised; committed `[render/bulk]`.
3. On-disk verification: **21/21 buttons carried `_mid`; 21 icon files copied into that profile's own
   `Images/`; 0 broken icon refs** — the §7 blank-icon trap is handled.
4. **Jamie's list-nav case proven:** removed a managed button → `reconcile` marked it
   `deprecated: cleared on this profile` and dropped it from `resolved` (21→20) → re-render did NOT put it
   back (0 occurrences), the other 20 stayed frozen.
5. `history log` showed the full attributed trail (`[cli/automatic]` copy-profile → `[render/bulk]` apply →
   `[cli/automatic]` remove).
6. `history revert 46e3919` → removed 35 added files, deleted the scratch profile, restored ProfilesV3
   byte-identical, and recorded the revert as its own commit.

So render + reconcile + history + revert are all proven against real Stream Deck state. Remaining known
gaps: group-displacement prompting (currently warns + skips when a group can't fit), page-0-only scope, and
the latent BOM read issue (`get_current_page_manifest` uses `encoding="utf-8"`; at least one profile has a
BOM). Next: the authoring UIs (§8.10) — wizard blank-slot "add group", the per-profile group checklist, and
the Stream Deck Miller.

### 8.14 The authoring BACKEND + render SWEEP — **[2026-07-15]**

The §8.10 UIs are three front-ends onto one small set of ledger edits, so the edits were built **once**, in
Python, with a JSON contract — not three times in AHK. The GUIs are now thin.

**`sdlib/authoring.py` (NEW) — the mutation API.** Pure ledger manipulation; no manifest is touched. Callers
render explicitly afterwards, which is what keeps displacement *promptable* (§8.4) instead of a silent
side-effect of ticking a box. Every function returns `{"ok": bool, "error": str, ...}` so AHK parses one
predictable shape instead of scraping text.

**Detach is the subtle one**, and the checklist must not have to care: unchecking a group means *drop from
`groups`* if attached here, but *record in `excluded`* if inherited or from a tag (§8.10 "I could even remove
the defaults"). `detach` branches correctly; `attach` is its exact inverse (un-exclude, then attach only if
still not effective) so untick→retick restores the prior shape rather than accumulating a redundant direct
attach.

**RENDER SWEEP — a real gap, found while planning the checklist and fixed before building it.** Render was
add-only: nothing removed a button whose group stopped being effective, so unchecking a group would have
**orphaned its buttons forever**. Since unchecking is the checklist's entire purpose, the UI was unbuildable
without this. Render is now gather → sweep → place:
- Swept slots go back to a **passthrough** (blank but still wizard-bindable), never a hole.
- Sweep runs **before** placement, so a slot freed this pass is immediately reusable.
- It only clears a slot whose button still carries the expected `_mid`. Hand-overwritten? reconcile already
  deprecated it and the disk is hers — render forgets the mid but **never touches the key**.
- A group whose source profile **couldn't be read** is left completely alone. A transient read failure must
  never be mistaken for "she deleted every button in this group" — that distinction is why `_gather` returns
  a `readable` set separate from the button list.
- A detached group's `deprecated` entries are dropped, so re-checking it later is the clean slate she asked
  for ("unless I intentionally re-add that specific button group").

**THE LEDGER NOW RIDES IN THE HISTORY.** `sd_ledger.json` lives in `AHK/INIDATA` — outside the Stream Deck
repo history.py versions. Reverting manifests *without* it would leave `resolved{}` describing placements
that no longer exist: precisely the drift §8 exists to prevent. So each commit mirrors the live ledger to
`<sdrepo>/sd_ledger.json` and stages it in the same commit; revert restores it and copies it back. One commit
= one complete, self-consistent state. We deliberately do **not** auto-commit to the AutoHotkey repo (INIDATA's
real home) — that's Jamie's working repo and machine commits don't belong interleaved with her history. The
mirror is snapshot storage; INIDATA stays the live file. The mirror is symmetric: **live ledger gone ⇒ mirror
deleted**, or a stale mirror would quietly resurrect a deleted ledger on the next revert.

**Ledger edits are history-recorded too** (`LEDGER_MUTATING`), even though nothing on disk moves until the
next render — the ledger IS state, and an attach is exactly what you'd want to walk back. `--actor` lets a
caller own its edits in the log (the wizard will pass `--actor wizard`); default is `authoring` for a ledger
edit, `render` for a `--write`.

**New CLI** (all accept `--json`): `checklist` · `attach` · `detach` · `new-node` · `set-parent` · `del-node`
· `new-group` · `del-group` · `new-tag` · `tag` · `info` · `overview`.

**Tests — 38 unit + 21 live, all passing.** Unit (`authoring_test.py`, in-memory, real file never touched)
covers the easy-to-get-wrong cases: detach-inherited excludes rather than no-ops; retick doesn't accumulate;
nearest-wins between an ancestor exclusion and a nearer attach; sweep skips unreadable sources; sweep never
clobbers a hand-overwritten slot; every guard (cycles, orphaned children, referenced groups).

**LIVE test — the actual §8.10 target workflow, on real profiles, then fully reverted:**
new profile → check 2 groups → apply → **3 buttons land** → **uncheck a group → apply → its 2 buttons are
swept back to blank, the other group's button survives, and the ledger stops tracking the swept mids** →
re-check → apply → **they come back**. `history log` showed the trail correctly split between
`[authoring/manual]` and `[render/bulk]`. Post-revert `git diff <pre-test> HEAD -- ProfilesV3` was **empty**
— every real profile untouched.

**Extraction:** `make_passthrough_button` + `_ensure_passthrough_svg` moved to `sdlib/buttons.py` (render's
sweep needs to build a blank). `streamdeck.py` 3234 → ~3190. Also fixed the extractor itself: its
usage-filter matched names in *comments*, which had left two dead re-imports (`ButtonView`,
`STREAMDECK_EXE`) behind — it now measures usage against a comment/string-stripped copy. Ruff F/E9 clean.

**Still open:** group-displacement prompting (§8.4 — render warns + skips rather than asking); page-0-only
scope; the BOM read issue; Deck B as children of A (§8.5). Next: the AHK UIs themselves — the per-profile
checklist GUI, the wizard blank-slot "add group" row, and the Stream Deck Miller — now all thin clients over
`ledger --json`.

### 8.15 The Stream Deck Ledger MILLER — the checklist is real — **[2026-07-15]**

Voice: **"stream deck ledger"** · `MAINFUN.bat OpenStreamDeckLedger`.

**Two of §8.10's three surfaces turned out to be one build.** The GUI conventions settle it: *"multi-select
from a set = an in-line toggle list inside the Miller"* (`gui-conventions.md` — "a value from a known set is
PICKED from a list, never typed"). So the per-profile group checklist isn't a separate GUI at all — it's a
level **inside** the Miller Jamie already wanted. Scaffolded with `new_miller.py` (which wired
MAINFUNCTIONS + the test runner), so it inherits recursive search, `N.M` row actions, layout, and the `?`
overlay for free.

```
Profiles ▸  → <node> ▸ → Groups ▸ (checklist: ✓ / Enter toggles)   ← THE CHECKLIST
                       → Apply to the deck   (preview → confirm → write → restart)
                       → Preview (dry-run)   → Reader with the full plan
                       → Parent: <x> ▸       (pick-from-list; sdlib rejects cycles)
                       → Delete node
Groups ▸    → <group> ▸ → Edit buttons on the deck   (VSD-switch, then the normal wizard)
                        → Members ▸ / Source / Delete
History ▸   → every commit [actor/kind] — Enter reverts to it
+ New profile node · + New group   (pick the SD profile, then type the name)
```

**It is a THIN CLIENT and that is the point.** Every decision lives in sdlib; the menu renders rows and
forwards keystrokes. Row actions: `N.2` apply, `N.3` preview on a profile; `N.2` edit-on-deck on a group.
The checklist's detail column spells out *why* a group is on ("own" vs "inherited:chrome" vs "tag:x"),
because unticking an inherited one is an exclusion — a different edit, which sdlib decides, not the GUI.

**Apply is deliberately not one keystroke.** It dry-runs first, shows exactly what would be placed/cleared/
frozen, and only writes on confirm (then restarts SD, which only reads profiles at startup). §8.4 wants an
apply to be a conscious act, not a side effect of ticking a box.

**"Edit a group's buttons" reuses the existing backend exactly**, as designed: `vsd-ensure` the group's
source profile → look up its `col,row` in the map `vsd-map` generates → `VsdEnsureOpen` + `VsdInvokeButton`
→ the physical deck flips to the group profile and she edits it with the normal wizard. No new editing
surface exists or is needed.

**New JSON contracts** (AHK never scrapes human text): `ledger --json` on every subcommand incl.
`render`/`reconcile`/`apply`; `history --json log|status|revert|snapshot`; and `list --json`
(`{name, uuid, deck}`). `--actor miller` attributes everything this menu does in the history log.

**Bugs caught by testing, not by reading:**
- **`SdPyPath()` is defined in `VoiceConfigManager.ahk`, NOT `StreamDeckDispatch.ahk`** (which only *calls*
  it — the definition's own comment explains why: VoiceConfigManager is in every entry script's include
  closure). Including the wrong file made the root builder throw at runtime while `validate` stayed green.
  The fixture's tree self-check is what surfaced it, and a repro script gave the real stack.
- **All three GUI template calls were wrong** and `validate` cannot catch a bad `opts` key:
  `_ConfirmationModalGui` takes a `buttons` array (not `confirm_text`/`cancel_text`), `_ReaderGui` wants
  `body` (not `text`), and `_SingleFieldSkipableInputGui` returns a **Map** `{result, value}` — not a
  string. Each now goes through one wrapper (`_SdLedConfirm` / `_SdLedAskText`).
- **The profile-list parser matched an invented format.** `list` prints `Deck A  Name  <UUID>`, not
  `Name [A]` — it parsed **0 profiles**, silently emptying both "+ New" pickers. Fixed properly by adding
  `list --json` rather than fixing the regex: scraping presentation output was the actual mistake.
- **History timestamps rendered in UTC** (a 15:43 commit showed as 19:43) — epoch seconds are UTC; now
  shifted by the machine's offset.
- **`_SdLedErr` said "unknown error"** for render/apply failures, which report `warnings` rather than
  `error`. It now falls back, so "no node named 'x'" surfaces as itself.

**Tests:** a headless smoke harness exercises every level builder + formatter against the real Python
(**13/13**, with explicit regression guards for the parser/timestamp/error bugs above), plus the generated
fixture tests. **Full GUI suite: 58/58 passing** (was 56 — the 2 new ones), `ahk.py validate` clean,
`miller_convention_check` clean, ruff F/E9 clean.

**Still open:** the wizard blank-slot "add group / adopt template" row (§8.10's third surface — it touches
the existing `MacroWizardMiller`/`SetMacroWizardFunctions`, so it wants its own pass); group-displacement
prompting (§8.4); page-0 scope; the BOM read issue; Deck B as children of A (§8.5). And then the real
groups: `profile-switching-defaults`, `other-defaults`, `navigate`, `chrome`.

### 8.16 The wizard's "add group" row — §8.10 authoring COMPLETE — **[2026-07-15]**

The third and last authoring surface. Press a blank SD button → the set-macro wizard now shows
**"Stream Deck groups ▸"** right above the function catalog, next to "add a function", exactly as asked.

**It mounts the SAME checklist the ledger Miller uses** (`_SdLedGroupToggleNodes`) — not a second copy.
That's the whole point of §8.10's "two authoring UIs onto the one ledger": a Miller branch composes, so the
wizard and the ledger Miller **cannot disagree**. Ordinary composition, no new abstraction.

```
Blank SD button ->  This slot ▸
                    Wizard tools ▸
                    Stream Deck groups ▸   <- NEW
                        VS Code  (ledger node: vscode)
                        ✓ navigate          <- the real checklist, live
                          chrome
                        Apply to the deck
                        Open the full ledger Miller
                    ─────────────
                    <every function…>
```

**Where the code lives:** all of it is in `StreamDeckLedgerMenu.ahk` (`_SdLedWizardNode` /
`_SdLedWizardChildren`); `MacroWizardMiller._MwRootNodes` gained ONE row, guarded by `device = "sd"`.
Ledger logic stays in the ledger file; the wizard just mounts a branch. Both hosts already include the
file (GuiHost → MAINFUNCTIONS), so nothing new enters the include closure.

**Unregistered profiles are a one-keypress on-ramp.** If the slot's profile isn't a ledger node yet, the row
offers `Register '<profile>' in the ledger` — no name prompt (a slug of the profile name is a fine default,
renameable later in the Miller). The profile is resolved from the wizard's `<deck>:<page>:<pos>` slot via
`button-info`, called WITHOUT `--render` (the existing `_Smw_StreamDeckButtonInfo` forces an icon-PNG
render that's irrelevant and slow here).

**Tests:** the smoke harness now covers the wizard row against the live VS Code profile — **19/19**
(resolves "VS Code" from a real slot, offers Register when unregistered, subtree validates). The REGISTERED
path was proven live end-to-end: created `vscode` + a scratch group → the row rendered header + a real
checklist row + Apply + the full-Miller escape hatch, and stopped offering Register → reverted; post-revert
`git diff <pre-test> HEAD -- ProfilesV3` **empty**.

**The pre-ledger revert edge case behaved exactly as designed and documented.** Reverting to a commit that
predates the ledger printed the honest warning ("ledger NOT restored… may not match these profiles") rather
than silently deleting her ledger. And the mirror **self-healed**: deleting the live ledger out-of-band left
a stale mirror, which the next commit removed on its own (`live gone ⇒ mirror gone`, §8.14). The invariant
held without intervention.

**A note on where the bugs are.** §8.15's six bugs were all found by running the thing, never by reading it;
this section's Python-backed logic needed none.

`caster_ahk_verify`'s bridge orphan scan flagged `group (isOn ? …)` in `_SdLedToggleGroup` as a call to an
undefined `group()`. **That was a FALSE POSITIVE** — verified empirically: AHK v2 reads `identifier (expr)`
with a space as CONCATENATION, so the original code worked. (A genuine call to a nonexistent function is a
LOAD-time error, which `validate` would have caught anyway — that's the tell.) The scan is a static
heuristic on `identifier (`. Using explicit `.` is still correct: it satisfies the verify gate (a build
gate) and reads unambiguously — but it was a lint fix, not a bug fix, and shouldn't be filed as a near-miss.

The real argument for **a test that FIRES leaf actions** rather than only validating tree structure is
§8.15's three wrong GUI template contracts (Confirm's `buttons` array, Reader's `body`,
SingleFieldSkipable's Map return). Those are reachable ONLY from leaf actions, they throw or misbehave at
runtime, and `validate` cannot see them — a bad `opts` key is just a Map literal. Structure-only tests pass
straight over them.

**§8.10 is now fully built.** Remaining on §8: group-displacement prompting (§8.4), page-0 scope, the BOM
read issue, Deck B as children of A (§8.5) — and then the actual content: `profile-switching-defaults`,
`other-defaults`, `navigate`, `chrome`.

### 8.17 Leaf-action tests — closing the gap that let six bugs through — **[2026-07-15]**

Every §8.15 bug reached a passing test suite. The reason is structural, not sloppiness: `new_miller.py`'s
generated test asserts the tree's SHAPE (`MlValidateTree` → "tree-ok"), and **a Miller's real work is all
in its leaves**. Three of those bugs were wrong GUI template contracts — reachable only by pressing Enter,
invisible to `ahk.py validate` (a bad `opts` key is just a Map literal). 58 tests passed; none pressed Enter.

**`test_stream_deck_ledger_leaf.ahk` + `Fixtures/stream_deck_ledger_leaf.ahk`** fix that:
- `sd_ledger_leaf_toggle_attaches_group` — Enter on a real checklist row drives AHK leaf → Python →
  ledger write → log marker. This is the menu's most-used action and nothing else covered it.
- `sd_ledger_leaf_preview_opens_reader` — fires Preview and asserts a Reader window actually opens,
  exercising `_ReaderGui`'s opts contract and the apply-JSON formatters.

**Isolation is what makes firing real leaves safe.** `sdlib/paths.py` now reads **`SD_LEDGER_PATH`** and
`sdlib/history.py` reads **`SD_HISTORY_OFF`**; the fixture sets both before anything shells Python (env is
inherited by child processes), so the tests drive the REAL code paths against a scratch ledger with history
disabled. Verified afterwards: Jamie's ledger absent, repo HEAD unchanged, no mirror, no scratch left
behind. Defaults with the vars unset are provably unchanged. `initial_path` opens the fixture straight at
the checklist, so the test asserts the ACTION, not navigation (`test_miller.ahk` already owns navigation).

**The tests were proven able to fail.** Sabotaging `_SdLedToggleGroup` into a no-op turned
`sd_ledger_leaf_toggle_attaches_group` red with the exact expected message; restoring turned it green. A
leaf test that passes against sabotaged code is decorative — always check.

Generalised into `docs/gui-testing.md` ("Structure-only tests are not enough — FIRE THE LEAF"), since this
applies to every Miller in the codebase, not just this one.

### 8.18 The BOM fix + `ledger capture` — **[2026-07-15]**

**THE BOM BUG WAS REAL AND IT WAS EATING A PROFILE.** Flagged as "latent" twice and deferred; it wasn't
latent. A cross-profile scan silently **skipped the entire Claude profile** — `json.loads(p.read_text
(encoding="utf-8"))` raises "Unexpected UTF-8 BOM", and exactly one manifest on disk has a BOM. Any walk
that reads it either crashes or (worse, where a broad `except` catches) quietly under-reports. It had been
patched three times at individual crash sites, one incident at a time, which is why it kept coming back.

Fixed properly: **`sdlib.manifest.read_json()`** — one helper, `utf-8-sig` (strips a BOM if present,
identical to `utf-8` when absent, strictly better with no downside), and **all 28 read sites swept onto it**
across `manifest`/`buttons`/`ledger`/`streamdeck.py`. Writes deliberately still emit no BOM. The scan now
reads **30/30** profiles.

**A cautionary tale from the sweep itself.** The regex rewrote `read_json`'s OWN body into `return
read_json(path)` — infinite recursion. It didn't blow up visibly: `load_profile_list`'s broad `except`
swallowed the RecursionError and fell back to UUIDs, so `list` printed `Deck ? <UUID>` for every profile —
a plausible-looking output hiding total failure. Caught only by *running the command and reading the
output*. Two lessons, both already house rules: a mechanical sweep must not include its own definition, and
a broad `except` turns a crash into a lie.

**`sdlib/capture.py` + `ledger capture` (NEW)** — the missing rung between "the engine works" and "the
groups exist". A group's source profile is a real profile holding only that group's buttons; hand-building
one means re-creating a dozen buttons by eye with their icons. Instead:

```
streamdeck.py copy-profile TEMPLATE "G Navigate"
streamdeck.py ledger capture navigate --from Chrome \
    --slots "4,2 5,2 6,2 7,2 4,3 5,3 6,3 7,3" --into "G Navigate" --write
```
Positions are preserved on purpose — a group's buttons keep the home Jamie's muscle memory already has, so
render's "preferred slot" is the right one. Icons are copied as FILES into the group profile's own `Images/`
(the §7 trap); ActionIDs are regenerated; `--src` is re-stamped by `safe_write`. Refuses a slot already
owned by another group (two groups fighting over one button on every render), and refuses to invent the
target profile — cloning lives in the monolith, and **sdlib must never import the monolith**, so it errors
with the exact `copy-profile` command instead. Dry-run by default.

**Proven live, then reverted:** dry-run listed the 8 list-nav buttons → `--write` copied all 8 into the
group profile at their real slots with `--src` re-pointed at the new page → the group registered with its
source. Post-revert `git diff <pre-test> HEAD -- ProfilesV3` **empty**, scratch profile gone, ledger and
mirror gone. Also fixed `ledger groups` printing "0 button(s)" for a freshly-captured 8-button group — it
was counting the unused JSON fallback list instead of naming the source profile.

**EVIDENCE FOR THE REAL GROUPS** (`find_common.py`, 30 Deck A profiles, identity = what the button DOES):
Jamie's description checks out against the deck.

| slot | shared by | button |
|---|---|---|
| 0,0 · 1,0 · 2,0 · 3,0 | 53–57% | Switch Profile ×4 — her "top four left" |
| 0,1 · 1,1 · 2,1 · 3,1 | 50–53% | CloseWindowOrTab · Alt+Tab · Ctrl+Alt+N · Ctrl+Alt+M |
| 0,3 | 57% | Input Device Control |

That's **9 slots at ≥50%**, not the 12 in cols 0–3 rows 1–3: row 2 (F13 23%, F14 23%, Ctrl+K 13%) is NOT
actually a shared default, it just feels like one. Her "eight buttons down and to the left" maps to the 4+4
of rows 0–1 (+ Input Device Control at 0,3), so `other-defaults` should be the row-1 four plus 0,3 — five
buttons — and the F13/F14 pair is a *different, smaller* cohort worth its own group if she wants it.
Deliberately NOT built yet: which buttons belong in which group is her muscle memory, not a decision to
infer from a percentage.

### 8.19 PINNED groups + what the templates already do — **[2026-07-15]**

Jamie: *"make sure that in all of the profiles these are all considered pinned buttons so they show up by
default on every page"* and *"take a look at my existing templates to see if any of these could be folded in
or copied."* Both were the right instinct, and looking first changed the design.

**HER TEMPLATE ALREADY IS THE DEFAULTS GROUP.** `TEMPLATE`'s Default page carries 11 pinned buttons — the
4 switchers, CloseWindowOrTab / Alt+Tab / Ctrl+Alt+N / Ctrl+Alt+M, Scroll Up/Down, Input Device Control.
That is `profile-switching-defaults` + most of `other-defaults`, already built, already pinned. So the groups
should be **captured from what exists**, not authored from scratch.

**THE DECK IS SPLIT IN HALF — the real finding.** Surveying all 30 Deck A profiles:

| style | count | who |
|---|---|---|
| **PINNED** (defaults on the Default page) | 14 | TEMPLATE, VS Code, Claude, Music, Links, Spotify Nav… |
| **PAGE-level** (defaults as ordinary page buttons) | 17 | **Chrome**, Anki, Home, Navigation, VLC, Youtube… |

This also corrects §8.18's "53%": it never meant half her profiles lack the defaults. **Essentially all 30
have them** — 14 store them pinned, 16 store them on the page. `find_common.py` only read the current page,
so it saw one half. Her ask therefore means **migrating 17 profiles** from page-level to pinned, and a
page-level button OVERRIDES a pinned one in the same slot, so migration = write the pinned copy AND remove
the page-level duplicate. Not yet done — that edits real profiles, so it wants its own pass.

**MODEL: `Group.pinned`.** A pinned group renders to the member's **DEFAULT** page (shows on every page);
a normal group (list-navigate) stays on the current page. `--pinned` on `new-group` / `capture`.

**Render now has TWO write targets** (`_Target`), and the separation matters:
- **Per-target claims.** `4,0` on the Default page and `4,0` on the current page are DIFFERENT slots; one
  shared claimed-map would wrongly treat the second as taken.
- **Per-target blocking.** On the current page, a Default-page button's slot is spoken for. On the Default
  page, only its OWN uniques block — a pinned button whose slot the current page happens to override is
  still correct (it shows on every other page), so render writes it and WARNS about the overlap instead of
  vetoing it.
- **Sweep is per-target** — a pinned group is swept off the Default page.
- **A pinned group with no Default page is skipped loudly**, never silently written to page 0 (that would
  look right on page 0 and be wrong everywhere else).
- **RECONCILE READS BOTH PAGES.** Caught before shipping: reconcile only read the current page, so on the
  first run it would have found zero pinned mids and **deprecated the entire defaults group — silently
  unpinning every profile.**

**Live-proven, then reverted:** captured a 3-button `--pinned` group → apply → `dump` reports them under
PINNED, and the `_mid`s are provably on the Default page with **none** on the current page → reconcile found
them (no false deprecation) → re-apply idempotent (all `frozen`). Post-revert `git diff` empty. Unit tests
44/44, including the pinned/page collision at the same slot and the missing-Default-page case.

**A SECOND SELF-INFLICTED SWEEP BUG — worse than §8.18's.** The `read_json` regex used `re.DOTALL` with a
non-greedy capture, which spans lines. It corrupted **4 sites in streamdeck.py**: two string parsers became
file readers (`json.loads(raw)` in **batch-add** → `read_json(raw)`, and find-clones' signature check), and
two lost the wrapper entirely (`json.loads(<Path>)`, which just throws). **batch-add was broken and every
existing test still passed** — nothing covers it. Found by running the command. All four restored (the two
string parsers are now byte-identical to HEAD) and the whole diff re-reviewed hunk-by-hunk against git;
sdlib was undamaged. Lesson, twice over now: **a regex sweep across a large file needs a line-anchored
pattern and a hunk-by-hunk diff review — `ruff` and an import check prove nothing about semantics.**

### 8.20 The REAL GROUPS exist + per-button modularity + ADOPTION — **[2026-07-15]**

**The five groups are built** (26 buttons), captured from the profiles that already had them:

| group | pinned | from | slots |
|---|---|---|---|
| `profile-switching` | **PINNED** | Google Voice | 0,0 1,0 2,0 3,0 |
| `other-defaults` | **PINNED** | Chrome | 0,1 1,1 2,1 3,1 · 0,2 1,2 1,3 0,3 |
| `list-navigate-numbers` | page | Chrome | 4,2–7,2 · 4,3–7,3 (ClickNth 1-8) |
| `list-navigate-scroll` | page | Chrome | 6,0 7,0 6,1 7,1 |
| `list-navigate-special` | page | Chrome | 4,0 UNREAD · 5,0 INPUT |

Plus tag **`list-navigate`** = the three list-navigate groups (tick one thing, get all three; or tick them
individually — that's Jamie's "buddies / siblings of a shared parent", and it's just `Tag`, which §8.2
already had). Tree: abstract `defaults` node (profile-switching + other-defaults) → `chrome` child.

**HER REFERENCE PROFILES DISAGREED.** She said "whichever is on chrome and Google Voice and my Claude code
profile", assuming they matched. They don't: **Chrome + Google Voice have F13/F14 at 1,2/1,3; Claude has
Scroll Up/Down** (matching TEMPLATE). Went with F13/F14 (2 of 3, and she led with "the chrome version").
Also `profile-switching` had to come from **Google Voice, not Chrome** — Chrome's 3,0 is a local Hotkey
because the 4th switcher targets *Chrome itself*.

**PER-BUTTON MODULARITY** (Jamie: *"select the whole thing or individual functions within it… that would
prevent me from having to make each template extremely modular — they'd just be modular by default"*).
`Node.excluded` now holds group names AND mids (`"profile-switching/3,0"`), told apart by the `/`. New
`ledger buttons <node> <group>` (sub-rows) + `ledger button <node> <mid> [--off]`. Deliberately **not
inherited** — a button opt-out is a local override, and inheriting it would need a re-include marker (a
one-way door) to undo. Re-ticking a hand-removed button also clears its `deprecated` entry, or the tick
would look broken. `attach`/`detach` now reject a mid rather than silently writing a junk group exclusion.

The motivating case proved itself immediately: **Chrome unticks `profile-switching/3,0`** — the switcher
that points at Chrome — and keeps the other three. Group on, one button off.

**ADOPTION — the migration key, found by a dry-run.** Applying the groups to the real Chrome first produced
**13 × "no free slot"**: the groups were captured FROM Chrome, so Chrome already had those buttons — but
unowned (no `_mid`), so render saw 31 foreign "unique" buttons, refused to touch them, and flowed the group
onto leftover slots until the grid ran out. Fixed: if the button at a group's home slot **is** the group's
button (`ButtonView.signature()` — action + label + content-hashed icon), render **ADOPTS** it: stamps the
`_mid` in place, changes nothing visible, no ActionID churn, no icon re-copy. Result on real Chrome:

```
13 "no free slot" warnings  ->  0
14 [adopted]   every list-navigate button, in place, at its real slot
11 [placed]    the pinned defaults onto the Default page
```
Selective, as it must be: Chrome's own 5,1 "New Tab" is NOT adopted (in no group), and 3,0 is skipped
(unticked). Without adoption, migrating any existing profile was impossible.

`capture` also had to learn to read the **pinned** page: half the profiles are pinned-style, so
"capture 0,0..3,0 from Google Voice" found four empty slots and refused. It now reads the VISIBLE view
(page button, else the pinned one) — what Jamie would see.

**Status: nothing applied to a real profile yet.** The dry-run on Chrome is correct and every pinned button
warns "the page one wins on THIS page" — because Chrome is page-style. **The migration is the remaining
step**: apply (writes the pinned copies + adopts the list-nav) AND delete the page-level duplicates so the
pinned ones show through. That edits 17 live profiles, so it wants an explicit go-ahead; it is fully
revertible via `history revert`. Unit tests 59/59.

### 8.21 DEMOTE + the widened signature — **CHROME IS MIGRATED** — **[2026-07-15]**

**Chrome is the first real profile on the ledger.** `dump` went from `31 real + 0 pinned` to
**`20 real + 11 pinned`**: the 11 shared defaults now sit on the Default page, so they appear on **every**
page of Chrome. Re-apply is 25×`frozen`, no drift.

> **This section originally claimed the migration was a "proven visual no-op". IT WAS NOT — it shipped with
> 11 blank buttons on the deck.** The claim came from a checker that shared the code's false assumption.
> See §8.22; kept here uncorrected-in-place because the wrong claim is the lesson.

**DEMOTE — the missing half of the migration.** A page button OVERRIDES a pinned one in the same slot, so
writing the pinned copy alone was invisible. Demote retires the page-level duplicate, gated on the SAME
identity as adopt (`signature`), which is what makes it safe: the removal is a visual no-op because the
identical pinned button underneath takes over. A page button that merely *sits* in the slot (Chrome's own
3,0 Hotkey) is a deliberate override and is left alone. Reported as its own `demoted` list, never silent.

**THE SIGNATURE COULDN'T SEE MOST OF THE DEFAULTS.** The first demote run fired on only 2 of 11 slots. Not
a demote bug — `_button_signature` returns `''` for hotkey / switch-profile / multi / plugin buttons, a §7
decision (the wizard only propagates open + key-logic buttons). But **the shared defaults ARE mostly
hotkeys (Alt+Tab, F13/F14, Ctrl+Alt+N/M) and profile switchers** — so they were unsignable, hence
un-adoptable and un-demotable, and the migration would have quietly done nothing for 6 of the 8
other-defaults. Now signed under an **opt-in `all_kinds=True`**:
- **hotkey** — the key combo, with SD's `Key_unknown` padding stripped (the pad count is not stable).
- **switch** — ProfileUUID (case-normalized) + PageIndex + DeviceUUID.
- **multi** — recursive over sub-actions; one unsignable step makes the whole button unsignable.
- **plugin fallback** — UUID + full Settings (e.g. the Volume Controller mic-mute at 0,3). Errs strict:
  per-instance noise costs an adoption, never causes a wrong one. Refuses a button with no UUID, and
  refuses any **unmapped CONTAINER** (`multiactions.routine2` — a second multi UUID this module doesn't
  map) whose behaviour is in `Actions`, not `Settings`, and would otherwise sign equal to a different one.

`all_kinds` is **opt-in, not the new default**, because `find-clones` / `propagate` share this code — the §7
callers were built for open + key-logic only, and widening their reach as a side effect of a §8 need is
exactly the class of change that broke batch-add in §8.19. Verified unchanged on the live deck: hotkey
0/256, switch 0/176, multi 0/16 signed by the narrow signature, same as before.

**FALSE-MATCH AUDIT (the one failure mode that destroys a button).** Swept all live Deck-A buttons: for
every group the wide signature calls identical, compared the raw behaviour payload (ActionID/State noise
stripped). **719 signed buttons of the widened kinds, 302 distinct signatures, 0 false matches.** (A first
pass reported 24 — the *audit* was wrong, not the signature: it compared raw `Settings`, but
`_path_to_fn_call` intentionally strips `--src` provenance. Worth remembering: a scary audit result is a
claim about the audit too.)

**A REAL LATENT DATA BUG, caught because demote wouldn't fire.** `capture`'s `_visible_actions` merged the
pinned page in but returned bare buttons, so the caller built every `ButtonView` against the CURRENT page's
manifest. For a button that actually lives on the **Default** page (Google Voice is pinned-style), the icon
then resolved against the wrong directory: `_propagate_icon` found no file, copied nothing, and the clone
kept a ref to a file that wasn't there. **All 4 switchers in `G Profile Switching` had dangling icon refs**
— they'd have rendered blank on every profile (§7's trap). Fixed: each entry carries its own page. Both
`capture` and `render` now WARN when a button has an image ref but no copyable file, instead of silently
shipping a blank; proven to fire by hiding an icon file. Re-captured; all 5 group profiles now clean.

**The test suite was living in the session scratchpad** (`/Temp/claude/.../authoring_test.py`) — one session
expiry from gone. Now `Scripts/StreamDeck/tests/test_sdlib.py`, cwd-independent, **85 tests**. Every new
gate was proven able to FAIL by sabotage before being trusted (deleting the signature check turns a demote
test red), per §8.18's lesson.

**Remaining:** 16 page-style profiles still to migrate (the same one-command apply, now proven), and 25
profiles not yet enrolled as nodes. `history revert` covers all of it; Chrome's migration is `05328d3`.

### 8.22 A PASSTHROUGH OVER A PINNED BUTTON IS A BLANK — and my checker agreed with my bug — **[2026-07-15]**

**Jamie's deck was broken by §8.21's migration.** Her report: *"the three top left buttons and the bottom
left buttons are all gone."* Her diagnosis was also exactly right, unprompted: *"my 'blank' placeholder
buttons overwrite them on pages, so we probably need to remove those on any pin button."*

**The bug.** `_demote_page_duplicate` was modelled on `_sweep`, which hands a freed slot back as a
blank-but-bindable **passthrough**. That is correct for `_sweep` (the slot really is free, and the wizard
should be able to bind it). It is exactly wrong for demote: **a passthrough is still a real page button, so
it overrides the pinned one** — all 11 demoted slots rendered blank. Only the *absence* of a page button
lets a pinned button through. Fixed: demote now `del`etes the key.

**The codebase already knew.** `populate_empty_in_page` skips pinned positions with the comment
*"passthroughs over pinned would visually wipe them out"* — the rule was written down, in this repo, and I
reimplemented the hazard anyway. That skip is also why deleting is stable: nothing refills a pinned slot
(verified across a restart). **Read the neighbouring code's comments before copying its shape.**

**THE REAL LESSON — a checker that shares the code's assumption cannot catch the code's bug.** §8.21's
"proven visual no-op" was verified with `_visible_actions`, which merged pinned+page with *"a page button
overrides the pinned one"* — but only counted **non-passthrough** page buttons as overriding. So the
checker believed a passthrough-covered pinned button was still visible. **The code and the check were wrong
in the same direction, so 31/31 matched while the physical deck showed 11 blanks.** Two green lights, one
shared false premise, zero information. `_visible_actions` now models what the DEVICE does — any page
button wins, passthrough included — and is proven to flag the exact state it previously passed.

**Demote is now self-healing** and repaired Chrome without a revert: a passthrough sitting on a pinned slot
is cleared unconditionally (it carries no information; over a pinned button it is pure blanking, never
intentional), while a real page button still needs the signature match. Re-apply reported
`11 × blank placeholder removed`, all 11 slots verified truly empty, and a **health scan across every
profile on both decks found 0 pinned buttons blanked by a passthrough**.

**Guard rails now in place:** a demote test asserts the slot ends with *no key* (not a passthrough) and
would have caught this on day one; the repair path has its own test; the all-profiles health scan is the
cheap check to re-run after any pinned work. Unit tests 85/85.

**Meta-lesson, third time in this project** (§8.18 vacuous test, §8.19 batch-add, now this): *green does not
mean working.* Ship-blocking claims about the physical deck need a check that is INDEPENDENT of the code
under test — ideally Jamie's eyes on the actual hardware, which is what caught this one.

### 8.23 THE MIGRATION IS DONE — 17 profiles on the ledger — **[2026-07-15]**

**16 page-style profiles migrated, every one a verified visual no-op**, plus Chrome = **17 nodes, 179
managed buttons**, all children of the abstract `defaults` node. The shared defaults now live on each
profile's Default page, so they show on **every** page of every one of them.

**"Whichever is on chrome and Google voice and my Claude code profile" meant VS CODE, not the profile named
`Claude`.** §8.20 recorded "her reference profiles DISAGREE" and picked F13/F14 on a 2-of-3 majority. That
finding was wrong — it was a misreading. Her "Claude code profile" is the profile she *runs Claude Code in*,
which is **VS Code**, and VS Code matches the group sources **12/12**. All three references agree; `Claude`
is the odd one out and she doesn't use it (*"don't worry about Claude I don't use that one"*). **When a
survey says the user contradicts herself, suspect the reading before the user.**

**STRICT VISUAL NO-OP was the migration rule.** A default slot migrates ONLY when the profile's existing
button IS the group's button (`signature(all_kinds=True)`). Everything else is **unticked**, so the
migration cannot change what she sees. 140 buttons migrated, **53 unticked**, split:
- **40 — she has her OWN button there.** Real divergence, verified by hand: `0,2` is not a shared default at
  all (VLC = a YouTube-subtitle Key Logic, Anki = a multi-action macro, TESTING = switch+workspace, Places =
  opens `C:\Users`, Home = a profile switch); `1,2`/`1,3` hold TobiiToggle, Ctrl+S/Ctrl+O, websites.
  Per-button modularity (§8.20) is what makes this expressible — group on, one button off.
- **13 — the slot is EMPTY.** Ticking these would ADD a default where she never had one. Deliberately left
  for her to opt into: `0,2` (Flux/Signal/Text Docs), `1,2` (Flux/Places/Signal/Text Docs), `1,3`
  (Flux/Places/Text Docs), `1,1`/`2,1`/`3,1` (Places).

**Not migrated:** `WinToolsXL` (0/12 match — a different profile entirely, nothing to inherit), `Profile 1`
(empty), `Claude` (unused). The 12 already-**pinned-style** profiles (VS Code, Google Voice, TEMPLATE,
Music, Meditations, Spotify Nav, Screenshots, TextEditing Rare, Links, Links To Text Files, Text Editing)
still need enrolling — for them it is pure ADOPTION on the Default page, no demote.

**How it was verified**, given §8.22 (a checker sharing the code's assumption proves nothing):
1. Per profile, before/after snapshot with the **truthful** visibility model — **16/16 no-op**, run aborts on
   first mismatch.
2. Dry-run must be **warning-free** before any write.
3. All-profiles health scan: **0** pinned buttons blanked by a passthrough.
4. **Re-verified after a restart** — the step that runs `populate-empty` and where §8.22's damage hid.

**A near-miss worth recording:** the post-migration untick report was garbage on first run — `classify()`
treats a `_mid`-stamped button as "not matching", so re-running it AFTER migration reported every
successfully-migrated button as unticked (179 vs the real 53). Harmless (a report, not a write), but it is
the same shape as §8.22: **a helper written for one phase silently lies when reused in another.** The ledger
is the source of truth for what's unticked; don't re-derive it from a pre-migration classifier.

### 8.24 ALL 28 PROFILES ENROLLED — 12/12 defaults everywhere — **[2026-07-15]**

**Every Deck-A profile that should have the defaults now has all 12, on its Default page, showing on every
page.** 28 nodes, **293 managed buttons**, all children of the abstract `defaults` node.

Three passes, each verified independently before the next:
1. **16 page-style** migrated (§8.23) — demote + adopt, 16/16 visual no-ops.
2. **11 pinned-style enrolled** (VS Code, Google Voice, TEMPLATE, Music, Meditations, Spotify Nav,
   Screenshots, TextEditing Rare, Links, Links To Text Files, Text Editing) — **pure ADOPTION**: their
   defaults were already pinned, so render just stamps the `_mid` in place. 11/11 visual no-ops.
3. **25 empty default slots FILLED** across 12 profiles — the one deliberately non-no-op step (Jamie asked;
   *"that button is a newer addition but I am wanting to basically get it eventually on every profile"*).
   Check was tightened to match: *the ONLY diff is that the intended slots gained the GROUP'S button* —
   any other slot moving, or a new button not matching the group's signature, aborts. Nothing else moved.

Result: **0 blank default slots on all 28 profiles**, 0 pinned buttons blanked by a passthrough, re-verified
after a restart.

**`0,2` (Ctrl+K DynamicNavigator) is now on 15/28.** The other 13 have their own button there — and it is
genuinely their own (Ambient Music/Apps copy/Music = website buttons, Anki/TESTING = multi-action macros,
Apps = OpenMyCurrentStuff, VLC = a YouTube-subtitle Key Logic, Home = a profile switch, Places = opens a
folder, Audacity/Reading EBookViewer = hotkeys). Getting to 28/28 means **relocating a real button on 11
profiles** — the §8.4 group-displacement decision, still unbuilt, and not something to do implicitly.

**Two of the 13 are only LABEL drift:** Google Voice and Youtube already run the identical DynamicNavigator
with the identical icon at `0,2` — only the label differs (blank vs `Ctrl+K`). Demote correctly declines
them, because making a label appear IS a visible change. Cheap to fix on request; not done silently.

**`list-navigate` exists on Chrome ONLY.** No other profile has any of its 14 buttons, so there are no
adoption candidates — putting it elsewhere ADDS 14 buttons over occupied slots. A real design decision per
profile, deliberately not guessed at.

**Not enrolled:** `WinToolsXL` (0/12 — different profile), `Profile 1` (empty), `Claude` (unused).

**Known follow-up:** TEMPLATE is now a managed node, so a profile cloned from it starts with `_mid`-stamped
buttons but no node of its own. Render recovers (a mid-carrying button isn't a "unique", so the slot is free
and gets a fresh clone), but reconcile on an unenrolled clone is untested. Enroll clones promptly.

### 8.25 THE LEDGER PAYS OFF — one edit, 15 profiles — **[2026-07-15]**

Jamie: *"I want them all to be dynamic navigator. that is the correct button. and I don't know what control K
is all about, they should be labeled dynamic navigator."*

**One group-source edit relabelled 15 profiles.** `batch-set-label "G Other Defaults" 0,2 "Dynamic
Navigator"` → `ledger apply` every node → **15 profiles updated, and NOTHING else on any of the 28 changed**
(verified per profile: the only differing slot was `0,2`, or the run aborts). This is the §8 thesis
demonstrated: the group is the source of truth, edit it once, re-apply, done. Pre-ledger this was 15 hand
edits and no way to know you'd got them all.

Mechanism (worth knowing): a `frozen` mid re-clones from the group source on every apply — render only skips
the rewrite for `adopted`. So **source edits propagate on the next apply, automatically**.

**NEW: `batch-set-label`** (mirrors `batch-set-icon`), with `--page` to target a specific page folder UUID —
needed because a **pinned button is not on the current page**, and the default target would silently miss it.
Registered in history's MUTATING set, or its writes would go untracked.

**A wrong assumption, caught by the data before it shipped.** I set `LinkedTitle: False` when writing a
label, reasoning it was the "show my custom title" switch. It is not: all 480 titled buttons on Deck A have
`LinkedTitle: True`. Setting it would have made these the only deviating buttons on the deck.
`batch-set-label` doesn't touch it. *Check a flag's actual distribution across real data before believing
its name.*

> **CORRECTION (Jamie, same session):** *"it is set to not actually show the title is just for like back and
> stuff."* She is right, and my replacement claim above — that those 478 buttons "render" their title — was
> ALSO wrong. **Title visibility is `States[0].ShowTitle`.** Deck A: **225 titled buttons have
> `ShowTitle: false`** (the Title is reference-only metadata, deck shows just the icon) vs **140 with
> `true`**. The group's `0,2` is `ShowTitle: false`, so the "Dynamic Navigator" relabel changes **nothing
> visible** — it is a metadata/consistency fix, which is what she asked for. `batch-set-label` now takes an
> optional `show` key and prints `(ShowTitle=false — reference only, not on the deck)` so nobody again
> mistakes a label edit for a visible one. **I checked the wrong field's distribution and drew a confident
> conclusion from it — "the data says" is only as good as the column you looked at.**

**`0,2` is now DynamicNavigator labelled "Dynamic Navigator" on 17/28** — every profile that had that
function. Google Voice + Youtube were the §8.24 label-drift pair (identical fn + icon, blank label): their
copy lived on the **Default page**, so ticking the group on alone would have found the slot blocked and
FLOWED "Dynamic Navigator" to a random free slot. Handled explicitly instead — `remove-at` the drifted copy
from the Default page, tick, apply, placed at `0,2`. The other **11 keep their own button** (websites,
multi-action macros, OpenMyCurrentStuff, a Key Logic, a profile switch): reaching 28/28 needs §8.4
displacement, which Jamie deferred (*"I don't care about the 11 profiles for now"*).

**Guard rail:** three tests now assert the label is part of the signature — same fn+icon with a different or
blank label is a DIFFERENT button. That is what makes group-source relabelling propagate at all, and what
stops a drifted copy being adopted and keeping the wrong label forever. 88/88.

### 8.26 `ledger health` + what Deck B actually needs — **[2026-07-15]**

**`streamdeck.py ledger health`** — the scans that caught the real bugs, now one read-only command
(`--json`, `--deck`, exit 1 on findings). Every round of running these by hand during the migration found
something, which is the argument for making them a command. Checks:
- **blanked-pinned** — a placeholder on the page hiding a pinned button. **The §8.22 bug.**
- **blank-managed** — the ledger placed a mid the deck doesn't show.
- **dangling-icon** — image ref naming a file that isn't there (renders blank; silently breaks adopt/demote,
  whose identity hashes the icon's bytes).
- **orphan-mid** / **duplicate-mid** — a `_mid` for a group the ledger lost, or one mid on two slots.

**It found a real bug on its first run.** `Reading Kindle 7,2` held a passthrough whose
`_passthrough_blank.svg` was missing from that page's `Images/` — so SD rendered its default **rocket icon**
instead of blank. Nothing would ever have fixed it: `populate_empty_in_page` only fills slots with NO button,
so an existing passthrough's icon is never re-checked (the same "skipped forever" shape its own
`is_passthrough_button` docstring warns about). It now repairs a passthrough with a missing SVG
(`_ensure_passthrough_svg` is idempotent, so the fix is just calling it). Health clean afterwards.

**DECK B — measured, and Jamie's instinct is exactly right.** *"deck b will need some of its own templates.
like the navigation buttons at the top and stuff. most of the default identical."* Across all **37** Deck B
profiles:

| group | Deck B match | why |
|---|---|---|
| `profile-switching` | **0/4 on EVERY profile** | the switchers carry a **ProfileUUID**, and Deck B's point at the Deck B copies |
| `other-defaults` | mostly 7/0 or 7/1 | hotkeys, mic-mute, DynamicNavigator are **deck-independent** |

Deck B's top row targets the SAME NAMED profiles (Home, Apps, Navigation) — identical layout, different
UUIDs. So the shape is:
- **`other-defaults` is SHARED** — one group, source on Deck A, rendered onto both decks. Render already
  supports this: `_gather` reads a group from `grp.source_deck` and clones onto `node.deck`; icons are file
  copies, deck-agnostic.
- **`profile-switching` needs a Deck B twin** (`G Profile Switching B`), because ProfileUUID is per-deck.
  This is the first group that is genuinely per-deck, and it validates keeping `source_deck` on the Group.

**BLOCKER, not yet resolved — the A→B copy process would destroy the ledger's work.** Deck B is currently
maintained by a **full-replace mirror** (delete every non-`GECK2` Deck B profile, re-copy from Deck A,
regenerate page UUIDs, remap switcher ProfileUUIDs, strip AppIdentifier). Enrolling Deck B as ledger nodes
puts two systems in charge of the same profiles: the next A→B copy wipes the `_mid`s and the ledger's
placements. **Deck B enrollment therefore isn't additive — it means replacing the copy process**, which is
the actual content of §8.5 and needs a decision, not an implementation. Deferred to Jamie.

### 8.27 DECK B ENROLLED — 62 profiles, one shared group across both decks — **[2026-07-15]**

**62 profiles on the ledger, 585 managed buttons** (Deck A 28, Deck B 34). Every enrollment a verified
visual no-op: 27 Deck B regular + 7 GECK2, aborting on the first mismatch.

**`other-defaults` now has 62 members ACROSS BOTH DECKS from a single Deck A source.** This is the real
prize: hotkeys / mic-mute / DynamicNavigator are deck-independent, so one group edit reaches both decks.
Render already supported it — `_gather` reads a group from `grp.source_deck` and clones onto `node.deck`,
and icons are file copies. `source_deck` on the Group (rather than inferring it from the node) is what makes
this expressible.

**`profile-switching-b`** is the first genuinely per-deck group: a switcher carries a **ProfileUUID**, and
Deck B's point at the Deck B copies. Measured: **0/4 match on every one of the 37 Deck B profiles**, while
`other-defaults` was mostly 7/0 or 7/1. Deck B's regular profiles are strikingly consistent — **25 of 28**
share Home / Apps / Navigation / Chrome, the same layout as Deck A. The 3 exceptions are self-reference
(Home's own `0,0`, Chrome's own `3,0`) and WinToolsXL.

**GECK2 IS NOT A COHORT — Jamie's warning was right and then some** (*"for the actual GECK2 profiles they do
not share the same profile switching for buttons. so be careful for that."*). Only `2,0 -> Apps` is common
to all 7; `0,0`/`1,0` differ **per profile** (NAV→MACRO WORK, CLAUDE→ATLANTIC, TEMPLATE→NAV…). They share a
switcher set with neither deck nor each other. So the `geck2` node carries **`other-defaults` ONLY** — no
switcher group exists that could touch them. Verified after enrollment: all 7 switchers intact, per-profile,
and carrying **no `_mid`**, so the ledger can never touch them.

**A PAGE UUID IS NOT UNIQUE — found because `remove-at` refused to guess.** `copy-profile` clones page
folders verbatim and does **not** regenerate their UUIDs (only the A→B mirror does, as its "critical" step).
So every profile copied from one source shares its page UUIDs. Live: **8 Deck A profiles share one Default
page UUID** (TEMPLATE, Claude, Spotify Nav + the 5 `G` profiles — **I minted 5 of those this session**), 3
GECK2 profiles share another, and `G Profile Switching B` shares with Deck B TEMPLATE.
- **It is tolerated within a device** — each profile keeps its OWN folder with its own content (12 vs 11
  buttons); only the *name* collides, and TEMPLATE/Claude/Spotify Nav have been like this for ages and work.
- **Across devices it is not** — that is exactly why the A→B mirror regenerates them. **No current duplicate
  spans two decks**, so the dangerous case doesn't exist today.
- It DOES make `find_profile_by_page` ambiguous. It now **refuses rather than editing whichever profile it
  walked into first**, and takes `--profile` to disambiguate. `ledger health` reports duplicates:
  cross-deck = error, same-deck = note.
- **Root cause left unfixed on purpose:** making `copy-profile` regenerate page UUIDs is the real fix, but
  it changes a shipped command's behaviour and the existing duplicates would remain. Jamie's call.

**`0,2` is Dynamic Navigator on 37/62.** Deck B gained 11 empties filled + 3 label-drift fixes (drift needed
`remove-at` first: the drifted copy is a UNIQUE at the group's home slot, so render would have found it
blocked and FLOWED the button to a random free slot). **The drift check could not be "signatures match"** —
the signature INCLUDES the label and changing it is the point — so the assertion was narrowed to *only 0,2
may differ, and both labels must be undrawn (ShowTitle=false), making the swap invisible*. **When the
checker and the intent disagree, narrow the checker deliberately; don't relax it.**

**STILL OPEN — the A→B mirror and the ledger both now own Deck B.** The mirror is a full-replace (delete
every non-GECK2 Deck B profile, re-copy from Deck A, regen page UUIDs, remap switchers). Run it today and it
wipes every `_mid` and placement the ledger just made. The ledger only owns GROUP buttons, so it cannot
replace the mirror outright — each profile's unique content still has to come from somewhere. The likely
shape is: mirror copies content → `ledger apply` every Deck B node fixes defaults + switchers (making the
mirror's switcher-remap step redundant). Not built; needs a decision.

### 8.28 The page-UUID bug, root-caused and repaired — **[2026-07-15]**

**`copy-profile` remapped only the pages listed in `Pages.Pages` — and the DEFAULT page is not in that
list.** So every copy inherited its source's Default page UUID. The code even *tried* to fix it
(`uuid_map.get(old_default, old_default)`) but `old_default` was never a key in the map, so it silently fell
through to the old value — a bug that looks correct at the call site and reads correct in review. It now
remaps every page folder on disk plus Current/Default/Pages, keyed case-insensitively (Pages.* store
lowercase, folders are uppercase). Proven: a fresh copy of TEMPLATE shares **no** page UUID with it and
keeps all 12 pinned buttons.

**Repaired the 10 profiles that already had it** (7 sharing TEMPLATE's Default page, 2 GECK2, 1 Deck B).
One profile per set keeps the UUID; the copies get fresh ones (rename folder + rewrite Pages.*). Result:
**0 duplicate page UUIDs, 0 visible changes on any of the 10, 3075/3075 `--src` stamps correct.**

Why it mattered beyond tidiness: a `--src` stamp names a page UUID, and `replace-at`/`remove-at` resolve a
button by it. With 8 profiles sharing a page UUID the lookup is ambiguous, so **the macro wizard refused on
every one of the 239 buttons living on those pages**. That's the user-visible bug this fixes.

**THE `refresh-srcs` PUZZLE — chased down in §8.30.** During the repair it reported *"All --src stamps
already up-to-date — no changes"* while 154 stamps demonstrably moved to the new UUIDs (git confirms: 22 ×
`src SD:A:a9ceb14b` → 22 × `src SD:A:3aa2b179` in Claude's manifest alone). Three controlled probes since
show `refresh-srcs` detects and reports correctly (1 profile → 42 stale → "Refreshed 2 page(s)"; 3 profiles →
126 stale → "Refreshed 6 page(s)"; a hand-corrupted stamp → found + fixed). **The message stays
unreproduced.** What the chase DID find is the mechanism that makes stale stamps rare — see §8.30 — and a
`stale-src` health check so the invariant is verified directly instead of trusting any tool's summary.

**Deck A→B mirror: still the open conflict** (§8.27). The mirror is a full-replace and would wipe every
`_mid` on Deck B. Nothing about today's work protects against running it.

### 8.29 The deck is now checked on every turn — **[2026-07-15]**

**`sd_health` is a `caster_ahk_verify` check**, running on EVERY turn (not just when Stream Deck code
changed) because **the deck drifts from outside this repo** — Jamie edits a button in the SD app, SD flushes
its cache on exit, a profile gets copied. Read-only, ~1.3s, **WARN not FAIL** (deck drift is not a code error
and must not block a commit).

**Proven by sabotage, not by assertion.** Put a passthrough over Chrome's pinned `0,0` — the exact §8.22
bug — and verify went `WARN … sd_health WARN — blanked-pinned: Chrome [0,0]`. **Had this check existed that
morning, Jamie would never have seen the blank buttons**; instead it took her looking at the hardware and
saying *"the three top left buttons are all gone."* `ledger apply` then repaired it automatically via the
self-healing demote. That is the whole loop: a machine check for the failure mode that only a human noticed.

**A→B mirror: the guard is the doc, because there IS no script.** Searched — the 8-step process exists only
as prose in the streamdeck skill; nothing implements it. So the hazard is a future session following those
steps and wiping Deck B's ~300 `_mid`s. The skill now opens that section with a STOP block: what breaks, the
`health` → copy → **re-apply every Deck B node** → `health` sequence if it must be run, the note that
re-applying makes the old switcher-remap step **redundant** (`profile-switching-b` targets Deck B by
construction), and that GECK2 are ledger nodes too (`other-defaults` only — never attach a switcher group).

### 8.30 `safe_write` RE-STAMPS `--src` on every write — **[2026-07-15]**

Chasing §8.28's puzzle found something more useful than the puzzle: **`safe_write` calls
`_stamp_src_on_page` before writing** (`sdlib/manifest.py`), re-deriving every button's `--src` from the
page's real path. So **any write through the helper silently self-heals that page's stamps.** That is why
stale `--src` is rare in practice, and it is almost certainly why the dedupe's stamps ended up correct: the
`restart` afterwards calls `refresh_all_srcs`, and every subsequent `safe_write` re-stamps anyway.

**It also invalidated my first test of the new check.** Sabotaging a `--src` via `safe_write` "didn't
work" — the in-memory string changed, the on-disk one didn't. Not a failed write: `_stamp_src_on_page`
*corrected the sabotage* on the way out. **A test that goes through the code path being tested can be
silently repaired by it.** Redone with a raw `write_text`, `stale-src` fires correctly and `refresh-srcs`
repairs it.

**NEW health check `stale-src`:** a `--src` naming a page the button doesn't live on. It matters because the
macro wizard trusts `--src` for slot identity — a stale one **rebinds the wrong button**. Only a raw folder
rename (bypassing the helper) can produce one, which is exactly what the §8.28 remap did.

**Verdict on §8.28's message: unreproduced, and no longer load-bearing.** Three probes show `refresh-srcs`
reports honestly. The invariant is now checked directly by `ledger health` — running on every turn via
`caster_ahk_verify` (§8.29) — so a stale stamp cannot hide behind a tool's summary again. That is the right
resolution for an unexplained report: **stop trusting the report, check the thing.**

### 8.31 The Miller, verified against the real ledger — **[2026-07-15]**

**`stream deck ledger` opens and renders all 65 nodes** (62 profiles + the 3 abstract templates) — screenshot-
verified, not just test-verified. Each row shows its deck tag and effective groups: `geck2-*` correctly show
**`other-defaults` only**, `defaults` / `defaults-b` / `geck2` show as `abstract`, and the Deck B nodes show
`profile-switching-b`. Root offers Profiles / Groups / History / + New profile node / + New group.

Every CLI call the Miller makes returns clean JSON at this scale — `list --json` (the monolith's, for real
profiles), `ledger --json overview|checklist|buttons|button|apply`. **60/60 GUI tests pass**, including the
four ledger ones (menu opens + valid, esc closes, leaf toggle attaches a group, leaf preview opens a Reader).

Worth knowing: the leaf tests run against an **isolated temp ledger** (`SD_LEDGER_PATH`), which is why they
are safe to run on a live machine — but it also means they prove the leaf FIRES, not that the menu is usable
at 62 nodes. The screenshot is what proves the second thing. Both matter; neither substitutes for the other.

**Latent inconsistency spotted, not on the Miller's path:** `ledger --json list` ignores `--json` and prints
the aligned text. The Miller doesn't call it (it uses `overview`), so nothing is broken — noted so the next
person doesn't parse the presentation format by accident.

### 8.32 §8.4 GROUP DISPLACEMENT — the last structural gap, closed — **[2026-07-15]**

**`ledger conflicts <node>` reports; `ledger displace <node> <mid> [--to POS] --write` acts.** The §8 design
is now feature-complete.

The problem: a group button has a HOME slot (its position on the group's source — the one Jamie has in
muscle memory). When a profile has *her own* button there, render did the safe-but-wrong thing — flowed the
group's button to a random free slot (page group), or wrote the pinned copy anyway and warned that her page
button wins on page 0 (pinned group: invisible exactly where she looks, visible everywhere else — the worst
of both). `0,2` is the live case: 25 profiles hold their own button there.

**WHY IT'S A SEPARATE MODULE, NOT A RENDER FLAG.** Everything else in §8 is a visual no-op or touches only
buttons the ledger owns. **This is the one operation that moves a button Jamie made and the ledger does not
own.** So it is deliberately hard to trigger by accident: never implied by `apply`, names the button and the
destination before acting, and **refuses rather than guesses**:
- `--to` that isn't free → error naming WHY (pinned-covered / occupied / invalid) + the free list.
- No free slot anywhere → refuses; never invents one.
- Home slot holds a MANAGED button → refuses, points at `ledger button` instead.
- Dry-run by default, and a dry-run **leaves the ledger untouched** (verified: the untick survives it).

Free means: not the home slot, not covered by a real **pinned** button (she'd never see it there), and no
real button on the page — **a passthrough IS free**, which is what passthroughs are for.

**Proven on the real VLC.** Its YouTube-subtitle Key Logic at `0,2` moved to `5,0` **byte-identical**
(signature match: icon, label, and both Key Logic sub-actions travelled with it), `0,2` became Dynamic
Navigator, and **no other slot on the profile changed**. `safe_write` re-stamps `--src` (§8.30), so the
wizard follows the button to `5,0` automatically — nothing else to update. `displace` is in history's
LEDGER_MUTATING, so it is one `history revert` away.

**7 new tests on the free-slot rule** — the one place a wrong answer relocates her button somewhere
invisible — each proven able to fail by sabotage (deleting the pinned guard turns one red). 95/95.

**Not rolled out** *(superseded by §8.33 — rolled out 2026-07-15 to 57/62 after Jamie chose the `near`
rule).* 24 profiles still held their own `0,2` at the time of writing.

---

### 8.33 The `near` rule, `move`, and the 0,2 rollout to 57/62 — **[2026-07-15]**

**The destination rule was wrong, and it mattered on 16 of 19 profiles.** `displace` picked the first free
slot in `FILL_ORDER`. That order exists to place a **new group button** well — start at `4,0`, work through
the primary zone — because a fresh button should land somewhere good. An **evicted** button is the opposite
case: Jamie already chose its slot, and a utility slot like `0,2` is *itself her statement that the button
is low-priority*. `FILL_ORDER` promoted it into prime real estate and filled the gaps the zone map
explicitly says to leave empty ("leave intentional gaps between unrelated function groups").

The default is now `near` — the closest free slot to its current home, ties broken on `FILL_ORDER` so it
stays deterministic. Measured on the real decks, `near` and `fill` **disagree on 16 of 19 profiles**
(`testing` 4,0→3,2; `places` 5,0→2,2; `reading-ebookviewer` 5,0→3,3), so it was a decision, not a
tie-break. `--order fill` keeps the old behaviour; `--to` overrides both. Jamie chose `near`.

**A silent cap was hiding the right answer.** `conflicts` returned `free[:8]`. On `anki` the true nearest
(`4,2`) ranked *behind* the cap, so the reported plan said `4,1` — a wrong answer that looked
authoritative. Cap removed. The "no silent caps" rule biting for real: a truncated list that still renders
as a complete one is worse than an error.

**`move_button` extracted** as the primitive under `displace`, exposed as `ledger move <node> <from> [to]`.
It is the fix-up when a past move went somewhere she didn't want (it re-homed VLC's Key Logic from the
`fill`-era 5,0 to 2,2), and it is what the GUI's "Move to…" needs. Same guards, because the hazard is
identical: refuses a **managed** source (render would just undo it), an occupied destination, a
pinned-covered destination. Note `move` ranks `near` from the button's **current** slot — right for a
nudge; pass `--to` when you want "where displace would have put it".

**`_wanted_homes` closed a latent hole.** A displaced button could land on the home slot of a group button
that is ticked but not yet placed — trading one conflict for the next. Reserved slots now exclude those.
Two exclusions keep it honest: an **unticked** button is never placed, so its home really is free (F13 is
unticked on many profiles; reserving 1,2 everywhere would throw away a usable slot), and a button
**already placed elsewhere** is frozen and doesn't want its home back. It changed **none** of the 19 moves
— a guard against the next one, not a behaviour change.

**history:** a one-button `move`/`displace` no longer reports `kind=bulk`. That label is what Jamie reads
when choosing a revert point, so it has to tell the truth about blast radius; only ops that re-place every
managed button on a node (`apply`, `render`, `propagate`…) are bulk.

**The rollout: 57/62.** 19 profiles displaced + applied in one pass. **5 skipped, deliberately:** Ambient
Music and Music (both decks) are soundboards — every one of 23–25 keys is a track, genre or playlist, so
there is no free slot and Dynamic Navigator would cost her a song. Apps on Deck B is full only because it
has **drifted from Apps on Deck A** (A has 7,0 empty; B has "Stream Deck" at 4,0 where A has a
multi-action) — it should inherit A's free slot once the mirror is narrowed, so it is deferred, not
refused. `ledger conflicts --mid other-defaults/0,2` lists exactly who is still blocked. Health clean
throughout.

### 8.34 The Conflicts UI — a destination is picked, never typed — **[2026-07-15]**

`ledger conflicts` grew a **whole-ledger** mode (omit NODE, optional `--mid`) returning every node's
conflicts in ONE call, so the GUI does not shell 62 times to draw one level, plus `all_conflicts`'s
`blocked: true` for profiles with no free slot — **a real answer, not an omission**. Dropping them would
read as "all done" when five decks still have none.

Shape follows gui-conventions rather than taste: a destination comes from a **finite known set** (the free
slots), so it is a pick-list, never a text box; nested menu ⇒ Miller; and **every action is a visible row**
— nothing hides behind an `N.M` compound. Root **Conflicts** section (all profiles, + bulk) and a per-node
**Conflicts** branch sitting next to `apply`, because it answers the question `apply` raises: *why isn't
that button on this profile?*

**The 8×4 grid preview** (`preview_text`, which the Miller already supports) is what makes a destination
mean anything — `3,2` is abstract; a picture of the deck is not. Building it surfaced a real bug: checking
`mid` before `src` painted the entire pinned block as "the ledger's" and **the pinned glyph never appeared
once**, hiding the very thing that makes a slot unusable. Pinned is now checked first. The Music grid then
explains its own skip at a glance — a wall of ▪, ★ at 0,2, not one ·.

Verified headlessly (`_SdLedGridText` is pure) against real `conflicts --json` payloads, after the keyboard
harness turned into a rabbit hole — SendKeys mangled digits into the filter box and arrows never reached
the list. Two attempts, then stop: the rest of the AHK is row-shaping over JSON already proven on the CLI.

**Found while committing: `Helpers/StreamDeckLedgerMenu.ahk` had never been committed at all** — the viewer
was tracked, the menu was not; collateral from the over-staging cleanup. A machine restore would have lost
the entire ledger Miller. Its header also still advertised the dropped "stream deck ledger" phrase, which
is where the stale spoken aliases in the auto-generated `.meta.json` came from.

### 8.35 The A→B mirror, measured — ⚠️ SUPERSEDED BY §8.36, see below — **[2026-07-15]**

> **This section's conclusions are WRONG.** It measured only Deck B's *Default* pages, so it missed
> that Deck B's CURRENT pages hold 96 buttons of Jamie's own content. Its "2 buttons lost" figure is
> debunked in §8.36 (they're duplicates of Deck A's), and its recommended "narrow current-page-only
> mirror" would have destroyed exactly the content it was meant to protect. Kept as a record of the
> reasoning error: a measurement of half the system read as a measurement of the system.

The STOP block claimed "~300 `_mid`s, silently destroyed". Counted properly:

| | |
|---|---|
| Profiles the copy deletes | **27** (of 34 Deck B nodes; 7 `GECK2` preserved by name) |
| `_mid` breadcrumbs destroyed | **265** |
| ...on a **Default (pinned)** page | **265 — all of them** |
| ...on a **current** page | **0** |
| Buttons **permanently lost** | **2** |

**Deck B has ZERO managed buttons on a current page.** Every one of the 265 is pinned at its home slot on
the Default page, so nothing on Deck B depends on frozen placement, and `ledger apply` rebuilds all 265
*exactly* where they were — the ledger resolves nodes by profile NAME, so fresh UUIDs do not break it. The
documented recovery is therefore **provably sufficient** for everything the ledger owns. That downgrades
this from "catastrophic" to "recoverable + 2 known casualties".

**The 2 casualties** are Jamie's own Deck-B-only Default-page buttons whose Deck A twin differs: VS Code B
7,2 (hotkey; A has nothing there) and 7,3 (hotkey; A has ClaudeNextChat). 14 more of her Default-page
buttons are identical to A and survive by luck. The skill's vague "individual Deck B customizations are
lost every copy" is now exact: **2 buttons, named.**

**The design this points at.** The Default/current split on Deck B is *clean* (265/0), so the copy never
needs to touch a Default page. Narrow it to: sync each B profile's **current page** from its A twin, never
delete a profile the ledger manages, never write a Default page. The hazard then disappears **by
construction** instead of by recovery, and the 16 unmanaged Default-page buttons (including the 2) stop
being casualties at all. **Not built** — it deletes and rewrites 27 real profiles, and deserves its own
session with a dry-run and a real test rather than the tail of a long one. The measurements are the hard
part, and they are done.

### 8.36 Deck B is not a mirror — the copy process was a loaded gun — **[2026-07-15]**

**§8.35 was wrong, and wrong in the dangerous direction.** It measured only Deck B's *Default* pages,
concluded "2 buttons permanently lost", and recommended building a narrow current-page-only mirror.
Both halves collapse under one more measurement.

**First: the 2 casualties were never casualties.** VS Code B's `7,2`/`7,3` are labelled `F13` but their
`QTKeyCode`s decode to **F17/F18** (`NativeCode` 128/129 = `VK_F17`/`VK_F18`). `bindings.json` maps
`F17 -> fn:ClaudeBackChat` and `F18 -> fn:ClaudeNextChat` — and Deck A's VS Code calls **ClaudeBackChat at
7,2 and ClaudeNextChat at 7,3**, the same two functions at the same two slots, just directly instead of
through a hotkey round-trip. Zero loss. (Their `F13` labels are simply stale; `ShowTitle` is false, so
nothing ever displayed the lie.)

**Second, and this is the finding: DECK B IS NOT A MIRROR OF DECK A.** Comparing the twins' CURRENT pages:

| | |
|---|---|
| identical | **281** — B genuinely was copied from A, once |
| **DIFFERENT** | **82** — *not* drift |
| **only on Deck B** | **14** — a full replace deletes these outright |
| only on Deck A | 28 |

The 82 are not stale copies of A's buttons. They are **different buttons**:

- **`links`** — A: `GAMES1/GAMES2/GAMES3/DREAM2/DREAM3`. B: **`TV1/TV2/TV3/ROT1/ROT2`**.
- **`navigation`** — A: TobiiToggle, SwitchTopDesktop, Chrome, StartScreenSaver, SwitchDragonMic.
  B: **a DisplayFusion resolution panel** — `1440p`, `1080p`, `720p`, `480p`, `Land` — plus **9 buttons
  that exist nowhere on Deck A** (SnapMoveMonitorMax, MoveCurrentWindowToNextDesktop, Win+Up, Win+Left…).

`navigation-b` is a purpose-built window-management deck. `links-b` is a different link set. Running the
documented copy would **delete Jamie's TV links and her entire resolution panel** and paste Deck A's
content over them. **96 real buttons** (82 + 14) are hers, not A's.

So the §8.35 recommendation — "narrow it to sync each B profile's current page from its A twin" — was
exactly the wrong tool: the current page is *precisely* where her divergent content lives. A "narrow"
mirror would have destroyed all 96 while feeling safer than the full replace.

**The ledger half was over-feared, symmetrically.** All 265 `_mid`s are on Default pages; zero managed
buttons sit on a Deck B current page. Nothing there depends on frozen placement and `ledger apply`
rebuilds all 265 exactly (nodes resolve by profile NAME, so fresh UUIDs are irrelevant). The danger was
never the ledger — it was always her own content, on the half nobody had measured.

**Built: `sdlib/mirror.py` — a reporter with NO WRITER** *(superseded within the hour: Jamie answered the
question, and §8.37 below builds the writer on her terms — every non-GECK2 profile IS a mirror.)* `ledger mirror [NODE]` compares
each twin pair's current page and leads with the 14 buttons a copy would destroy. There is no `--write`
because there is no correct sync until someone decides what the two decks are *for*, and a wrong one
costs 96 buttons. If the goal ever becomes "B should get A's new buttons", the rule must be **add/overwrite
only where A has a button, never delete a slot only B has** — and even that overwrites the 82 deliberate
differences, so it needs a per-profile opt-in, never a blanket run. That is a decision for Jamie, not a
default for a tool.

**One classifier rule earns its keep:** a switch-profile button on B *should* differ from A's — each
targets its own deck by construction (`profile-switching-b`). Sabotaging that tolerance turns 82 real
diffs into 110, burying the signal under 28 fakes. Tested, and proven able to fail.

**The skill's Deck B section is retired**, not softened: the copy steps are kept only as a record of what
the old process did, under a 🛑 header, and the two lines that called B "a mirror of A" and warned that
"edits on B will be overwritten next time a full copy runs" are corrected — that warning was backwards and
would have discouraged her from editing the deck that holds 96 of her own buttons.

### 8.37 The mirror, built on Jamie's terms — and the three bugs it took to get right — **[2026-07-15]**

§8.36 ended with "there is deliberately no writer — someone has to decide what the two decks are FOR".
Jamie decided, immediately and unambiguously:

> *Besides the actual decks labeled GECK2, every other profile should just be mirrored, all of those
> differences are nonsense. Mark every profile as opt in except for those pages that I mentioned.*

So the writer exists now. The measurement wasn't wasted — it's what made the question askable, and it's why
the answer could be acted on in one pass instead of discovered mid-destruction.

**Shape.** `Node.mirror` (opt-in per node, never a blanket sweep over whatever is named `-b`);
`ledger mirror-set all|<node> on|off`; `ledger mirror-sync [NODE] [--write]`, dry-run by default, naming
every copy and every DELETE. All 27 twins opted in; the 8 GECK2 nodes were never candidates — they have no
Deck A twin, so `twins()` excludes them by construction as well as by rule. One history entry per run.

**What sync refuses to touch, each for a reason that would otherwise be a bug:**
- **Default pages** — all 265 of Deck B's `_mid`s live there and `apply` owns them (§8.35).
- **Managed buttons on A's current page** (14) — copying a `_mid` forges a breadcrumb the target's
  `resolved` map never issued → instant `duplicate-mid`/`orphan-mid`.
- **Slots B pins** (4) — a page button *overrides* a pinned one; that is the Chrome-blanking bug (§8.21).
- **Switchers with no B twin** (2: "Claude", "Spotify Nav" — Deck-A-only).

**THE THREE BUGS.** Worth recording because two of them were re-inventions of hazards this codebase had
already documented, and the third was invisible without the health check.

1. **The switch remap didn't recurse.** A's 32 switchers point at Deck A profiles; copied verbatim they turn
   Deck B into a remote control for Deck A. Handling only *bare* switches silently skipped **11** buttons —
   because "switch profile, then open the app" is one of Jamie's commonest shapes, so the switch hides
   inside a multi-action's `Actions[0].Actions[]` or a Key Logic's press slots. Recursing fixed it; the
   skip count went 26 → 16.

2. **Icons didn't travel — 50 blank buttons on Deck B.** `Image` is a path relative to each page's OWN
   `Images/`, so a whole-button JSON copy leaves the target referencing a file that isn't there.
   `buttons._propagate_icon` exists for exactly this and its docstring literally says *"This is the piece a
   naive whole-button copy misses"*. I wrote the naive copy anyway — the same failure mode as the Chrome
   bug (§8.21), where the hazard was documented and re-implemented regardless. **`ledger health` caught it
   on the first run**, which is the entire argument for the `sd_health` check existing.

3. **A blank button compared EQUAL to its healthy source, so it never healed.** A signature includes the
   icon *ref*, not the file's existence — so after the icon fix, 48 of the 50 blanks were repaired (they
   differed for other reasons) and **2 survived**, classified `same` forever. `_icon_broken` now makes a
   dangling icon fail the comparison, which makes the sync self-healing. Health: 50 → 2 → **0**.

**Comparison is remap-aware** (`_same_button`, shared by the report AND the sync). B is compared against
A-*as-it-would-be-copied*, not raw. Compare raw and every correctly-mirrored switcher reads as drift
forever: the report nags about 24 buttons that are already right, and the sync rewrites all 32 on every
run instead of being idempotent. The same principle as §8.4's "a reported plan and the write that follows
can never disagree" — which is also why the **pinned guard moved into `plan_sync`**: it lived only in the
write path at first, so the dry-run promised 4 copies the write would silently skip. A plan that lies is
worse than no plan.

**Result:** identical **281 → 372**, different **82 → 14**, only-on-B **14 → 0**. Second run: `copy 0,
DELETE 0` — idempotent. `ledger health` clean, deck restarted.

**The 14 remaining differences are correct and should stay.** They're Chrome's ListNav buttons: *managed* on
Deck A, while `chrome-b` has its own tab panel (First / Last / M5 / NewT / PutL / KillT). I briefly
attached the three `list-navigate-*` groups to `chrome-b` to "finish the mirror" — inferring past what she
actually asked — and `apply` immediately refused: **13 of her buttons blocked, and exactly 1 free slot** to
displace them into. That refusal was the tell that the inference was wrong, and it was reverted cleanly
(no residue in `groups`/`excluded`/`resolved`). **Do not re-attempt it.** Group membership is a ledger
decision Jamie makes; the mirror only syncs content.

### 8.38 Mirroring MEMBERSHIP, the GECK2 switcher template, and health's silent abort — **[2026-07-15]**

Jamie, unambiguously: *"EVERY PROFILE ON B SHOULD BE IDENTICAL TO A UNLESS IT IS ONE OF THOSE SPECIAL
PROFILES. OVERWRITE THE CHROME PROFILE. THAT HAS DRIFTED."* — and, fairly, *"PLEASE STOP ASKING ME ABOUT
THIS."* §8.37 left Chrome as an open question it should have just resolved.

**The sync only mirrored CONTENT, so `chrome-b` could never receive ListNav — ever.** Not a tuning issue,
a structural one: those buttons are MANAGED, so the content sync must not copy them (a copied `_mid` forges
a breadcrumb the target's `resolved` map never issued), and only the ledger can place them — which it won't
for a node that isn't a member. **Membership is half of what a profile IS**, and a mirror that skips it is
quietly incomplete forever.

**THREE AXES — and the first cut mirrored one and changed NOTHING.** `mirror_groups` aligned `groups`, ran,
and reported no work: **both `chrome` and `chrome-b` have `groups: []`**. Chrome's ListNav arrives via the
`list-navigate` **TAG**; the shared defaults arrive via the **PARENT**. So the real set is:

| axis | mirrored? | why |
|---|---|---|
| `groups` | yes | direct attachments |
| `tags` | **yes** | Chrome's only membership route — the one that broke this |
| `excluded` | yes | drop it and a button she deliberately turned off comes back |
| `parent` | **NO** | A roots at `defaults`, B at `defaults-b` — the deck split working as designed |

with `profile-switching` → `profile-switching-b` substituted (same role, different deck).

**Managed slots gained a rule.** A's managed button can't be copied, but B must still END UP with it, so
anything of B's in the way is DELETED and `apply` — which owns managed placement — fills it. `sync` now runs
reconcile+render per pair afterwards; without it the mirror leaves holes exactly where Deck A has its group
buttons ("I synced and Chrome B still has no ListNav").

**Result: identical 281 → 387, different 82 → 2, only-on-B 14 → 0, idempotent.** `chrome-b` gained the tag
and lost 12 drifted hotkeys. The final 2 diffs + 2 A-only are *correct* skips: pinned-covered on B, or a
switch to Claude / Spotify Nav, which have no Deck B twin.

#### The GECK2 switcher template

`geck2-switching` (PINNED, source `G GECK2 Switching` [B]), captured from GECK2 MACRO WORK's top-left four:
`0,0→HOME  1,0→MACRO WORK  2,0→CLAUDE  3,0→ATLANTIC`. Each profile **excludes the switcher aimed at itself**
and keeps her "blocked" green button there — the pattern she described on GECK2 HOME, now a clean diagonal:

```
GECK2 HOME         [BLOCKED]   MACRO WORK*  CLAUDE*     ATLANTIC*
GECK2 MACRO WORK   HOME*       [BLOCKED]    CLAUDE*     ATLANTIC*
GECK2 CLAUDE       HOME*       MACRO WORK*  [BLOCKED]   ATLANTIC*
GECK2 ATLANTIC     HOME*       MACRO WORK*  CLAUDE*     [BLOCKED]
```

Exactly the `profile-switching` pattern (where `chrome` excludes the switcher aimed at Chrome). The blocked
button is a hotkey whose `QTKeyCode` is **33554431** — the "no key" padding value `_hotkey_actions` already
strips — so it fires nothing by design. Cloned to the three slots that lacked it, **icon included**
(`_propagate_icon`), because MACRO WORK's self-slot held a pointless switch *to itself* and CLAUDE's/
ATLANTIC's held blanks.

**Gotcha worth its own line: a group is EVERY button on its source profile.** Building the source with
`copy-profile "GECK2 TEMPLATE"` gave it 16 buttons, and the first apply cheerfully planned to place
TEMPLATE's leftovers at `3,3/4,3/5,0/6,0/7,0` on every GECK2 profile. Caught in the dry-run; source stripped
to exactly 4. **A group source must contain ONLY the group.**

**NOT attached to GECK2 TEAMS / OUTLOOK / TEMPLATE.** Their `2,0 → Apps` is the **only escape out of the
GECK2 world on any profile** — the other four have none — and the template would overwrite it. Left alone
pending her call. Attaching it there is a one-liner once she says so.

#### Health was ABORTING, and its silence read as a pass

`geck2-nav-g` pointed at profile `"GECK2 NAV"`, which Jamie **renamed to `"GECK2 HOME"` in the SD UI at
11:40** while this session ran (`capture_external` caught it as commit `d65c472` — exactly what that
mechanism is for). `resolve_profile` raises on an unresolvable name, so `ledger health` **died partway
through** with `No profile matching 'GECK2 NAV'` and printed nothing — and every earlier "health OK" had
been a *pass over a shorter list*, not a clean bill. Repointing the node (all 8 frozen placements kept)
made health see four GECK2 profiles it had never checked, carrying **90 real findings**.

**The root cause, and it is a good one:** health ALREADY had a try/except meant for exactly this — and it
had **never fired once**. `resolve_profile` reports a bad name with `sys.exit()`, which raises **SystemExit
— a BaseException, NOT an Exception** — so `except Exception` sailed straight past it. (The rest of sdlib
knows this: `mirror.drift`, `all_conflicts` and `displace` all catch `(SystemExit, OSError, ValueError)`.
Health was the one that didn't.) **FIXED** — it now records an `unresolvable-node` finding, says how to
repair it, and keeps scanning.

**The lesson generalises: a checker that dies on the first bad node is blind to every node after it, and a
checker whose failure looks like success is worse than no checker at all.** The sabotage test is the
clearest one in the suite: restoring `except Exception` doesn't turn a test red — it kills the entire test
run mid-suite with the same bare `No profile matching` line health printed on the real deck.

The 90, all pre-existing and all repaired: 89 dangling `_passthrough_blank.svg` refs (`populate-empty` —
the documented repair), 21 stale `--src` on the new group source (`refresh-srcs` — which *did* report work
here, incidentally retiring the old "refresh-srcs says no changes" puzzle as it correctly reporting nothing
to do), ATLANTIC's duplicate CLAUDE switch, and two page-passthroughs hiding pinned buttons — the
Chrome-blanking shape (§8.21), deleted outright rather than replaced.

### 8.39 "What's next?" — page-0 scope was the answer, and it hid two bugs — **[2026-07-15]**

Asked what to work on next, the honest way to answer was to read the roadmap's own open list rather than
invent something. Of what §8.16 left open: the BOM issue is fixed (§8.18), §8.4 displacement and §8.5 Deck B
are done (§8.33–8.38). **`page-0 scope`** was still there, described as a theoretical scope note. It wasn't
theoretical — **10 profiles have 2–4 pages** — and checking it turned up two real bugs in the mirror built
one turn earlier.

**BUG 1: the sync wrote to the WRONG PAGE.** It used `get_current_page_manifest` — `Pages.Current` — which
is *whatever page the deck happened to be showing when SD last flushed*. A UI cursor, not an identity. For
26 of 27 twins Current was page 0, so it looked right. On **`Links [B]` Current was page INDEX 1**, so Deck
A's page 0 was compared against, and written onto, **Deck B's page 2**. Actual damage: one button (git shows
a blank became `TALK1` on a page already 22/23 identical) — but the rule was wrong, and every "identical"
figure was measured on whichever page each deck happened to be parked on. Pages are now keyed by **index in
`Pages.Pages`** — the order they appear on the deck, and the only page property that is stable and visible.

**BUG 2: only page 0 was ever synced.** "Every profile on B identical to A" was true for one page out of up
to four; 29 buttons sat unmirrored on pages Jamie reaches with a Next-Page button. `drift` and `plan_sync`
now walk every page by index, and each action carries its `page` — "copy 4,0" is ambiguous on a 4-page
profile. Each page has its **own manifest and its own `Images/` dir**, so the write resolves both per page;
getting that wrong drops the icon in the wrong folder and the button renders blank.

Pages that exist on only one deck are **reported, never invented** — creating one means minting a UUID, a
folder and an `Images/` dir and registering it in `Pages.Pages`: profile surgery, not a button copy.
`reading-kindle` and `vs-code` each have a Deck A page with no Deck B counterpart, and `ledger mirror` now
says so out loud instead of quietly comparing 2 pages against 1.

**THE JUDGEMENT CALL, and it is the point of this entry.** The newly-visible pages wanted **14 deletions**.
Checked before writing (the same "does Deck A still have it?" scan that vetted the first 14):
- **12 were duplicates** — `links-b`'s page 1 was carrying a copy of `links`' page 0. Every one survives on
  Deck A. Safe.
- **2 were real and unique**: `apps-b`'s `DRAGON` multi-action, and **`caster` — which launches
  `Run_Caster_DNS.bat`**. Her Caster launcher.

*"All of those differences are nonsense"* was Jamie's call on **page-0 data she had been shown**. These are
page-1 buttons no report had ever surfaced. Deleting her Caster launcher on that authority would have been
wrong; asking her again, right after *"PLEASE STOP ASKING ME ABOUT THIS,"* would also have been wrong. So
neither: both slots were **blank on Deck A**, so the two buttons were **promoted to Deck A** instead of
deleted from B. The decks end up identical — which is exactly what she asked for — and the buttons survive.
When an instruction and a hazard seem to collide, look for the move that satisfies the instruction without
paying its price; it is often there.

**Result:** identical **281 → 442 across ALL pages**, different → 2, only-on-B → **0**, idempotent, health
clean. **135 tests** — the index rule is proven by sabotage: restoring "trust Current" turns three red with
the precise wrong answer it produced on the real deck.

**Still genuinely open on §8:** creating a missing page on Deck B (`reading-kindle`, `vs-code`); the wizard
blank-slot "add group / adopt template" row (§8.10's third surface); and whether `mirror-sync` should run
automatically after an `apply` on a Deck A node, so "edit A, B follows" needs no remembering.

### 8.40 The mirror runs itself — and two bugs found by testing that it does — **[2026-07-15]**

Jamie: *"yes I mean it should automatically sync always after changes."* Correct, and it's the whole game:
having to remember `mirror-sync` is **exactly the discipline that let Deck B drift 96 buttons**. A mirror
you have to maintain isn't a mirror, it's a chore with a good reputation.

**`safe_write` is the one chokepoint every write passes through** — `add`, `batch-add`, `remove`, the
wizard, `render`, `displace`, `capture`, all of it — so it now records the `.sdProfile` it touched, and
`streamdeck.py` mirrors those profiles' twins once, at the end of the command, **inside the same history
commit** as the change that triggered it. A change and its mirror therefore share one revert point. Matching
is on the profile **directory**, never the name: "Apps" exists on both decks.

Four properties keep it from becoming a menace:
- **Scoped** — only twins of what *this process* wrote, not all 27. (Tested: nothing written → nothing synced.)
- **No-op when clean** — sync is idempotent, so an unchanged twin writes nothing.
- **Non-fatal** — an auto-mirror must never fail the command she actually ran; it reports and returns.
- **Never recursive** — `mirror-sync`/`mirror-set`/`restart`/`history` don't trigger it. `history revert`
  especially: re-mirroring would instantly undo the revert, which is the precise opposite of the point.

Proven end-to-end: `add` on Deck A prints `auto-mirrored to Deck B: audacity (1 copied)` and Deck B has the
button *with its icon*; `remove` mirrors the deletion. `--no-sync` opts out (read off `argv` rather than
registered on all 30 subparsers — it's one cross-cutting concern, not a per-command option).

**Missing pages created.** `reading-kindle` and `vs-code` each had a Deck A page with no Deck B counterpart,
so those pages could never mirror. `create_page` does the surgery deliberately: a **fresh** UUID (a shared
page UUID makes every page lookup ambiguous — breaks `replace-at`/`remove-at` and therefore the wizard, and
across decks SD ignores the duplicate outright), the folder plus its `Images/` dir (icons resolve per PAGE),
an empty manifest, and registration in `Pages.Pages` (a folder SD doesn't know about is invisible).
`Pages.Current` is deliberately not moved — that's where her deck is parked, and yanking it to a blank page
would be a visible jolt from a background fix. `ensure_pages` runs BEFORE planning, so a Deck-A-only page
mirrors on the first run instead of needing a second.

#### Two bugs, found only because the feature got exercised for real

**1. `add` refused a genuinely free slot.** Its pinned check counted a Default-page **passthrough** as a
pinned button — while `dump` listed that same slot under *"EMPTY / PASSTHROUGH (safe to add)"*. The two
commands disagreed about the same slot, and since `restart` runs `populate-empty` (which seeds those
placeholders), the wrongly-blocked set **grows over time**. `is_passthrough_button` exists for exactly this
distinction and the rest of sdlib already uses it (`displace._free_slots`); `add` didn't. Two slots were
affected today — both created by the add/remove test that found it.

**2. The sync's pinned guard was too broad, and it cost real function.** Deck A *deliberately* overrides its
own pinned buttons with page buttons — `page-previous` at `0,3`, `SoftRebootDragon` at `0,2` — and refusing
to mirror those left **Deck B's new extra pages with no way back to page 0**. What the guard actually
protects against is §8.21 (a blank placeholder hiding a pinned button, which blanked 11 buttons on Chrome),
and that **cannot happen on the copy path**: `_views_of` skips passthroughs, so the source is always a real
button. It *can* happen on the delete path, so the concern moved there. **If A overrides its pinned button
at a slot, B must too — that is what "identical" means**, and `add --force` is the same override by hand.

The lesson both share: a guard copied from a real incident can outlive the shape of that incident. §8.21 was
about *blanks over pinned*; both of these generalised it to *anything over pinned* and started refusing
legitimate work. Guards should encode the failure, not the neighbourhood of the failure.

**Result: identical 442 → 455, different → 1, only-on-B → 0, only-on-A → 0**, idempotent, health clean. The
single remaining difference is `apps` `4,0` — a switch to Claude, which has no Deck B twin. A correct
refusal, and the only one left on either deck.

**137 tests.** Still open on §8: the wizard blank-slot "add group / adopt template" row (§8.10's third
surface, untouched); and multi-parent inheritance (§8.2, deferred at v1).

---

### 8.41 `parent` finally means what it reads like — a child IS its parent, plus its own stuff — **[2026-07-16]**

Jamie's ask, and the shape she wants:

> "we will likely have parent profiles and children profiles, but they will be slightly different than the
> current like template system. basically I might want an Instagram profile, and also have an Instagram
> messaging page profile and an Instagram landing page profile. and those two Instagram profiles would be
> children of the main profile. **and so they would inherit everything from it but then also have some of
> their own stuff.**"

The ledger already had `Node.parent`. It inherited **group membership** and nothing else — so a button she
drew by hand on Instagram stopped dead at Instagram. That is not what "parent" reads like to anyone, and the
gap is invisible until you look for a button that should be there and isn't.

**The model: an ancestor's own buttons are a synthetic group.** `"@node:instagram"` — source profile =
the ancestor's profile, membership implied by the parent edge, never present in `lg.groups`. Everything
downstream then worked with no new plumbing: `mid = "@node:instagram/4,0"`, freeze in `resolved`, the sweep,
per-button `excluded`. That was the whole point of the disguise — content inheritance needed a new way to
NAME a source, not a new mechanism.

Four things it had to get right, each of which is a way it could have silently half-worked:

* **`own_only`.** A node's "own" buttons are the ones with no `_mid`. Without that filter every inherited
  button arrives twice — once as its real group, once laundered through `@node:` — and the two copies fight
  for slots.
* **The whole chain, not just the parent.** A grandparent's buttons land on the parent as *managed* buttons,
  so reading the parent's uniques finds the parent's own work and none of the grandparent's. Each ancestor is
  read directly or inheritance silently stops one level short.
* **`effective` must contain the synthetic groups.** `_sweep` reads that set as "is this mid's group still
  wanted". Leave them out and every inherited button is classified `group detached` and swept straight back
  off the page it was just placed on.
* **`health` must not read `@node:` mids as orphans.** The real orphan condition is the *ancestor* going
  away, not the group's absence from `lg.groups`.

**`inherit_content` defaults to True**, which is only safe because of a fact that had to be checked rather
than assumed: **every parent in the live tree (`defaults`, `defaults-b`, `geck2`) is an ABSTRACT node with no
profile** — nothing of its own to pass down. So the default changed the meaning of `parent` for all 65 nodes
and the rendered output of exactly zero of them. A test pins that (`abstract ancestor contributes no content`),
because the day someone gives `defaults` a profile is the day the default silently becomes a big change.

It stays modular the same way groups do: `@node:instagram` in `excluded` drops the whole inheritance,
`@node:instagram/4,0` drops one button of it, `inherit_content=False` is the blunt form. `group_buttons` /
`set_button` / `attach` / `detach` / `checklist` all learned the synthetic form — without that, "inherit
Instagram except this one button" had no surface at all, which is *precisely* the case she described.

### 8.42 Creating a profile was five chores that didn't know about each other — **[2026-07-16]**

> "we don't really have a good system for actually creating a new profile. so when I go into a context and I
> say open context... from that and the open stream editor we need to be able to create a new profile and
> simultaneously link into a context."

Every piece existed. None of them knew about the others:

| chore | tool | what it didn't know |
|---|---|---|
| make the profile | `copy-profile` | the ledger existed |
| register it | `ledger new-node` | needed the profile to exist first |
| link the context | hand-edit `streamdeck_profile` | the name was typed, and typos are silent |
| the Deck B twin | nothing | so every new profile broke "B is identical to A" at birth |
| the switcher button | `vsd-ensure` | this is what the detector actually *presses* |

Miss any one and it fails **silently**: the profile is real, it just never appears. `new-profile` is all five,
in the order that works, with the context token validated *before* anything is created (a bad token used to
mean a half-built profile and a context pointing nowhere).

`delete-profile` is the inverse, because creation being one action makes deletion one action too. Its
leftovers would otherwise be exactly the failure shapes §8 exists to prevent — a node pointing at a profile
that no longer resolves is what *aborted a whole `health` scan* before §8.39. It refuses to strand children
unless `--force`, which re-parents them onto the grandparent rather than dangling.

**Three things testing it changed:**

* **`NEW_PROFILE_SOURCE` is `TEMPLATE` on both decks, not `GECK2 Home` on B.** The skill's rule predates B
  mirroring A: the twin arrived carrying GECK2 Home's buttons and the auto-mirror immediately deleted 13 of
  them to make it match A. Copying A's own twin lands it right and the mirror is a no-op.
* **`vsd-map` now warns on a NAME COLLISION.** The map is keyed by profile name, but a name is only unique
  *within* a deck — every mirrored profile is called the same thing on A and B, and §8.42 creates same-named
  twins by design. If the switcher ever holds a button for both, the second silently overwrites the first and
  every switch for that name targets one deck. **Latent, not live** (no context sets `streamdeck_profile_b`
  today) — but it fails silently and by name, which is the shape that hides. Not fixed, surfaced.
* **The Deck B twin is parented to the twin**, not across decks: a child of `instagram` on A becomes a child
  of `instagram-b` on B.

**The GUI, both surfaces.** The one question asked first is what the profile STARTS AS, because it is the only
choice that isn't a tick you can untick. In the context Miller the **recommended** parent is computed, not
guessed: if this context already inherits a Deck A profile from an ancestor (Instagram inheriting Chrome's),
that profile's node is almost certainly the right parent — she is specialising the thing she was already
getting. That is exactly the Instagram → Instagram Messages case.

**§8.10's third surface is now complete**: the wizard's blank-slot row already mounted the group checklist;
it gained the "adopt template" half (the parent picker), which §8.41 is what made meaningful — a parent now
hands down its own buttons, so picking one there is "make this profile start as that one". All three
authoring surfaces are the same `authoring.py` calls and the same composed Miller branches, not three copies.

**152 tests.** Still open on §8: multi-parent inheritance (§8.2, deferred at v1).

### 8.43 Making a group out of buttons you can already see — **[2026-07-16]**

> "they are all like media manipulation buttons. I want to turn those into their own group. which makes me
> think of **what would be the easiest way for me to create new groups?** so for example I see a group of
> buttons on a page and I realize I want that group of buttons on another page or multiple pages, what is
> easiest way for me to select that and modularize that?"

`ledger capture` already did the hard half — copy the buttons *and their icon files* into the group's source
profile, register the group. What it could not do is **create** that source profile: cloning lives in the
streamdeck.py monolith and sdlib must not import it, so capture told you to run `copy-profile` first.

That instruction was a trap with a scar already in the skill: **a group is EVERY button on its source page**,
and a fresh `TEMPLATE` clone arrives holding TEMPLATE's buttons. `G GECK2 Switching` was built exactly that
way, kept 16 buttons, and the first apply planned to scatter TEMPLATE's leftovers across every GECK2 profile.
So the missing step wasn't "create the profile" — it was "create it **and blank it**", which you had to know
to do by hand, and which failed quietly when you didn't.

`new-group-from` is the whole thing:

```
streamdeck.py new-group-from "Home" --slots "5,2 6,2 7,2 5,3 6,3 7,3" --name media
```

**ADOPT-BACK is the point, and it's why `--attach` defaults on.** Capturing from Home leaves Home's six
buttons exactly where they were but still *unowned* — the group exists and the profile it came from isn't
using it. Attaching + rendering makes render recognise them **by signature** and adopt them in place: nothing
moves, nothing is rewritten, and they become the group's. Only then is "modularized" true.

**The media group, live:** 6 captured, **6 adopted in place** on Home, and the auto-mirror + membership mirror
put it on Deck B's Home too — B's copies were deleted and re-placed as B's *own* managed buttons, which is
§8.37's rule working (a `_mid` must never be forged across decks). Both decks now show `media/5,2`…`media/7,3`.

A dry-run onto Youtube is the whole feature in one output: its 5,2–7,2 are the *same three buttons* and were
**adopted**; its 5,3–7,3 are occupied by Jamie's own, so those three are correctly refused — the §8.4
conflict/displace case, surfaced rather than trampled.

**`ledger capturable`** backs the picker. The GUI must not invent its own idea of what can be captured — that
is how two rules drift and the picker starts offering slots capture then refuses — so it applies capture's own
tests (a passthrough is not a button; an already-managed button belongs to another group) and reports the
reason with the row. Un-capturable rows are **shown, greyed, with the reason**, because "why can't I pick that
one" is a question the screen should answer on the spot.

**The GUI** is pick-profile → **tick the buttons** → name it. Multi-select is an in-line toggle list, not a
text box of positions: the set is finite and known, and typing `"5,2 6,2 7,2 5,3 6,3 7,3"` by eye is the exact
tedium the feature exists to remove.

### 8.44 The VSD map's name collision, fixed rather than warned about — **[2026-07-16]**

§8.42 surfaced this as a warning and left it: `vsd_profile_positions.json` was `{name: pos}`, but **a profile
name is only unique within a deck**. Every mirrored profile is called the same thing on A and B, and §8.42's
twins are same-named by design — so the moment the switcher held a button for both, the second silently
overwrote the first and every switch for that name went to one deck.

The deck is the missing half of the key, so it belongs **in** the key: `{"A": {name: pos}, "B": {...}}`.

What the fix turned up: **`geck2-switching`'s source profile (`G GECK2 Switching`) lives on Deck B**, and the
ledger Miller's "edit this group" looked its position up by name alone. Deck-keying it without passing the
group's `source_deck` would have broken editing that group — and `vsd-ensure` didn't even accept `--deck`, so
it resolved a twin's name to Deck A and would wire the wrong deck's button. Both fixed.

AHK reads both shapes: a pre-2026-07-16 flat map is read as Deck A (which is what it meant — nothing ever
referenced a Deck B profile while that format was live), so a stale file on disk degrades to the old behaviour
instead of resolving nothing.

**152 tests.** Still open on §8: multi-parent inheritance (§8.2, deferred at v1).

### 8.45 Chrome split: the clean parent, and the rename that would have moved 35 buttons silently — **[2026-07-16]**

> "we create a new chrome profile that is currently just a copy of the base template. and then what we can
> consider chrome currently we will call chrome navigation. and that will be a child of chrome. very few chrome
> pages will actually end up using that navigation stuff. and I want any new profile I create to actually be
> clean children of chrome but not deal with all that stuff."

The first real use of §8.41, and it needed two tools that didn't exist: **`rename-profile`** and **`ledger
rename-node`**.

**A name is an identity in more places than the manifest**, and each omission fails silently in its own way:
the ledger node's `profile` (a node pointing at a name that no longer resolves is what *aborted a whole health
scan* before §8.39); the contexts' `streamdeck_profile` (resolves to nothing, stops switching, says nothing);
and for a node, its own key plus every child's `parent` plus the `@node:` content groups embedded in
descendants' **frozen mids** — miss those and health reports the whole inherited set as stranded.

**THE TRAP, and the reason this section exists.** Switch-Profile buttons target a profile by **UUID**, which a
rename doesn't change. Usually that's exactly right — the buttons follow the thing you renamed. Here it was
catastrophic-quiet: renaming Chrome → Chrome Navigation silently turned **35 profiles' "Chrome" switcher into
a "Chrome Navigation" switcher**, and the only way to notice would be to press one. So `rename-profile` walks
every page on both decks and NAMES the switchers that now point at the renamed profile. It does not repoint
them — only the person renaming knows which of the two they meant — but it never lets the case be silent.

**I THEN REPOINTED THEM, AND THAT WAS WRONG.** I read "few chrome pages will use the navigation stuff" as
"so the plain `chrome` context should get the clean profile", pointed all 57 switchers + the `chrome` context
+ Claude's pinned button at the new Chrome, and told her so. Her correction:

> "chrome navigation should replace chrome in every instance that existed before including like the stream
> deck page and the context switching. the new chrome is more just for me to have as a parent to create
> children from. functionally it should be like identical to how it was before."

The clean `chrome` is a **parent, not a destination**. Nothing points at it; it exists to be inherited from.
"Few pages need the nav stuff" was about the pages she'll create NEXT, not about what today's Chrome does.
The whole restructure was supposed to be invisible from the deck — I turned it into a behaviour change.

Reverting was the same one edit in reverse, which is the §8 thesis paying off either way: the shared switcher
is one button in one group source, so both the wrong move and its undo were `batch-add ... --overwrite` +
render, and 55 profiles followed. **36 (A) + 29 (B) switchers now target Chrome Navigation — every one, Claude
included.** `vsd-sync` then reclaimed the clean Chrome's switcher button on its own: no context references it,
which is exactly right for a parent.

**The exclusion inverts with the target, which is the genuinely interesting bit.** `profile-switching/3,0` is
excluded on whichever profile it POINTS AT — a switcher to yourself is dead weight. That's `chrome-navigation`
(it's the destination), while the new `chrome` carries the normal four. Same mid, opposite meaning on two
nodes, decided entirely by where the button goes.

Notes from checking rather than assuming:
* **`Claude`'s pinned 3,0** doesn't follow renders — it isn't a ledger node, so nothing manages it. Both the
  repoint and the revert had to touch it by hand (`ProfileUUID` **lowercased** — the §7 scar). One profile
  quietly disagreeing with the other 35 is the exact drift the ledger exists to kill, and it's invisible
  because it lives on the Default page.
* 7 switchers still name Chrome Navigation from the Default pages of hidden `G *` group sources: unmanaged,
  never read (a group's buttons come from its CURRENT page), invisible. Left alone.
* `TEMPLATE` **is** a node, so it moves with the rest — future `new-profile` copies are born correct.
* `chrome-navigation`'s own page 3,0 is her Hotkey, covering the pinned switcher on page 0. Pre-existing and
  deliberate — render.py already calls that out as "a real override, leave it".

**Result — functionally identical to before, plus a parent.** `chrome-navigation` IS the old Chrome: same
UUID (432121E7), same page, same 20 buttons, and the `chrome` context points at it exactly as it always did.
`chrome` is clean (0 page buttons, the shared defaults, 18 free slots) and reachable by nothing — a template
in the §8.41 sense, waiting for children.

**AND THE AUTO-SWITCH BROKE — from §8.44, not from any of this.** Both Chrome and Youtube stopped switching.
The always-on detector is a LONG-RUNNING AHK process: it had the pre-§8.44 flat-map reader in memory, the map
on disk was now deck-keyed, so `m.Has("Youtube")` was false for every profile. `ahk_event.log` said so
outright — and the giveaway was that the error text lacked the "Deck A" wording §8.44 added, proving the code
in memory was old. Reloading AHK fixed it.

**The lesson is about the SHAPE of the change, not the bug.** Changing an on-disk format that a resident
process parses is a live migration, and the resident half doesn't update when the file does. The new AHK reads
both shapes (a flat map is read as Deck A), so the *next* such change self-heals — but that tolerance only
helps code that's already running it. A format change wants its consumer reloaded in the same breath, or the
window between them is silent breakage that looks like something else entirely.

Also fixed here: **`ledger <cmd> --no-sync` was rejected by the ledger sub-parser on every command**. The
dispatcher read the flag for the autosync decision but never stripped it from the argv it handed to argparse,
so a flag the top-level parser supports killed every `ledger` command that used it.

**152 tests.** Still open on §8: multi-parent inheritance (§8.2, deferred at v1).

### 8.46 Every new GUI entry point shipped broken — I tested the engine, not the wiring — **[2026-07-16]**

Jamie tried to make `chrome-plex` from the Context Manager and got "a really long tooltip that I could not
read because it went off the side of the screen and then nothing really happened." `ahk_event.log`:

    streamdeck.py: error: unrecognized arguments: --actor voice

`--actor` was a **top-level** argparse option, legal only BEFORE the subcommand. The AHK wrote it after. All
**three** new entry points from §8.42/§8.43 — "+ New profile" in both Millers and "+ New group from buttons" —
had it, so none of them could ever have worked. The `ledger` sub-CLI escaped only because ledger.py owns a
parser that declares `--actor` itself.

**The miss is the lesson, not the flag.** Every command was tested from the shell, where I never passed
`--actor` — so I verified the engine and inferred the wiring. The one caller that always passes it is the one
I never ran. §8.38's line applies to me here: *a checker that shares the code's assumption cannot catch the
code's bug.* Testing `new-profile` from bash tests bash, not the button.

Two fixes, at two depths:

* **`--actor` is now stripped from argv wherever it appears**, exactly like `--no-sync` (which was already
  immune, having been designed this way after being bitten). An option that means the same thing wherever it
  appears should PARSE wherever it appears. Position-sensitivity is a trap laid specifically for callers who
  build a command as one flat list — i.e. every AHK caller — and it fails LOUDLY in a shell and SILENTLY in a
  GUI. Verified both orders, and that `[voice/bulk]` still lands in the history log.
* **`SdPyErrLine`** — a failed streamdeck.py call now surfaces ONE line. Raw argparse stderr is a full usage
  dump listing all 40 subcommands; piped into ShowTooltip it rendered as a single enormous line off the edge
  of the screen. The real message was on it, and unreadable. **A tooltip that can't be read is the same as no
  error** — which is why this presented as "nothing happened" rather than as a bug report. It prefers the line
  containing `error:` (argparse always puts the complaint there), else the last non-empty line, collapses
  whitespace, caps at 180 chars. The log still gets everything.

Also fixed here: **`ledger <cmd> --no-sync`** was rejected by the ledger sub-parser on every command — the
dispatcher read the flag for the autosync decision but never stripped it from the argv it passed to argparse.
Same family of bug, found the same afternoon, in the path I *did* run.

### 8.47 The kill with no counterpart, the squatters on 4,0, and landing where you edit — **[2026-07-16]**

Three complaints from one session of Jamie actually USING §8.42, and all three were the tools being right in
isolation and wrong as a workflow.

**"I have had to restart my stream decks so many times... the whole thing just crashes afterward until I
rebooted."** It was never a crash. `safe_write` FORCE-KILLS Stream Deck before every write — the §7 protection
against SD flushing stale cache over the edit on exit, and correct. What was missing is its other half:
**nothing ever started SD again**. Every write left both decks dead until she ran `restart` by hand, and the
skill's own rule ("the script never restarts automatically") had made that documented behaviour rather than a
bug. That rule was written for a CLI where *I* make ten edits and restart once. It is exactly wrong for a GUI,
where every action is atomic and the person pressing the button is looking at the deck.

`_autorestart` now fires on both dispatch paths (top-level AND `ledger` — the one the Millers actually use, so
omitting it would have fixed this for me and not for her). The trigger is precise rather than a command
whitelist: `WRITTEN_PROFILES` is populated *by safe_write itself*, so it restarts **iff we actually killed the
deck and wrote**. A read restarts nothing. `--no-restart` batches. The rule reads: *if we killed it, bring it
back* — a tool that kills a process to do its job owns restarting it, and "run restart afterwards" is not a
design, it's a chore delegated to the user.

**"there is buttons at 4,0 and 5,0 and I don't really know why they are there or what for."** Nobody would:
they were an accident of copy order, and they were MY bug. Measured: **41 profiles have F13 at its home 1,2;
exactly 2 had it at 4,0 — `chrome` and `chrome-plex`, the only two ever made by `new-profile`.**

TEMPLATE's Default page is exactly the 12 buttons of `profile-switching` + `other-defaults` at their home
slots — but **2 of the 12 carry no `_mid`**: 1,2/1,3 are pre-ledger hand-made "Scroll Up"/"Scroll Down" that
the group never adopted, because adopt matches on signature and the group's versions are labelled "F13"/"F14".
So every new profile inherited two unowned squatters on the group's home slots, `_uniques()` correctly refused
to touch them, and F13/F14 flowed to the first free slots: **4,0 and 5,0**. The worst possible landing — the
skill's own layout rule says "most important action goes at 4,0" — and what landed there was a **duplicate**:
F13/F14 are `CenterScrollUp`/`CenterScrollDown`, the exact same keys the "Scroll Up"/"Scroll Down" at 1,2/1,3
already sent (vk 124/125). Two dead buttons squatting on the prime slots of every profile she'd ever create.

`_clear_squatters_on_pinned_homes` runs before the first render: on the Default page, delete any button that
is unowned AND sits on a slot a pinned group owns. Narrow on purpose — a real button of hers is never at risk,
because on a brand-new profile there is no such thing; everything on that page came from the template. Both
existing profiles repaired (unfreeze the mid, drop the flowed copy, re-render). **All 43 nodes now place F13 at
1,2**, and chrome-plex went from 14 pinned / 18 free to 12 pinned / 20 free — 4,0 and 5,0 handed back.

The general shape: **a copied profile is scaffolding, not content.** The ledger owns the buttons. Anything the
clone brings along that the ledger also owns is a squatter, and `resolved` FREEZES the first placement — so a
bad first render isn't self-correcting, it's permanent until something unfreezes it.

**"after creating a brand-new profile it should automatically take me to the spot to edit it... currently I
create it and then I have to open the new user interface and blah blah."** `OpenStreamDeckLedger(path)` now
takes an internal path and both creation surfaces land on the new node's group checklist. It mirrors
`OpenContextEditor`'s `deep` flag and needs the same trick for the same reason: `LaunchMillerViewer` only
ACTIVATES an already-open viewer, so a deep open must render in-process rather than ask the launcher for a
window that exists and won't re-navigate. Creating a thing and choosing what's on it are one intent; making
her navigate back to what she just made is the tool forgetting what she was doing.

**152 tests.** Still open on §8: multi-parent inheritance (§8.2, deferred at v1).

### 8.48 The checklist Jamie actually wanted — nested, per-button, and named — **[2026-07-16]**

Five asks from using it, and one bug that §8.47 had just introduced.

**"I just added the media and it says that it failed even though it seems to have worked."** It HAD worked.
`_SdLedParse` took the LAST stdout line as the JSON payload, and §8.47's auto-restart had started printing one
line after it. So every `--json` Miller call now reported FAILED for commands that succeeded.

The invariant was mine and I'd even written it down in `cmd_new_profile`: *"LAST — nothing human may be printed
after this point."* It survived exactly one new print statement. That's the tell: an invariant every future
writer must remember is not a design, it's a trap with a comment on it — and the cross-cutting hooks
(auto-mirror, auto-restart) print BY DESIGN and cannot each know they're last. So the parser got robust
instead: scan bottom-up for the last line that is a bare JSON object. Trailing noise is now simply skipped, and
the two other hand-rolled last-line parsers in ContextEditorMenu were routed through it.

**"I don't want to see all of the B versions of everything... we can just hide all of those except for the
exclusive be ones."** Twins are hidden from the profile list (`<node>-b` with a Deck A node of that name), with
a greyed "(28 Deck B twins hidden)" row saying why. The GECK2 profiles are Deck-B-ONLY, have no Deck A
counterpart, and stay — they're the only things on B she edits. Since §8.40 a twin's content is a *consequence*
of its Deck A node, not an edit surface, so this is the list finally matching the model.

**"the groups need a lot of work. for one they should be like nested."** Sections come from the TAGS — the
`list-navigate` tag's members are already exactly those three groups, so the structure she wanted to see was
structure the ledger already had and simply wasn't rendering. Deriving sections from a name prefix instead
would have been a second, silently-diverging definition of the same fact (and would have folded
`profile-switching-b` in with `profile-switching`).

**"the default should be like accept all which is the top row and then the individual ones... underneath."**
The shape is now identical at every depth — an "all" row, then the parts. Enter drills, `2` toggles the whole
thing without drilling: the common act is "give this profile the nav buttons", and it shouldn't require
walking into a level you didn't want. "Turn all on" for a group also clears INDIVIDUAL exclusions — otherwise
"all on" would leave the buttons she'd removed one-by-one still missing, which is not what "all" means.

**"ideally all of the buttons should be labeled very easily and clearly both by location and by name."** This
was the real work. **Most buttons carry no Title** — icon-only is the convention here (3,217 have ShowTitle
off) — so a checklist showing Title alone is a column of blanks with the position as the only handle.
`describe_button` derives the name from the ACTION:

    profile-switching   0,0 -> Home      1,0 -> Apps       2,0 -> Navigation   3,0 -> Chrome Navigation
    other-defaults      0,1 CloseWindowOrTab   1,1 Alt+Tab   2,1 Ctrl+Alt+N    1,2 F13
    media               5,2 Mute   6,2 Volume Down   7,2 Volume Up   6,3 Play/Pause
    list-navigate-numbers   4,3 ONE   5,3 TWO   ...   7,2 EIGHT

Two things it had to get right. **NativeCode, not just VKeyCode**: SD fills them differently depending on
origin — a helper-written hotkey sets VKeyCode, one made in the SD UI can leave it -1 and set only NativeCode.
The F13/F14 buttons on every profile are the second shape, so a VKeyCode-only reader would call the most
widespread button on the deck "hotkey" and stop. And **the multimedia map is verified, not guessed**: each
icon was extracted from the live deck and looked at (mute is 4; 5/6 are up/down — not an order anyone would
predict). `actionIdx` 3 is deliberately absent: nothing on either deck uses it, so there's nothing to check a
guess against, and a plausible guess would be indistinguishable from a fact.

Fixing this also surfaced a §8.41 bug nothing had ever run: `group_buttons` on a content group did
`dict(inherited_sources(...))`, which raises — those are 3-tuples. Tested `set_button` on content groups, never
`group_buttons`. Same shape of miss as §8.46: the engine was tested, the wiring inferred.

`SPECIAL_KEYS` moved to `sdlib/constants.py` (with a `KEY_NAMES` inverse) so sdlib can name a key without
importing the monolith — the layering rule held.

**"open stream should automatically take me to the profile/context that I'm on."** It resolves the current
context chain to its Deck A profile, deepest-first exactly like VsdSwitcher, and opens there. The node is
resolved in the DISPATCHER process and passed as a command-line arg, because the viewer is its own process and
only includes the Stream Deck half.

**152 tests.** Still open on §8: multi-parent inheritance (§8.2, deferred at v1).

### 8.49 The nested checklist shipped broken, and the test that now catches it — **[2026-07-16]**

§8.48's rebuild rendered an EMPTY level. Two bugs, both invisible to `ahk.py validate`, both surfacing as
"the groups are no longer showing" with no error anywhere:

* **An IIFE split across two lines.** `((nm, rs, a) => ...)` on one line and `(node, rows, allOn)` on the next:
  AHK does not apply that as a call, so `MlActions` received the un-invoked lambda and the enclosing `MlBranch`
  threw "Expected a String but got a Func" — pointing at a line that was fine. Every working example in the
  file keeps the IIFE on one line; I broke the pattern to fit a line width.
* **`_RegDivider` lives in RegistryEditorMenu.ahk**, which `Scripts/StreamDeckLedgerViewer.ahk` does not
  include. In AHK v2 an unknown function name reads as an unassigned VARIABLE, so it threw at runtime and only
  in the viewer — under MAINFUNCTIONS (where the Context Manager runs) the same code worked. All four uses were
  mine; the viewer's own header even warns about this exact class ("fails at runtime, not at validate"). Fixed
  with a local `_SdLedDivider`: six lines beat a cross-menu dependency for a horizontal line.

Diagnosis was a headless probe that built each level inside try/catch — 30 seconds to the real error after
guessing had produced nothing. **The Miller swallows a builder throw and renders an empty level**, which is
what makes this class so quiet.

**The test that should have caught it, and why it didn't.** `test_stream_deck_ledger_menu.ahk` validates the
ROOT ONLY — its own comment says "shallow ... no drilling" — and the bugs were one level down. `MlValidateTree`
has always taken `Map("deep", true)`; nothing used it.

Turning it on in the window fixture **failed**, and the failure was the useful part: deep drills *every* root
branch, and Conflicts / Deck A vs Deck B / History each scan the real decks or shell git regardless of ledger
size, so the fixture blew even a 12s window-open wait and both window tests failed with a misleading "didn't
open in time". Deep-from-root is not affordable anywhere in this menu.

So the check went in the LEAF fixture — which already points the ledger at a scratch file — **rooted at the
checklist subtree** rather than the menu root. One group, one drill, under a second, and it covers exactly the
surface §8.48 rebuilt. Crucially the leaf fixture's include list mirrors the lean standalone viewer, which is
the only reason it can catch a `_RegDivider`-shaped dependency at all.

**Then I checked the guard by re-breaking the code**, because a green test proves nothing until you have
watched it go red: with `_RegDivider` put back, `sd_ledger_leaf_every_level_builds` fails alone and names the
row — `root / tog:zzleafgrp: children builder threw`. That is three bugs in one day (§8.46 `--actor`, §8.48
`group_buttons`, this) with the identical shape: **the engine was tested from a shell, the wiring was
inferred.** The GUI suite is manual (~90s, not in the Stop hook), so it is on me to run it after touching a
menu — that is the actual lesson, and it is now the first thing in the skill's Miller section.

**One deliberate behaviour change fell out.** A group row is now a BRANCH: Enter drills to its buttons, and
arrowing onto it previews them in the right pane — which is Jamie's "when hovering a group I want to see all
of the individual buttons". A branch cannot also be a leaf, so the whole-group toggle moved from Enter to row
action `2` (and Enter-Enter still does it, via the "Turn all on/off" row inside). The existing leaf test caught
that the instant it landed, which is precisely what it is for.
