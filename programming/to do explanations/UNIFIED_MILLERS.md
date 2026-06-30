---
tags: [design, principle, miller, architecture]
related: ["[[CONTEXT_MANAGER]]", "[[VOICE_COMMAND_SYSTEM]]"]
status: principle
updated: 2026-06-29
---

# Unified Millers — the consolidation principle

**The problem:** the system grew scatterbrained — each capability got its own little tool/GUI
(`AddContext`, `AddProgramByVoice`, `AddListNavSite`, ad-hoc forms, hand-edited JSON). Too many
front doors for too few real domains.

**The principle:** collapse them into a SMALL set of **unified Miller editors**, one per domain.
Each is the single home for its domain; new capabilities mount as **branches/nodes inside the
right one**, never as a new standalone tool.

## The Millers (the domains)

| Miller | Domain | Voice |
|---|---|---|
| **Registry Editor** | dictation value-lists (sites, docs, contacts, directories — key→value) | "open registry" |
| **Context Editor** | app/page profiles (the 9 context facets) — [[CONTEXT_MANAGER]] | "open context" |
| **Voice Command Editor** | function ↔ voice command join | "show fun" |

Siblings, not rivals: they share machinery and **cross-mount** — Registries mount inside "show fun"; the
**Context Manager** (`_ContextEditorAsNode`, key `ctxmgr`) now mounts inside BOTH "show fun" and the Registry
Editor (built 2026-06-26), folding every context into each host's recursive search. Same look = consistency,
not redundancy. The boundary is the data shape (flat value-lists vs rich profiles vs command rows), not the UI.

## What "unified" buys (and the conventions that make it work)

- **Shared machinery** — `MlLeaf/MlBranch` nodes, `_MillerColumnPickGui`, recursive search, and the
  unified **placement control** (`MillerPlacementNode` + a per-system `MillerPrefBackend` adapter; prefs
  `set-pref` scope per domain). Build a new editor by composing these, not reinventing.
- **Deep-links are the unifier** — every node is addressable by `initial_path`, so a thin voice phrase
  opens the right Miller landed on the right node ("open nav" → Context Editor at the ListNav node).
  This is what lets us delete standalone tools: they become *entry points*, not separate apps.
- **Add == Edit** — a facet/entry absent shows `+ add`; present shows its value. Same path either way,
  so "everything goes through here" holds without extra code.
- **One reuse rule:** if you're about to write a new management GUI, ask which existing Miller it's a
  branch of. If truly none, it's a new domain — and it still composes from the shared machinery.

## Portable nodes — the cross-Miller composition convention (Jamie 2026-06-26)

The goal: drop a node-cluster from one Miller into another and have it FULLY work — e.g. the
Voice-Command actions (rescope / remove) usable from inside the Context Manager's "scoped commands"
view. A node's *data* is already portable (it's a Map; splice any builder's `[nodes]` in). The only
thing that doesn't travel is **what to refresh after acting**. So the convention:

A reusable cluster is `_XActionNodes(data…, reopenFn) -> [nodes]` that:
1. **closes over its data** (phrase/fn/token), not the host's globals;
2. **takes a host-supplied `reopenFn`** — the caller's refresh ("reopen MY menu here"), so the same
   node refreshes whatever Miller it's embedded in;
3. **mutates via the owning subsystem's CLI** (host-agnostic logic, single source of truth).

**Reference implementations (copy either shape):**
- Placement/fav-hide is now the engine's `MillerPlacementNode` + a per-system BACKEND ADAPTER
  (`_RegEnsurePlacementBackend` in RegistryEditorMenu.ahk registers get/set/clear via `MillerPrefBackend`).
  This SUPERSEDED the old hand-rolled `_RegPrefCycleRow`/`_RegFavHideNodes` rows (deleted 2026-06-29).
- `_CmdActionNodes(phrase, fn, reopenFn, opts)` (`Helpers/CommandActionNodes.ahk`) — the per-voice-command
  Remove / Rephrase / Rescope rows. Built 2026-06-26 as the first DELIBERATELY-shared cluster. The Voice
  Command Editor's `_VceCommandActions` delegates to it (`reopenFn = reopen show fun at this fn`); the
  Context Manager's scoped-command rows drill into it (`reopenFn = reopen this context's scoping view`,
  `opts.defaultContext = the viewing context`). Mutates only via the generic-store CLI, so the same row
  edits a command identically from either Miller. This was the migration of the old anti-pattern below.

Anti-pattern (now resolved): a cluster that hardcodes `OpenXEditor()` for its refresh — `_VceCommandActions`
used to reopen the Voice Command Editor directly. Migrated to `_CmdActionNodes` taking `reopenFn`, so it's
droppable into any Miller. Any future cross-Miller cluster follows the same `reopenFn` shape.

## Shared-engine backlog (eventual — Jamie 2026-06-26)

Fix once in `_MillerColumnPickGui`, every Miller benefits:

1. **Navigation memory for deep-links.** The engine already restores the cursor per level on MANUAL
   back-out. The gap: a `initial_path` jump (e.g. "open context" → current context nested) drills the
   parent levels programmatically without seeding their selected row, so backing out lands on row 1.
   Fix: when applying `initial_path`, set each traversed level's remembered selection to the child it
   drills into (symmetric with manual nav); key that memory by chain path so it survives within a session.
   **DONE + tested (2026-06-26):** `_Mcp_NavigateToPath` seeds `sel_memory` per traversed level (added
   `_Mcp_FindIndexByKey`); regression test `miller_initial_path_seeds_nav_memory`.
2. **In-place refresh.** Today an action does `close.Call(); OpenX(deepPath)` — rebuilds the whole window
   (flicker + lost position). Add an engine `refresh` primitive: re-run the CURRENT level's children
   builder and redraw, keeping window + cursor. Handlers call `refresh()` instead of close+reopen.

   **DECISION (built 2026-06-26) — seam (b), a process-global hook, NOT a 4th `do` arg.** The engine
   ALREADY re-renders in place after a row-action whose handler doesn't close (clear `catalog_cache` +
   `search_universe`, `SetTimer(_doRender,-50)`, cursor preserved). We exposed exactly that as a
   process-global the clusters prefer over close+reopen — no `do` signature change, so every existing
   handler is untouched:
   - `MillerActiveRefresh()` → if a live Miller is showing, clears its caches + re-renders the CURRENT
     level (keeping window + cursor) and returns `true`; returns `false` if none is open (or it already
     closed — the hook self-guards on `state["closed"]`).
   - `MillerRefreshOrReopen(close, reopenFn)` → the cluster idiom: `if (MillerActiveRefresh()) return;`
     else fall back to the old `close()`+`reopenFn()`. The engine pushes its in-place hook on open and
     restores the previous on cancel.
   Migrated callers: `_RegPrefCycle` / `_RegPrefSet` (fav/hide), `_CmdRemove` / `_CmdRescope`. `_CmdRephrase`
   stays close-first (its wizard wants the Miller gone). Net effect: cycling a fav, removing/rescoping a
   command, etc. refresh the host engine in place — and **this is also what removes the mount edit-reopen
   seam** (editing a context from inside "show fun" now refreshes "show fun", never pops the standalone
   Context Manager). Caches are cleared wholesale, so a later Left back-out rebuilds the parent list and
   shows any fav/hide reorder.

3. **Unified placement prefs + composable systems** (the big one — full spec below). Lift the fav/hide
   cycle out of the Registry host into an engine primitive every Miller gets for free, generalize it to
   Projects' local/global scope, and make ANY node (a whole system OR a single sub-branch) plug-and-play
   insertable into any other Miller without reworking the original.

**Forward-compat:** the portable-node **`reopenFn`** (above) is the exact seam for #2 — today it's
close+reopen, later it's refresh-in-place, and **no node-cluster changes**. So building clusters with
`reopenFn` now is the right prep; the engine upgrade flows through them for free.

## Placement prefs + composable systems (spec — Jamie 2026-06-26)

The problem this kills: favorites/hide is reimplemented per Miller (Registry's `RegPrefs`, Projects'
Python bands, Spotify's `personal_status`), so Jamie keeps re-wiring it in each new menu; and mounting
one Miller inside another is hand-rolled per `_FooAsNode`, with edits popping a standalone window. Make
both **engine-level and automatic**.

### Placement preference — the data model

Named axes, semantic-keyed (survives across every place the item appears):

```
prefs[system_id][item_key][axis] = { dir: "top" | "bottom" | "",   target: "local" | "root" | "<page_key>" }
```

- **axis (decided 2026-06-29)** — keep an axis layer from day one (default axis `"placement"`). One pref per
  item was rejected: Projects already needs TWO independent things ("current" + "favorite"), and a flat
  `{dir,target}` can't express both. The `[axis]` layer lets any system carry N independent placement
  directives and lets Projects' current/favorite eventually run through the SAME engine — and "add an axis"
  never touches stored-data shape.
- **dir** — `top` pins, `bottom` **sinks** (UI word is "sink", NOT "hide", to stay distinct from the
  engine's separate Ctrl+R layout true-hide). **There is no true removal** — sink just pushes to the very
  bottom; nothing ever disappears (Jamie 2026-06-26).
- **target** — *which level the float applies at*, **contextually editable** (global is not always the
  system boundary):
  - `local` = top/bottom of the item's **own immediate level**, wherever that level renders.
  - `root` = promote to the top/bottom of the item's **system root level** (the default "global"). Bounded
    by the system: a `root`-pinned project floats to the top of the **Projects** subtree, NOT above the
    mount point into a host master. Global never escapes its `system_id`. A `root` (or page) promotion
    shows the item at that target and **leaves its home-folder order normal** (decided 2026-06-29 — matches
    Projects' global favorites, which appear in the root band without reordering their workspace).
  - `<page_key>` = promote to a **specific page** — identified by that page's **stable node key**, NOT a
    positional chain-path (decided 2026-06-29: a path breaks on restructure; a key survives, consistent
    with semantic keying everywhere else). If that page key never renders in the current mount, the pref
    just no-ops there.

**Keying is semantic** (`system_id` + stable `item_key`), not by `layout_id` — so a fav set in the
standalone Miller is the same fav when the item is mounted elsewhere. **Invariant (decided 2026-06-29):**
`item_key` must be STABLE (a rename must not orphan the pref) and UNIQUE within its `system_id` (the pref
has no level qualifier — same key at two levels would share one pref). Assert it when converting each
system. Backends are **pluggable**: default is one engine JSON store (`miller_prefs.json`), which is the
**canonical** store; Projects (Python bands) and Spotify (`personal_status`) register their own get/set as
**transitional adapters** (don't rip out working code now; migrate onto the engine store later once the
model proves out — not two permanent code paths by accident).

### The control — one button that is a node

A single placement row per item (label shows current state, e.g. `Placement: ★ top · global`); **drill in**
and the choices are pickable rows — top/bottom × {this level, system root, a specific page…} + clear.
"One button" satisfied; "every action is a visible, pickable row" satisfied; **adding an option later = add
a row** (extensible by construction). Optional focused-row hotkey cycles the most-common axis in place.

### Engine mechanics (free for every Miller)

At each level render the engine: walks the chain to the nearest ancestor carrying a `system_id` (that's the
current system + the pref scope); partitions the level's rows by `prefs[system_id][rowKey][axis]` — `local`
sorts within this level, a `root`/`<page_key>` target matching THIS level floats/sinks here; and at the
level that came straight from a system's `rootBuilder` (tagged "system root") — or whose node key matches a
`<page_key>` target — prepends/appends the promoted items. Promoted rows are **materialized by an explicit
`resolve(item_key) -> node`** on the `MlSystem` (decided 2026-06-29), so a promoted band drills exactly like
the real one; the system's `search_index` is only the FALLBACK resolver. Systems with neither get `local`
only until they add one.

### Composability — whole systems AND sub-nodes, retrofittable

Formalize "a mountable thing" as a first-class object instead of hand-written `_FooAsNode`:

```
MlSystem(system_id, label, detail, rootBuilder, opts)   ; opts: search_index, pref backend, enabled axes
```

- **Standalone == mounted, same code.** The standalone GUI becomes `OpenMiller(system.root)`; mounting is
  `parent.push(system.asNode())`. Both call the *same* `rootBuilder`, so there is no second window to pop
  to — **this deletes the edit-reopen seam by construction** (an edit calls the engine refresh hook (#2),
  which re-runs the current builder whether standalone or mounted).
- **Sub-nodes are equally insertable, after the fact, with no rework** (Jamie 2026-06-26: "if I want to
  put the bands sub-node into another GUI I should just insert it"). The enabling rule, enforced for all
  authoring: **every node's children-builder is a pure function of the data it closes over, never of where
  it sits in the tree** (no `chain[1]`-is-X assumptions). Then any branch — system root or a single Spotify
  band — is droppable anywhere. Its favorites travel because the node carries its **origin `system` id +
  stable key**; mounting a whole system (root node stamps `system_id` on its subtree) and mounting one
  sub-node (already carries `system_id` from its origin builder) are the *same* mechanism.

The scaffolder (`new_miller.py`) emits this shape by default; the 5 existing `_FooAsNode` systems convert
to it incrementally (each conversion is local and behind the same UX). **Guard the composability rule with
a test (decided 2026-06-29):** when converting a system, add a case that mounts it under a dummy root and
confirms it still works — the cheapest defense against someone later baking in a `chain[1]`-is-X assumption.

### Authoring & discovery — registry, export units, JSON deferred (decided 2026-06-29)

Primary goal (Jamie): a future Claude session should **whip up a new Miller by clicking together existing
pieces — including parts grabbed from old systems — WITHOUT having to understand the old system.** `MlSystem`
+ pure builders solve the *mechanical* side (once you hold a node, it drops in). These three add the
*discovery + extraction* side:

1. **Export unit** — the one grabbable unit is a pure node-producer `XxxNode(minimalData) -> node` (and
   `MlSystem` for whole systems): self-fetching, closes over its data, assumes nothing about where it sits.
   It is the ONLY thing another session ever calls — so grabbing "the Spotify bands" never touches scraping
   or catalog internals. Bake the export-unit shape + a doc annotation in DURING each `MlSystem` conversion
   (same pass), not as a later cleanup.
2. **Node registry (the linchpin)** — every `MlSystem` and annotated export unit **self-registers** into a
   global table at load (dumped to JSON). It is NOT a hand-maintained duplicate — it's a reflection of what
   exists, so it can't drift. One source, many consumers: the scaffolder's `--mount <ids>`, a future
   "browse mountable nodes" Miller, qmd indexing, AND the name→function resolution table a JSON layer would
   need. Build this as part of the foundation — wanted either way. **Registry vs qmd are complementary, not
   redundant:** qmd = fuzzy *discovery* ("is there a node that does X?"); registry = authoritative
   *reference/resolution* (stable id → real provider, exact input, mount example). The registry is also
   indexed by qmd.
3. **JSON authoring layer — DEFERRED, but designed-for.** A hybrid JSON Miller (JSON describes structure +
   wiring; references named providers/functions for anything dynamic — never pure-JSON) would enable
   no-code / voice / GUI / runtime menu-building. Its KILLER feature is non-code authoring, NOT Claude
   authoring (Claude writes AHK fine; code + registry already gets ~80% of the plug-and-play win for it).
   Since **Claude will author basically all Millers and Jamie only does minor reorder edits** (already
   covered by the layout/rearrange system), JSON is **low priority**. Decision: make the registry the
   resolution table NOW so JSON drops in cleanly later (forward-compat, same move as reopenFn→refresh), but
   do NOT build the interpreter until the registry + conversions prove out. Revisit only if voice/GUI
   menu-building becomes a near-term want.

### Build order

Rides on backlog #2: **#1 nav memory (tiny) → #2 in-place refresh → this**, because the no-seam
same-builder composition uses #2's refresh hook. **#1 and #2 are DONE + tested (2026-06-26).**

- **3a DONE + tested (2026-06-29):** `Helpers/Gui/MillerPlacement.ahk` — the store (`miller_prefs.json`,
  named-axis schema, AHK-native cached CRUD: `MillerPrefGet/Set/Clear/List`), the LOCAL-scope partition
  (`_MillerApplyPlacement` — top floats above a divider, bottom sinks below, marked), and the
  one-button-as-node control (`MillerPlacementNode` → drill in → Pin top / Sink bottom / Clear, writes +
  refreshes in place via #2). Engine `#Include`s it. Tests: `placement_store_set_get_clear`,
  `placement_apply_writes_pref`, `placement_partition_orders_top_and_bottom`,
  `placement_partition_renders_pinned_to_top`. NOT yet wired into any LIVE menu (that's 3d) and NOT
  auto-applied by the engine core yet (hosts call `_MillerApplyPlacement` explicitly until 3b).
- **3b DONE + tested (2026-06-29):** `Helpers/Gui/MillerSystem.ahk` — `MlSystem(id, label, detail,
  rootBuilder, opts)` (class `MlSystemDesc`: `.root` + `.asNode()`), the self-registering registry
  (`MlSystemGet/List`, `MlRegisterNode`/`MlNodeGet/List`, `MlRegistryDumpJson`), and auto-applied placement
  via a **deep-wrap** (`_MlSystemWrapBuilder` wraps the root builder + every descendant's children builder,
  so placement runs at every depth WITHOUT engine-core surgery — non-system menus untouched). Standalone ==
  mounted (same wrapped builder). Tests: registry get/list, root + nested auto-placement, asNode carries
  system_id, mount-under-dummy-root composability, node-registry round-trip.
- **3c DONE + tested (2026-06-29):** cross-level promotion unified into `_MlSystemPlaceLevel` — a `root`/
  `<page_key>`-targeted item that lives deep is `resolve()`d and surfaced at the system root / matching page
  (deduped: an item already at its target level floats in place, no duplicate). Control extended
  (`MillerPlacementNode(sys, key, chain)`) with this-level / system-root / specific-page (from chain
  ancestors) options. Tests: promote-to-root, promote-dedupe-float, promote-to-page, options enumeration.
  GOTCHA logged: an IIFE's closure and its call-args must stay on ONE line — splitting them (even inside
  call parens) makes AHK read `<Func> <args>` as concatenation ("Expected a String but got a Func") or
  return the builder uncalled ("Missing a required parameter"). Bit both `_MlSystemWrapBuilder` and
  `_MlPlacementOpt`.
- **3d (in progress — touches LIVE menus):** convert the 5 systems to `MlSystem`, wire the placement
  control into each, retiring per-host RegPrefs/bands.
  - **Engine prerequisite DONE + tested (2026-06-29): `pref_key`.** A node may carry `pref_key` to override
    its nav `key` for placement lookup only (`_MlNodePrefKey`, used by `_MillerApplyPlacement` +
    `_MlSystemPlaceLevel`). Needed because the deep-wrap keys placement on one `system_id`, but some
    systems scope items per-sub-level (registry ENTRIES: "foo" in Sites ≠ "foo" in Docs). `pref_key`
    namespaces the prefs (`<regId>/foo`) without rewriting nav keys / deep-links / search. Test
    `placement_pref_key_namespaces_lookup`.
  - **First conversion DONE + verified (2026-06-29): Voice Command Editor ("show fun").** Chosen as the
    lowest-risk proof (nothing to retire, no data migration — placement purely ADDED). Pattern that worked
    (the **recipe for the remaining 4**):
    1. lazy cached getter `_XSystem()` returning `MlSystem("<id>", label, detail, (chain)=>_XRootNodes(chain))`
       (lazy avoids any top-level/include-order dependency on MillerSystem.ahk);
    2. opener: `millerOpts["root"] := _XSystem().root` (the wrapped builder — auto-applies placement);
    3. `_XAsNode()` → `_XSystem().asNode()`;
    4. add `MillerPlacementNode("<id>", <itemKey>, chain)` to each item's options/detail level; thread
       `chain` into that builder for the page-picker;
    5. ONE-TIME (already done): MAINFUNCTIONS.ahk now `#Include`s MillerPlacement.ahk + MillerSystem.ahk
       DIRECTLY (the engine's circular include of them isn't seen by the static closure checker; callers
       with no own #Includes, like VoiceCommandEditorMenu, need MAINFUNCTIONS to carry them);
    6. for per-sub-scope items, set node `pref_key` to namespace.
    Verified live by screenshot (the function options now show a "Placement: normal" row) + suite 83/83.
    Jamie test-drove it — "works perfectly."
  - **Second conversion DONE + verified (2026-06-29): Context Manager ("open context").** system_id
    **"ctxmgr"** (NOT "contexts" — asNode's key = system_id and a registry "contexts" exists; "ctxmgr" was
    already the mount key, so reused it). Removed BOTH `_RegPartitionByPref("contexts")` calls (root +
    nested) — the deep-wrap now floats/sinks contexts at every level; swapped the facet-hub's
    `_RegPrefCycleRow("contexts", token)` for `MillerPlacementNode("ctxmgr", "ctx:"+token, chain)`; threaded
    `chain` into `_CtxFacets`. NO migration (the old "contexts" RegPrefs were dormant — a pre-existing
    key mismatch: cycle wrote bare token, partition read "ctx:"+token). `_ContextEditorAsNode` rebuilt on
    `_CtxSystem().root` (wrapped) keeping its dynamic count + "ctxmgr" key. Verified: `_CtxSelfTest` 58
    contexts deep-valid, screenshot shows "Placement: normal" in the facet hub (row 10), suite 83/83.
    NOTE: at a context's nested level (kids + divider + facets in one list) the deep-wrap floats fav kids
    above everything and a sunk kid would land below the facets — acceptable edge; fav (the common case)
    is clean.
  - **Pluggable-backend hook DONE + tested (2026-06-29).** `MillerPrefBackend(systemId, ops)` in
    MillerPlacement.ahk: a system that already owns a placement-ish store registers an adapter (a Map of
    any subset of `get(itemKey,axis)->Map{dir,target}|"" / set(itemKey,axis,dir,target) / clear(itemKey,axis)
    / list(axis)->Map`); `MillerPrefGet/Set/Clear/List` delegate to it, else fall back to the canonical
    `miller_prefs.json`. Pass `""`/empty to unregister. These are TRANSITIONAL adapters (don't move working
    data now). Test `placement_backend_overrides_store` (set/get/list/clear route to the adapter, JSON
    untouched, unregister falls back).
  - **Configurable control DONE + tested (2026-06-29).** `MillerPlacementNode(sys, key, chain, axis, opts)`
    gained an optional `opts` Map (backward-compatible — VCE/Context call it positionally unchanged):
    `"key"` (node nav key, so a host with TWO axes gives each its own key + layout slot), `"label"` (state
    prefix), `"state"` (precomputed suffix — skip the store read when the host already has the value),
    `"options"` (explicit Array of `Map("dir","target","label")` that REPLACES the default this-level/root/
    page/clear rows — "add an option = push a Map"), `"detail"`. Test `placement_custom_options_replace_default`.
  - **Third conversion DONE + verified (2026-06-29): Projects ("open projects").** system_id **"projects"**.
    KEY DIFFERENCE from VCE/Context: Projects already does its own banding/promotion IN PYTHON (root global
    bands + in-container pinned), so it is **NOT** deep-wrapped with MlSystem auto-placement (that would
    double-sort). Instead it registers a **backend adapter** (`_ProjEnsurePlacementBackend()` in
    ProjectsMenu.ahk) so the unified control reads/writes THROUGH the existing `current`/`favorite` node
    fields; Python keeps doing the sort. Two axes — `"current"` + `"favorite"` — map cleanly onto the schema
    (the agent confirmed both are already two-scope none/local/global): `none`↔cleared, `local`↔`{top,local}`,
    `global`↔`{top,root}`. Adapter pieces: pure translators `_ProjScopeToPref` / `_ProjTargetToScope`;
    ops `_ProjPlacement{Get,Set,Clear,List}` (`get` shells `projects.py node`, `set/clear` shell `projects.py
    set --id X --<axis> <scope>`; `list` returns empty so the engine never promotes — Python owns that);
    explicit option rows `_ProjPlacementOptions(axis)` (○/● current, ☆/★ favorite, none/local/global wording);
    `_ProjPlacementLeaf` swaps the old `act_cur`/`act_fav` cycle leaves (same layout keys preserved) for two
    `MillerPlacementNode` controls. The N.M quick-cycle actions (`_ProjRowActions`) are KEPT (fast path).
    ProjectsMenu.ahk now `#Include`s MillerPlacement.ahk directly (it's the caller); MillerPlacement added to
    `AlwaysOn/_CuratedClosure.ahk` (always-on entry points include ProjectsMenu → need it in their closure;
    the engine's circular include isn't seen by the static checker — same reason GuiLayout is listed there).
    Verified: suite 85/85, live screenshot shows "Current: off" / "Favorite: off" rows in the right slots
    with "off > local (this list) > global (root)" detail, and a reversible CLI round-trip confirmed
    `set --id X --current local/global/none` mutates + resets cleanly. **Jamie test-drove it 2026-06-29 —
    "works perfectly."** DONE.
  - **Fourth conversion — Spotify, REFRAMED as a multi-tag system (Jamie 2026-06-29).** On reviewing the
    choices Jamie said her real want is "to_listen and OTHER kinds of tags... very easy to implement into the
    data" — i.e. independent, extensible tags on artists/albums/playlists, not the single mutually-exclusive
    `personal_status` enum it is today. Decision (`0.00` = all recommendations): evolve `personal_status`
    (one of ""/favorite/to_listen) → `personal_tags` (a LIST), tags stored in music.json, an EDITABLE tag
    vocab, designed generic for the whole media catalog. Built so far:
    - **Python foundation DONE + verified (2026-06-29)** in `Scripts/MediaCatalog/media_ingest.py`: editable
      vocab (`personal_tags.json` next to the catalog; `personal-tag-vocab` / `personal-tag-def` upsert+delete
      — adding a tag = one CLI call; default favorite ★ / to_listen ◇); lazy+additive helpers `_item_tags`
      (derives from personal_status if `personal_tags` absent — no destructive migration) / `_set_item_tags`
      (writes the list + mirrors a legacy `personal_status` so order-sensitive AHK regexes still parse);
      `_tags_emit_fields` appended to every row emit (`list-items`, `list-albums-for`, `_album_row`) — keeps
      `personal_status` IN PLACE (regex-safe) and ADDS `personal_tags` (comma) + `personal_tags_badges`
      (pre-rendered symbols, Python owns vocab→symbol); commands `personal-tag <id> --toggle/--add/--remove`,
      `item-tags <id>`, `migrate-personal-tags` (optional bulk-fill); `list-items`/`pick-random`
      `--personal-status` now means TAG MEMBERSHIP (back-compat for `spot favorite`/`spot listen`). Verified
      via CLI: favorite+to_listen held SIMULTANEOUSLY on one artist (was impossible), badge "★ ◇", filter,
      vocab add/delete — all reversible, test data (music:2pac) reset clean.
    - **Generic control DONE + tested (2026-06-29):** `Helpers/Gui/MillerTags.ahk` — parallels MillerPlacement
      (pluggable `MillerTagsBackend(systemId, ops{vocab,get,toggle})` + drill-in `MillerTagsNode` where each
      vocab tag is an INDEPENDENT toggle row, ✓ when present + `MillerActiveRefresh` on flip + `MillerTagsBadges`
      for row display). Distinct from MillerPlacement (positional) — tags are membership. Added to the test
      runner. Test `tags_control_toggles_independently` (favorite+to_listen coexist, ✓ marking, badges). Suite
      86/86.
    - **Spotify WIRING DONE + screenshot-verified (2026-06-29).** Tags backend adapter for system "spotify"
      (`_LibEnsureTagsBackend` in SpotifyLibraryManager: vocab/get/toggle via `media_ingest.py
      personal-tag-vocab / item-tags / personal-tag --toggle`, all SYNCHRONOUS so a follow-up
      MillerActiveRefresh reads post-write). The unified `MillerTagsNode` "Tags ▸" control is prepended to
      the artist drill (`_LibAlbumNodes` returns `[tagsNode, albumSection]`). `_LibStatusAction` now TOGGLES
      a tag (N.2/N.3 quick-toggles are independent); `_SetItemPersonalStatus` (the modal/album path) adds/
      removes via `personal-tag`; `_LibStatusBrowse`'s clear uses an explicit `_LibTagSet(...,false)`. All
      four artist/album/item badge sites read `personal_tags_badges` (catalog-list path + the 4 regex
      builders — each got a `personal_tags_badges` capture appended after their last field, which the emit
      guarantees is present). SpotifyLibraryManager `#Include`s MillerTags directly; MillerTags added to
      MAINFUNCTIONS (static-closure, same as MillerPlacement) — GuiHost inherits via MAINFUNCTIONS.
      Verified: live screenshot shows "2Pac ★ ◇" (both tags independent) in the artist list + "Tags ▸" as
      row 1 of the album preview; suite 86/86; 2Pac reset to untagged (data restored).
      GOTCHAS LOGGED: (1) `MillerTagsNode` must NOT fetch tag state for its label by default — a Miller
      previews the highlighted row's children eagerly, so a per-preview backend call (Python subprocess)
      hangs/slows the build; the label is static "Tags ▸" unless `opts.show_state` + a cheap backend. (2)
      `load_personal_tag_vocab` is called once per row emit (179×/list) — CACHE it per-process (it reads E:).
      (3) bash/MSYS pipes mangle UTF-8 symbols (mojibake in CLI inspection) but the AHK path
      (`_RunIngestCapture` → `FileRead UTF-8`, `_emit` reconfigures stdout to UTF-8) is correct — don't chase
      the bash mojibake. (4) standalone AHK probes via `ahk.py run` get a different PATH/env (py not found) —
      don't trust them; instrument the real host or screenshot.
      DEFERRED (lower value, future): per-album drill-in Tags control (albums keep Enter=play + N.2/N.3
      quick toggles); generalize browse-by-tag beyond favorite/to_listen; the `_AskPersonalStatusGui` modal
      could become a multi-select checklist. Jamie's interactive test-drive still owed (drill "Tags ▸",
      toggle a tag, confirm badge updates).
  - **Fifth conversion DONE + screenshot-verified (2026-06-29): Registry Editor ("open registry").** The
    earlier FINDING feared a hard data migration — but Registry turned out to be the **Projects pattern
    exactly**: it already does its OWN sort (`_RegPartitionByPref` for the root registry list,
    `_RegOrganizeEntries` for entries — the latter KEEPS group clustering) reading the in-memory `RegPrefs`,
    so a **backend adapter** sidesteps migration entirely and `_RegOrganizeEntries` is UNTOUCHED. system_id
    **"registry"**, ONE system with the two scopes namespaced into the itemKey: `"<scope>/<key>"` where scope
    = `"_registries"` (root list, key=regId) or a regId (entries, key=entry key) — regIds carry no "/", so
    split on the FIRST. Mapping: `fav`<->`{top,local}`, `hidden`<->`{bottom,local}`, normal=cleared. Pieces:
    `_RegPrefStoreSet` (the new single write path — `registries.py set-pref`+`cache` AND updates in-memory
    `RegPrefs`, fixing a latent staleness bug where the old `_RegPrefSet` left `RegPrefs` stale so an
    in-place refresh showed nothing until reopen); adapter `_RegPlacement{Get,Set,Clear}` + `_RegSplitItemKey`
    + `_RegEnsurePlacementBackend` (registered in `_RegLoadData`, NO "list" op so the engine never promotes);
    `_RegPlacementControl(scope,key,nodeKey,label)` + `_RegPlacementOptions` (the unified `MillerPlacementNode`
    with Registry's Favorite/Hide/Normal vocabulary + precomputed state via `_RegPrefStateLabel`). Swapped the
    two `_RegFavHideNodes` sites (`__regfavhide` registry + `__favhide` entry). The N.2/N.3/N.4 quick row-
    actions (`_RegPrefRowActions`/`_RegPrefSet`) are KEPT (now route through `_RegPrefStoreSet`, so they
    update `RegPrefs` too). `#Include`s MillerPlacement directly + added to MAINFUNCTIONS. Verified: root
    screenshot shows ★ Google Docs/Sites/Youtube floated to top + ★ Journal/Lyrical entries floated; the
    Journal entry's actions show "Placement: ★ Favorite" (row 6); `set-pref`+`cache` round-trips reversibly;
    validate + closure clean. DEAD CODE removed (2026-06-29): `_RegFavHideNodes`, `_RegFavHideRow`,
    `_RegPrefCycleRow`, `_RegPrefCycle`, and `_RegPrefNext` deleted (all unused after the placement-control
    unification); `_RegPrefStateLabel` KEPT (still used by `_RegPlacementControl`'s state line). Stale comment
    refs updated in CommandActionNodes.ahk + ContextEditorMenu.ahk. GUI suite 86/86. (The `.ahk.meta.json`
    function-index sidecars still list the removed names until the Stop hook regenerates them — auto-corrects.)

  **ALL 5 CONVERSIONS DONE (2026-06-29).** Voice Command Editor, Context Manager, Projects, Spotify (as the
  multi-tag system), Registry Editor — every Miller's favorite/placement now flows through the unified engine
  control (`MillerPlacementNode` / `MillerTagsNode`) + pluggable backend. The unification arc (TASK 3) is
  complete. Transitional adapters (Projects bands, Spotify personal_tags→catalog, Registry RegPrefs) can
  migrate onto the canonical `miller_prefs.json` later, but the model is proven.

  **PERFORMANCE — root cause found + fixed (2026-06-29).** "Show fun" took ~10s to open. We MEASURED instead of
  guessing, and the suspected culprits were wrong: the Python regen is only 268ms (cold-start floor 42ms); the
  ~1000-node build is cheap. The real cost was the **pure-AHK JSON parser** (`JsonFunctions.ahk` `JsonParse`)
  chewing char-by-char through big files: `voice_by_function.json` (306 KB) = **8,953ms**, `fn_to_file_index.json`
  (142 KB) = 1,484ms → 10.4s, paid on EVERY open (each "show fun" is a fresh AHK process; no cache survives).
  FIX = rewrote the parser hot path (kept the same name/API/output, so all ~13 callers + the serializer are
  untouched): (1) index a pre-split char array instead of `SubStr(bigstring,i,1)` per char; (2) the string
  reader bulk-copies runs via `InStr` instead of appending char-by-char. Result: 10,437ms → **438ms (24×)**,
  deep-compare byte-identical on the real files, 28-case edge test added (`Helpers/Tests/test_json_functions.ahk`,
  in `_run_unit_tests.ahk`). This is MACHINE-WIDE: every big-JSON reader benefits — Registry (`registries_dump.json`
  240 KB), Projects (`projects.json` 110 KB + `history.json` 101 KB), Context, Spotify. Full GUI suite 86/86,
  unit suite 45/45 (incl. the new JSON tests). Also added `_VceIndexStale()` (mtime skip-if-fresh): the 268ms
  Python regen now only runs when the grammar dump or command store changed, so a repeat open skips it entirely
  (the add-then-reopen flow still regenerates because `_VceAdd` writes the store + reloads grammar).

  **Debounce (#6) — evaluated, NOT changed.** The engine's `SetTimer(_doRefreshRightOnly, -50)` on ItemFocus is a
  one-shot that RESTARTS on each keystroke, so fast arrowing already coalesces — the preview builds only for the
  row you land on. Bumping 50→120ms would add latency with no benefit. Left as-is.

  **SPOTIFY LIBRARY drill — fixed (2026-06-29).** Every drill (artist → albums) took ~1–2s. MEASURED: each level
  shells a fresh `py media_ingest.py` that called `mc.load_media_type("music")` — parsing the **54 MB / 9,931-item**
  `music.json` (~900ms) — and a data-section drill fires TWO commands (`catalog-list-spec` + `catalog-list`), with
  the eager preview rebuilding it on every arrow. THREE fixes, all shipped:
    1. **SQLite read-cache** (`Scripts/MediaCatalog/catalog_db.py`, NEW). Derived index next to `music.json`,
       rebuilt only when the JSON is newer (mtime) or the schema version bumps; reads (`get` / `by_subtype` /
       `children`) are indexed lookups. NO primary key on id (the catalog has duplicate ids — a PK collapsed 41
       artists/42 albums; verified by deep-compare). Every query FALLS BACK to a full parse on any error, so it
       can't regress. Routed `cmd_get`, `cmd_list_albums_for`, `_albums_loader`, `_artists_loader`,
       `cmd_list_items(--subtype)` through it. Equivalence: artists 3171=3171, albums 6755=6755, 30 artists'
       children + 15 `get`s all byte-identical.
    2. **Latent-bug fix:** `_personal_tag_vocab_path()` called `load_media_type` (full 54 MB parse, ~381ms) JUST to
       get the catalog's *directory*, on EVERY tag-emitting row command. Swapped to `mc.media_catalog_path()` (no
       read). cProfile found it hiding behind the tag-badge emit.
    3. **AHK side:** session read-cache in `_RunIngestCapture` (SpotifyAddFunctions.ahk) — caches the heavy READ
       commands (catalog-list / spec / get / list-albums-for / list-items) keyed by full args, flushes on ANY
       non-whitelisted command so a mutation never serves stale rows; and a per-Miller `preview_debounce` opt
       (MillerColumnPickGui.ahk, default 50) set to 240ms for the Spotify hub so a brisk scroll doesn't fire a
       build per row.
  RESULT: drill commands 942/951/539ms → **~140ms** (the bare py-spawn+import floor); album drill ~1.1s → ~280ms;
  re-visits hit the AHK cache (no spawn). TRADEOFF: a catalog write (tag toggle rewrites `music.json`) makes the
  next read rebuild the DB (~2.3s, build pragmas on). Pure browsing never rebuilds. Duplicate ids block a safe
  incremental update, so it's a full rebuild. OPTIONAL follow-up: `pip install orjson` (catalog_db could use it
  with a stdlib fallback) would cut the rebuild ~4×; or the persistent daemon to erase the ~140ms spawn floor —
  neither needed unless the rebuild hitch or the floor starts to bite.

  **POST-ACTION REFRESH — "every action takes 3-4s" (2026-06-29).** Jamie: any action that changes data makes
  the Miller sit for 3-4s, across all of them. MEASURED each menu's load/regen+mutation Python cost (not guessed):
  Context `contexts --json` 77ms, Projects `list` 82ms, Spotify drill ~140ms (post-SQLite), registry `set-pref`
  76ms — all fine. The outlier: **`registries.py dump`/`cache` = ~2,500ms**, and EVERY registry write calls
  `_RegPy("cache")` to regenerate `registries_dump.json`. So a registry placement toggle = set-pref (76ms) +
  cache regen (2.5s) ≈ the felt 3s. cProfile: `_load_json` was called **28,199×** (`read_text` alone 2.07s) —
  `discover_registries` re-reads the same source files (the function index + per-registry sources) thousands of
  times with no memoization. FIX: a per-process TEXT cache in `_load_json` (cache the file bytes, parse fresh each
  call so no shared-mutable-object risk; invalidated in `_atomic_write` so a write-then-read is never stale; safe
  because each registries.py CLI call is a fresh process). Result: dump/cache **2,579ms → 568ms (4.5×)**; a
  registry action ~2.6s → ~0.6s. Output verified deterministic + unchanged. Combined with the Spotify SQLite work,
  the two big post-action offenders (Registry 2.5s, Spotify mutation-rebuild 2.3s) are both addressed; Context /
  Projects / VCE post-action paths already measured sub-300ms. (If a menu still feels slow post-action, profile
  THAT command — don't assume; the wins here all came from measuring, and each "obvious" culprit was wrong.)

  **CLOSE+REOPEN "FLASH" on edits — fixing (2026-06-29).** Jamie: after an edit the Miller window quits out for a
  beat and pops back — wants it to reload IN PLACE. Root cause: the in-place refresh mechanism (`MillerActiveRefresh`
  / `MillerRefreshOrReopen`, TASK 2) exists and Spotify's tag-toggle uses it flash-free, but the other menus' edit
  actions still did `close.Call(); OpenX(...)` (full close + reopen = the flash). The in-place path re-renders only
  the CURRENT level; its builders re-read fresh data (Context via `GetContext`→`LoadContexts`, cache-invalidated by
  `WriteContextProfile`; Spotify via the SQLite read), so values update WITHOUT a reopen, cursor kept. CONTEXT done:
  `_CtxReopen` now tries `MillerActiveRefresh()` first and only closes+reopens as a fallback; the 10 edit/toggle sites
  drop their premature `close.Call()` and pass `close` for the fallback (add→drill-into-new and remove→go-to-root
  keep the reopen since they deliberately change location). Verified: validate clean, GUI suite 86/86.

  **Then made it the ENGINE DEFAULT (Jamie: "make this the default that Millers are automatically usable on
  creation").** The engine already had an OPT-IN "leaf that mutates + stays open re-renders in place"
  (`refresh_leaf_on_fire`) — flipped to DEFAULT-ON (opt out with `refresh_leaf_on_fire:=false`). Now ANY Miller,
  existing or newly scaffolded, auto-refreshes the current level in place after a leaf action that doesn't `close()`
  — zero per-menu wiring. Added `opts["reload"]` (called before re-render) for menus whose builders read in-memory
  globals, and a shared `_refreshInPlaceNow()` that both the post-action auto-refresh and `MillerActiveRefresh`
  route through; it shows a brief "⟳ refreshing…" breadcrumb (the "reload animation if it hangs" idea — static text,
  since a synchronous reload freezes the UI thread; a true spinner would need async). Documented as the default in
  `miller-authoring.md`. GUI suite 86/86 across runs. orjson installed + wired into catalog_db (stdlib fallback) but
  only ~200ms — the Spotify rebuild is I/O-bound (54MB write), not serialize-bound.

  **BUG found on test-drive + fixed: Remove/Rescope "flashed refreshing then did nothing" (2026-06-29).** In the
  Voice editor, removing/rescoping a command deleted it but the UI didn't update (had to reopen). Two causes: (1)
  the in-place refresh re-rendered the CURRENT level — the deleted command's OWN action submenu (Remove/Rephrase/
  Rescope) — whose effect actually shows one level UP (the command list); (2) the Voice builders read the in-memory
  `VceIndex`, never reloaded, so even the right level would redraw stale. FIX: new engine primitive
  `MillerDrillOutRefresh()` (process-global like `MillerActiveRefresh`) — pops ONE level then reload+re-render, so
  the change lands on the parent list in place, no flash. `_CmdRemove`/`_CmdRescope` (shared cluster, used by Voice
  AND Context — command-actions is exactly one level below the command list in both) now call it (fallback: close +
  the host reopen). Voice sets `opts["reload"] = _VceLoadData` (mtime-gated) so the parent re-render is fresh. A
  `_refreshHandled` flag stops the engine's post-leaf auto-refresh from double-firing when an action self-refreshes.
  GUI suite 86/86. Lesson: in-place "re-render current level" is right for stay-here edits but WRONG for actions
  whose effect shows on a parent (delete/rescope) — those need drill-out-refresh.

  REMAINING (existing menus): most of Registry/VCE/Projects' explicit close+reopens are NAVIGATIONAL (land on a moved
  entry, enter a new registry, the add-wizard needs the screen) — CORRECT as reopens, and the engine default leaves
  them alone (they call close). Only the gratuitous "edit-then-reopen-same-place" ones are worth converting (Context
  done; Registry rename + a few entry-field edits are candidates): drop their close+reopen so the engine default
  takes over, plus `opts["reload"]` for Registry/VCE (in-memory globals). Best done with Jamie spot-checking each,
  since in-place re-renders the CURRENT level and only a human confirms the landing matches per action.

JSON layer last, only if wanted. Authoring conventions documented in the ahk-functions skill
(`references/miller-authoring.md`).

---

Keep this list short. If a fourth Miller appears, it earns a row here + a domain doc; it does not earn
a bespoke UI stack.
