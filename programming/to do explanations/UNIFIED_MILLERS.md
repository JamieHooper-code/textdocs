---
tags: [design, principle, miller, architecture]
related: ["[[CONTEXT_MANAGER]]", "[[VOICE_COMMAND_SYSTEM]]"]
status: principle
updated: 2026-06-26
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
  ONE fav/hide **cycle** (`_RegPrefCycleRow`) + prefs (`set_pref` scope per domain). Build a new editor
  by composing these, not reinventing.
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
- `_RegPrefCycleRow(scope, key, reopenFn)` — the fav/hide cycle; used by both the Registry Editor and
  the Context Manager, each passing its own reopen.
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

One pref per item, semantic-keyed (survives across every place the item appears):

```
prefs[system_id][item_key] = { dir: "top" | "bottom" | "",   target: "local" | "root" | "<level-path>" }
```

- **dir** — `top` pins, `bottom` sinks. **There is no true hide** — "hide" just pushes to the very
  bottom; nothing ever disappears (Jamie 2026-06-26). (The engine's separate Ctrl+R layout true-hide is
  a different, opt-in thing and stays as-is.)
- **target** — *which level the float applies at*, and it's **contextually editable** (Jamie: global is
  not always the system boundary):
  - `local` = top/bottom of the item's **own immediate level**, wherever that level renders.
  - `root` = promote to the top/bottom of the item's **system root level** (the default "global"). Bounded
    by the system: a `root`-pinned project floats to the top of the **Projects** subtree, NOT above the
    mount point into a host master. Global never escapes its `system_id`.
  - `<level-path>` = promote to a **specific page the user chooses at edit time** (a chain key-path within
    the system). This is the "sometimes global means *this* page" case.

**Keying is semantic** (`system_id` + stable `item_key`), not by `layout_id` — so a fav set in the
standalone Miller is the same fav when the item is mounted elsewhere. Backends are **pluggable**: default
is one engine JSON store (`miller_prefs.json`); Projects (Python bands) and Spotify (`personal_status`)
register their own get/set so they render through the same UI without ripping out working code.

### The control — one button that is a node

A single placement row per item (label shows current state, e.g. `Placement: ★ top · global`); **drill in**
and the choices are pickable rows — top/bottom × {this level, system root, a specific page…} + clear.
"One button" satisfied; "every action is a visible, pickable row" satisfied; **adding an option later = add
a row** (extensible by construction). Optional focused-row hotkey cycles the most-common axis in place.

### Engine mechanics (free for every Miller)

At each level render the engine: walks the chain to the nearest ancestor carrying a `system_id` (that's the
current system + the pref scope); partitions the level's rows by `prefs[system_id][rowKey]` — `local` dir
sorts within this level, `dir`+`target` matching this level floats/sinks here; and at the level that came
straight from a system's `rootBuilder` (tagged "system root"), prepends/appends the system's `root`-targeted
favs — **materialized by reusing the system's existing `search_index`** as the item→node resolver (no new
per-system code; systems without a `search_index` get `local` only until they add one).

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
to it incrementally (each conversion is local and behind the same UX).

### Build order

Rides on backlog #2: **#1 nav memory (tiny) → #2 in-place refresh → this**, because the no-seam
same-builder composition uses #2's refresh hook. Pref backend + the placement node first (visible win on
existing Millers), then `MlSystem` + incremental conversion.

---

Keep this list short. If a fourth Miller appears, it earns a row here + a domain doc; it does not earn
a bespoke UI stack.
