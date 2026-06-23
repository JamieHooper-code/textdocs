---
tags: [miller, gui, design, layouts, projects, autohotkey, testing]
created: 2026-06-16
status: design-locked
owner: Jamie
---

# Miller Layout System — catalog + per-context layouts (reorder / hide / sections)

Design spec for a **generic, machine-wide layout system** for Miller column menus:
reorder action rows, hide rows, and split them above/below a divider — **per context**,
over a **shared catalog**, persisted in a queryable registry, editable by the UI *and*
programmatically (by Claude / automation). Build this AFTER compaction.

Related code:
- Engine: `AutoHotkey/Helpers/Gui/MillerColumnPickGui.ahk`
- Node constructors + headless snapshot/lint: `AutoHotkey/Helpers/Gui/MillerNodes.ahk`
- Projects menu (first adopter): `AutoHotkey/Helpers/ProjectsMenu.ahk`
- Projects engine (pattern to copy): `AutoHotkey/Scripts/projects/projects.py`
- GUI conventions: `AutoHotkey/docs/gui-conventions.md`
- Skill ref to update when done: `~/.claude/skills/ahk-functions/references/` (miller-authoring.md)

---

## STATUS — BUILT (2026-06-18)

All 8 phases are implemented + verified (54/54 GUI suite green incl. 15 layout/section
tests; layouts.py selftest green). What shipped:
- `Scripts/gui_layouts/layouts.py` + `INIDATA/Layouts/gui_layouts.json` registry + CLI + selftest.
- `Helpers/Gui/GuiLayout.ahk` thin store API (cached get + mutators).
- Engine: pure `_MlApplyLayout` + `_MlExpandSections` (in MillerNodes.ahk); `layout_id`
  wired into level build; rearrange routes layout rows → `GuiLayout*`, data rows → `on_move`;
  `Del`=hide, `NumpadSub`=divider, hidden rows ✕-marked; `Ctrl+S` cycles a data section's sort.
- ProjectsMenu adopted (per-type `proj.functions.{workspace,project,todo}`).
- Layouts browser `OpenLayouts()` (`Scripts/LayoutsViewer.ahk`) + voice "open layouts".
- Data-section mechanism (`MlDataSection` + expand/collapse) — **field + direction** model:
  sort FIELDS are the vocab (each with a default dir); DIRECTION (asc/desc) is an orthogonal
  toggle, so one `year` field gives newest AND oldest (no doubled modes — Jamie 2026-06-18).
  **Ctrl+S** cycles the field, **Ctrl+E** flips direction; persisted as `"field:dir"`
  in `sorts[sectionId]`. Pure transform `_MlExpandSections` (17 layout/section tests).
- **Generic sortable-list framework** (Jamie: "build a generic sortable-by-category, don't
  reinvent per list"): `Scripts/sortlib.py` = declarative sort engine (a list declares its
  fields + extractors once; `sort_items` pairs field+direction — no per-list comparator).
  `media_ingest.py` `CATALOG_LISTS` declares the catalog's lists + the generic
  `catalog-list-spec` / `catalog-list --ctx --sort --dir` commands. AHK
  `MlDataSectionFromCatalog(list_id, ctx, rowToNode)` builds a sortable section from a
  `list_id` with ZERO per-list code. Declared: `artist.albums` (plays/name/year),
  `library.artists` (name/albums). Proven end-to-end against real catalog data.
- Decisions resolved during build: sort-field cycle = **Ctrl+S**, direction toggle =
  **Ctrl+E**, divider = **NumpadSub**, hide = **Del** (all both modes where relevant).
  The drill-in "sort sub-menu" tunnel was deferred in favor of the Ctrl+S/Ctrl+E
  cycle (simpler, covers the same need); revisit if wanted.

- **Spotify album browse WIRED (2026-06-18):** the live node-mode library
  (`OpenSpotifyLibrary` → `_LibRootNodes` → Browse → genre → artist → `_LibAlbumNodes`) now
  renders albums as a `MlDataSectionFromCatalog("artist.albums", …)` section. All artist
  branches share `layout_id "spotify.album.level"`, so the album sort (`sorts["artist.albums"]`)
  is library-wide. Ctrl+S cycles most-played / name / year; Ctrl+E flips direction.
  Falls back to the legacy inline list if the catalog factory is unavailable. Engine gained
  two general fixes for levels-mode data sections (per-level `layout_id`; preview-pane
  expansion). Verified: 16 real albums, default plays-desc → "All Eyez On Me"; 56/56 suite.

Remaining/optional: **genres sortable by total plays** (Jamie 2026-06-18) — feasible via the
same pattern: a Python catalog aggregation summing album plays per genre (one pass: artist→
play-total, then genre→sum over its artists), declared as a `library.genres` list (fields
name / plays / items) + wiring the Browse genre level (`_LibBrowseGenreNodes` under the
`browse` node, with a `layout_id`) as a section. Then more lists (one entry each); optionally
the drill-in sort sub-menu; per-PAGE layout ids. The rest of this doc is the original design.

---

## 0. What already exists (built this session, 2026-06-16)

The **rearrange-mode ENGINE is already built and verified** in MillerColumnPickGui.ahk:
- `Ctrl+R` toggles rearrange mode (opt-in: only live when the menu supplies `opts["on_move"]`).
- In mode: `Enter` grabs the focused **reorderable** row (`node["reorderable"] = true`) or drops it;
  while holding, `Up/Down` move it among reorderable siblings; `Esc` leaves the mode.
- Breadcrumb shows the live state ("* REARRANGE (HOLDING — Up/Down to move) · Esc=done").
- `_rearrangePreserveSel()` keeps the focused row selected across mode-toggle re-renders.
- Callback: `opts["on_move"](chain, key, dir, neighborKey)` persists each adjacent swap.
- 39/39 GUI suite still passes (mode is inert without `on_move`).

**Data rows** already reorder through this: `_ProjRowToNode` sets `reorderable:true`, and
`_ProjOnMove` calls `projects.py reorder <key> --dir up|down` (engine `order` field). Verified.

`_ProjOnMove` currently **no-ops for `act_*` keys** — action-row (catalog) reordering is THIS
spec's job. The layout system plugs into the SAME rearrange mode; it adds the catalog/layout
store, the apply transform, hiding, sections, and a browser.

---

## 1. Core concepts — Catalog vs Layout (the key distinction)

- **Catalog** — the full set of rows *available* in a context-class, defined ONCE in code.
  Each catalog row = `{ key, label, detail, builder/action }`. Example: the "project functions"
  catalog = Open, View/edit, Edit title, Current, Favorite, Mark done, Finish, Convert, Add…,
  Set spoken, Re-link, Delete, etc. The catalog is the universe; it never changes per page.

- **Layout** — a per-context *arrangement* that references catalog keys. Stores:
  - `order` — ordered list of catalog keys (rows not listed fall to the end in catalog order).
  - `hidden` — set of keys not shown (still visible, dimmed, in rearrange mode so you can unhide).
  - `sections` / divider — a split point (or named sections) so rarely-used rows sit **below a
    divider**. Minimum: one `divider_after` key (rows after it render beneath a divider line).
    Extensible to N named sections later.

**Why this resolves shared-vs-per-type:** the catalog is shared (define project-functions once),
but each context (workspace / project / todo — or even an individual page) gets its OWN layout.
So todo can float "Mark done" to the top and bury "Convert" below the divider, while project does
the opposite — same catalog, different layouts. (Jamie 2026-06-16.)

**Context granularity:** layouts are keyed by a stable `layout_id`. Start at per-TYPE
(`proj.functions.workspace` / `.project` / `.todo`). The id scheme allows finer (per-page) later
without redesign — a page just uses a more specific id, falling back to the type default if unset.

---

## 1.5 Data sections + sort modes (generated rows that collapse in edit mode)

A THIRD row kind, beyond catalog (action) rows: a **data section** — a slot whose rows are
GENERATED (every album for an artist, every todo, a media list), not hand-authored. You don't
hand-order generated rows; you pick a **sort mode**. (Jamie 2026-06-16, from the Spotify library.)

A data-section slot declares (in code):
- `section_id` (slot key, e.g. `"albums"`), `label` (`"Albums"`),
- `sort_modes`: the orderings it supports, each `{id, label}` — e.g. `plays_desc` ("Most played"),
  `alpha` ("A–Z"), `year_desc` ("Newest"), `year_asc` ("Oldest"),
- `rows(sortMode) -> [generated nodes]` — the provider.

Behavior:
- **Normal mode:** the section EXPANDS inline — its generated rows render in place (ordered by the
  section's `current_sort`), positioned among the other slots per the layout's slot order.
- **Layout (edit) mode:** the section COLLAPSES to a single row showing `label` + current sort
  (e.g. `Albums  ·  Most played`). So a page like
  `[Play random album · Play random unheard · album1 · album2 · …albumN]` collapses to
  `[Play random album · Play random unheard · Albums▸]` — three reorderable SLOTS. You can
  reorder / hide / divider-place the Albums slot like any slot, AND **tunnel into it** (drill) to a
  tiny sub-menu of its `sort_modes`; picking one sets `current_sort` (persisted per context). So you
  change "most played → alphabetical → chronological" for the whole block instead of dragging rows.
- **Runtime sort cycle (the "blank hotkey"):** a dedicated key cycles the page's (focused/primary)
  data section's `current_sort` through its `sort_modes` and re-renders — one press = next sort.
  Available in NORMAL mode (not just edit), so re-sorting a library is a single button. Key TBD
  (non-letter so the filter Edit doesn't eat it; candidates `Ctrl+S` / a numpad key / a Stream Deck
  button) — pairs with a Stream Deck/numpad binding for "tap to cycle the library's sort."

Persistence: the layout registry entry gains `sorts: { "<section_id>": "<mode_id>" }`. The
`sort_modes` + provider stay in code; only the CHOSEN mode is stored (per `layout_id`), so it's
editable by the CLI/automation too (`layouts.py set-sort <id> --section S --mode M`).

Generalizes beyond Spotify — any generated list (todos, media catalog, search results) becomes a
sortable, collapsible section. Larger task: build as a PHASE AFTER the core catalog/layout
reorder+hide system is solid (see §8).

## 2. Storage — `layouts.py` + machine-wide registry (DECISION: yes)

Mirror the `projects.py` / `clog.py` pattern: **Python owns all JSON; AHK shells it.**

- Engine: `AutoHotkey/Scripts/gui_layouts/layouts.py` (BOM-less UTF-8 both ways; `@@FILE:` arg
  passing for any free text, same as projects.py).
- Store: `AutoHotkey/INIDATA/Layouts/gui_layouts.json`. Atomic writes (tempfile + `os.replace`).

Registry schema (self-describing so a browser/automation can edit a layout WITHOUT its owning
menu running):

```json
{
  "version": 1,
  "layouts": {
    "proj.functions.todo": {
      "label": "Project functions — todo",
      "owner": "ProjectsMenu / _ProjActionLeaves",
      "order":  ["act_open", "act_view", "act_cur", "act_done", "act_finish", "..."],
      "hidden": ["act_convert"],
      "divider_after": "act_finish",
      "sorts":  { "albums": "plays_desc" },
      "items":  { "act_open": "Open", "act_view": "View / edit text", "...": "..." },
      "updated": "2026-06-16T..."
    }
  }
}
```

- `items` carries key→label so the registry is human-readable + browsable standalone.
- A menu **registers/refreshes** its catalog on use: `layouts.py ensure <id> --label .. --items @@FILE`
  (adds any new catalog keys to `order`'s tail, prunes keys no longer in the catalog, never
  reorders existing). So code can add a new action and it appears at the end without clobbering
  a user's arrangement.

CLI surface (also the programmatic-edit API — DECISION: editable by Claude/automation, not just UI):
```
layouts.py list                          # every layout: id, label, owner, #rows, #hidden
layouts.py get <id>                      # full entry (JSON)
layouts.py ensure <id> --label L --items @@FILE   # register/refresh catalog (code-side)
layouts.py move <id> --key K --dir up|down        # swap K with neighbor in order
layouts.py set-order <id> --keys @@FILE           # replace order wholesale
layouts.py hide <id> --key K   /   show <id> --key K
layouts.py set-divider <id> --after K | --clear
layouts.py set-sort <id> --section S --mode M     # data-section sort mode (see §1.5)
layouts.py apply <id> --rows @@FILE       # (optional) headless: return ordered/visible rows
```

This gives Jamie the thing she wants: **one queryable list of every layout in the system**, all
in one place, linkable (menus reference by id; two menus can share an id), version-stamped.

---

## 3. Engine integration (DECISION: bake apply into engine; persistence via thin store API)

The Miller already has opt-in hooks (`row_actions`, `search_index`, `watch_file`, `nav_file`,
`on_move`). Layout support is one more, consistent with that pattern:

- **Apply (in the engine):** a node/level may carry `layout_id`. When building that level's rows,
  the engine runs a **pure transform** `_MlApplyLayout(nodes, layoutEntry) -> nodes`:
  - reorder `nodes` by `layoutEntry.order` (unknown keys keep catalog order at the tail),
  - drop `hidden` keys (unless rearrange mode is active → show them dimmed),
  - insert a divider node after `divider_after`.
  This is a PURE function (no I/O) → unit-testable headlessly (see §6).
- **Persistence (behind a thin store API):** the engine never reads/writes the JSON directly.
  It calls a small AHK shim `GuiLayout.ahk`:
  - `GuiLayoutGet(id) -> layoutEntry` (cached per render; shells `layouts.py get`),
  - `GuiLayoutMove(id, key, dir)`, `GuiLayoutHide(id, key)`, `GuiLayoutShow(id, key)`,
    `GuiLayoutSetDivider(id, key)` (shell the matching CLI).
  So the engine stays format-agnostic; the store schema can evolve without engine edits.
  `GuiLayout.ahk` is the ONE place that knows the file/CLI — reusable by every Miller + the browser.

Adoption cost for any menu: set `layout_id` on the level + call `GuiLayoutEnsure(id, label, items)`
once when building the catalog. That's it — reorder/hide/sections/persistence come free.

How it composes with the existing `on_move`: in rearrange mode, when the held row belongs to a
layout (`act_*` / catalog row), the engine routes the move to `GuiLayoutMove(layout_id, key, dir)`
instead of `opts["on_move"]`. Data rows (no layout) keep going through `on_move` → `reorder`.
(Engine decides by: does this row's level have a `layout_id` AND is the key in that layout?)

---

## 4. Reorder / hide UI — Mode A (DECISION: A, mode + grab)

Keep the built mode-and-grab interaction; add hide + divider controls:

| Key | In rearrange mode |
|-----|-------------------|
| `Ctrl+R` | toggle rearrange mode on/off |
| `Up/Down` | navigate (not holding) · move the held row (holding) |
| `Enter` | grab the focused row / drop the held row |
| `Del` | toggle **hide** on the focused row (non-letter, so the filter Edit doesn't eat it) |
| `Ctrl+Enter` *(or `-`)* | set the **divider** after the focused row (TBD key; non-letter) |
| `Esc` | leave rearrange mode (does NOT close the menu) |

Hidden rows render **dimmed** while in rearrange mode (so they're unhideable); normal mode omits
them. Breadcrumb already shows mode/holding state; extend hint text to mention `Del`=hide.

(Reason for A over Ctrl-arrow chords: explicit, discoverable, no accidental moves, avoids
modifier soup — fits Jamie's numpad-first world. Reordering is an occasional dev/tailor action.)

---

## 5. Layouts browser (DECISION: build it)

A Miller (`OpenLayouts` / `_LayoutsBrowserShow`) that lists every registry entry and edits any
layout WITHOUT its owning menu running (possible because the registry stores `items` labels):
- Root: one row per layout (`label`, owner, `#rows · #hidden`). Drill into one →
- Its rows in current order; rearrange mode (Ctrl+R) reorders/hides/sets-divider live via the
  same `GuiLayout*` API. `watch_file` on `gui_layouts.json` so edits elsewhere refresh live.
- This is itself a Miller using the layout system on its own row list — dogfooding.

This is the "get lists of our layouts, all accessible, build links between them" capability.

---

## 6. Testability (Jamie's standing interest — make Millers easier to test/build)

- `_MlApplyLayout(nodes, layoutEntry)` is a **pure function** → direct unit tests: feed a node
  array + a layout, assert the output order / hidden / divider placement. No GUI, no I/O.
- The existing **headless snapshot** (`MlSnapshot` / `MlSnapshot_Json` / `MlLintTree` in
  MillerNodes.ahk) renders a tree to data without opening the GUI — wire it into the suite so a
  menu's layout-applied structure can be snapshotted + diffed. (This is the "test by rendering,
  not real UI" design that was speced but not yet wired into a runner — adopt it here.)
- Add tests under `Helpers/Tests/Gui/`: `test_layout.ahk` (pure-transform cases) + extend
  `test_miller.ahk` for the rearrange/hide key paths (fixtures, `GuiTestAssertLogContains`).
- `layouts.py` gets its own pytest-style self-checks (order/hide/divider/ensure-merge).

Net: building layouts as a pure transform + leaning on the headless snapshot makes Millers MORE
testable, not less.

---

## 7. Decisions locked (2026-06-16)

1. **Bake layout-apply into the engine**; persistence behind a thin `GuiLayout.ahk` store API. ✓
2. **`layouts.py` + machine-wide `gui_layouts.json`** registry; self-describing (label, owner,
   items, version); listable/queryable/linkable; editable programmatically via the CLI. ✓
3. **Catalog vs Layout**: shared catalog, **per-context layout** (start per-TYPE; id scheme allows
   per-page later). Each layout owns order + hidden + divider, so same catalog arranges differently
   per page. ✓
4. **Reorder UI = Mode A** (Ctrl+R → grab/move/drop; `Del` hide; Esc done). ✓
5. **Hiding = yes**; **sections/divider = yes** (min: one `divider_after`; extensible to N sections). ✓
6. **Layouts browser = build it.** ✓
7. **Editable by Claude/automation = yes** (the `layouts.py` CLI is the API). ✓
8. **Data sections + sort modes (§1.5) = the direction going forward.** Generated lists collapse
   to one slot in layout mode, carry declared sort modes, tunnel-in to switch sort, and a runtime
   hotkey cycles the sort. Build as the last phase; design for it now so the slot model fits. ✓

---

## 8. Build order (post-compaction)

1. `layouts.py` engine + `gui_layouts.json` schema + CLI (`list/get/ensure/move/hide/show/set-divider`)
   + Python self-checks.
2. `GuiLayout.ahk` AHK shim (cached get + the mutators).
3. Engine: `_MlApplyLayout` pure transform + wire `layout_id` into level build + route rearrange
   moves/hides/divider to `GuiLayout*` when the row is layout-managed; show hidden rows dimmed in
   rearrange mode; `Del`/divider keys.
4. `test_layout.ahk` (pure transform) + extend `test_miller.ahk`; wire `MlSnapshot` into the runner.
5. ProjectsMenu adoption: give `_ProjActionLeaves` a shared catalog + per-type `layout_id`
   (`proj.functions.{workspace,project,todo}`), `GuiLayoutEnsure` on build, drop the `act_*` no-op
   in `_ProjOnMove` (engine now routes layout rows). Verify reorder/hide/divider per type.
6. Layouts browser (`OpenLayouts` Miller) + a voice command (ASK Jamie before adding the command).
7. Update skill ref (`miller-authoring.md`) + `gui-conventions.md` with the layout system.
8. **Data sections + sort modes (§1.5)** — the bigger phase, after 1–7 are solid: declare
   data-section slots (`section_id`/`label`/`sort_modes`/`rows(sort)`); engine expands them in
   normal mode, collapses to one slot in layout mode; tunnel-in sort picker; `sorts` persistence +
   `layouts.py set-sort`; runtime sort-cycle hotkey (key TBD). First adopters: Spotify library
   (albums) + the projects todo/album lists; then media catalog broadly.

## 9. Open items to confirm during build
- Exact key for "set divider" (`Del` taken by hide; pick a non-letter — maybe `-` / NumpadSub).
- Whether the layouts browser also exposes catalog labels editing (probably read-only labels).
- Migration: first run with no registry entry → `ensure` seeds order = catalog order, nothing hidden.
- Runtime **sort-cycle** key for data sections (§1.5): non-letter; candidates `Ctrl+S` / a numpad
  key / a Stream Deck button. Decide when building phase 8.
- Slot model must be designed in phases 1–3 so a "data section" is just another slot kind the
  layout orders/hides — i.e. don't bake an assumption that every slot is a single static row.
</content>
</invoke>
