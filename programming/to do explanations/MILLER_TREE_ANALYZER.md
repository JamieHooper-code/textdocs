---
tags: [todo, autohotkey, testing, gui, miller, claude-tooling]
---

# Miller Tree Analyzer — design plan

A no-GUI analyzer that anyone building or editing a [[Miller column navigator]] runs to
sanity-check the node tree before they ever open the window. The Miller's two panes
(left list, right preview) are a **pure function of the node tree**, so almost every
defect — structural, UX, performance, behavioural — is detectable by walking the tree
in memory. This is the "test from JSON, not by clicking around" tool.

Related code today: `Helpers/Gui/MillerNodes.ahk` (`MlValidateTree`, `MlLintTree`,
`MlSnapshot`, `MlSnapshot_Json`), engine `Helpers/Gui/MillerColumnPickGui.ahk`.
Authoring quickstart: `~/.claude/skills/ahk-functions/references/miller-authoring.md`.

## Why

The empty-right-pane Triage bug (a root node was a `do`-leaf where every sibling was a
branch, so selecting it previewed nothing) shipped because nothing checked it without
opening the GUI. `MlLintTree` now catches that one case. The goal here is to generalise
that instinct into a thorough, reusable analyzer so **no Miller regression needs a human
to notice it visually**.

## What exists (baseline)

- `MlValidateTree(rootBuild [,{deep}])` → structural problems: missing/dup keys,
  branch-XOR-leaf, non-empty `cols`, callable `children`/`do`/`search_index`, empty
  builder return. Shallow by default; `deep` drills every branch one level.
- `MlLintTree(rootBuild)` → the above PLUS one UX lint (root leaf ⇒ empty preview).
- `MlSnapshot(rootBuild,{maxDepth})` + `MlSnapshot_Json` → render the tree (incl. each
  node's `preview: children|EMPTY` and `row_actions`) to JSON for diff/inspection.
- `DumpMillerSnapshot(target)` (in `SpotifyTriage.ahk`) → writes snapshot+lint JSON to
  `%TEMP%\miller_snapshot.json` for `library` / `triage`.

## Target design: `MlAnalyze(rootBuild, opts) -> report`

One entry point returning a structured **report**, not just a string list:

```
report := {
    ok:       bool,                  ; no error-severity problems
    problems: [ {severity, category, where, msg, fix} ],
    stats:    {nodes, maxDepth, levels, shellouts, ms, perLevel:[...]},
    snapshot: <MlSnapshot tree>,
}
```

- **severity**: `error` (broken — must fix), `warn` (UX/perf smell), `info` (style).
- **category**: `structure | preview | search | perf | behaviour | a11y`.
- **where**: a path like `root / artists / <id>` so the problem is locatable.
- **fix**: a one-line suggested remedy where one is obvious.

Convenience wrappers: `MlAnalyzeProblems()` (flat strings, back-comp with
`MlValidateTree`), `MlAnalyze_Json()` (report → JSON for the harness / for me to read).

## The checks (the actual value)

### A. Structure (have most; extend)
- node is exactly branch XOR leaf; `key` present + unique among siblings; `cols`
  non-empty and **≤ the level's header count** (a 3-col schema with a 4-col node
  silently drops data — currently unchecked).
- `children`/`do`/`search_index` are callable; builder returns a non-empty array.
- **Cycle / runaway-depth guard:** drilling a chain that revisits an already-seen
  `(level,key)` signature ⇒ infinite-drill risk. Flag and stop.

### B. Preview / UX consistency
- leaf among branches at a level ⇒ empty right pane (have, generalise to any level not
  just root).
- **mixed kinds** at one level (some branch, some leaf) ⇒ `info` (often intended —
  e.g. an "open in browser" leaf beside drillable branches — but worth surfacing).
- **detail-column density:** some siblings have a detail col, others don't ⇒ `info`.
- **label length:** a label that will truncate at the level's column width ⇒ `warn`.
- **row_actions hygiene:** duplicate digits within a node; a digit that collides with
  the row-selection buffer; row_actions present on a node whose level shows no legend.

### C. Search integrity (currently a blind spot — high value)
- For every `search_index` entry, **resolve its `path` against the live tree** (the same
  walk the engine's reveal/drill uses). A hit whose path doesn't resolve silently fails
  to navigate — invisible until a user tries it. Sample- or fully-verify.
- **Search coverage:** flag subtrees that are neither covered by a `search_index` nor
  reachable by the live-walk (so they're invisible to whole-tree search), and the
  inverse — large branches NOT `search_skip`'d that the walker will re-expand on every
  keystroke (the perf lesson from the Spotify library: artists/placeholders are
  `search_skip`'d precisely because the flat index already covers them).

### D. Performance / cost (needs light instrumentation)
- Wrap the per-level build to count **Python shell-outs** (`_RunIngestCapture` calls)
  and **wall-clock ms**. Flag N+1 patterns (a list builder that shells once *per row*),
  slow levels, and oversized levels (≫ a few hundred rows ⇒ wants pagination or
  `search_skip`). Report `stats.perLevel` so regressions are visible in a diff.

### E. Behaviour (dynamic, opt-in)
- **Determinism:** build the same level twice; assert identical key order. Catches
  builders that sort nondeterministically or depend on wall-clock/`Random` (which also
  breaks snapshot diffing).
- **`do`/row-action smoke (guarded):** optionally invoke each `do`/row-action with a
  **stub close + a dry-run flag** to ensure it doesn't throw *immediately*. Off by
  default — side effects — gated behind `opts.fire := true` for fixtures that pass a
  sandbox. Most value comes from A–D without this.

### F. Snapshot regression (golden files)
- `MlSnapshot_Json` → commit a **golden snapshot** per registered Miller. A test diffs
  current vs golden and fails on unexpected structural drift, with a one-command update
  when the change is intended. This is the literal "test the tree from JSON" loop.

## Making it universal — the registry + enforcement

The reason "anyone editing a Miller runs it" only works if it's **automatic**:

1. **Registry.** Each Miller registers `{name, rootBuild}` in one place
   (`MillerRegistry()` static map, or a directory the scaffolder appends to). New
   Millers from `new_miller.py` auto-register.
2. **One command** `AnalyzeAllMillers()` runs `MlAnalyze` over every registered root,
   writes a combined JSON report + a human summary, exits non-zero on any `error`.
3. **Stop-hook gate.** `caster_ahk_verify.py` calls `AnalyzeAllMillers` (cheap static
   pass: A, B, C-paths, F) on every turn that touched a Miller file, and fails the turn
   on `error`-severity problems — same way it already runs ruff / bridge / includes.
   Deep/dynamic checks (D, E) run on demand or in the GUI test suite, not every turn.
4. **Test harness.** Extend `GuiTestAssertMillerValid` → `GuiTestAssertMillerClean`
   (asserts zero `error` problems via `MlAnalyze`). The scaffolder's generated test
   calls it.

## Phasing

- **P1 (small, do first):** grow `MlLintTree` into `MlAnalyze` with the report shape +
  all cheap static checks (A, B, the col-count + cycle guards). Wire `MlAnalyze_Json`
  into `DumpMillerSnapshot`. ~persisted JSON I can diff.
- **P2:** search-integrity (C) — path resolution against the live tree. This is the
  highest-value novel check and reuses the engine's `_Mcp_NavigateToPath` logic.
- **P3:** registry + `AnalyzeAllMillers` + Stop-hook gate + harness assertion (the
  "everyone runs it automatically" piece).
- **P4:** perf instrumentation (D) + determinism (E) + golden snapshots (F).

## Open questions
- Registry: explicit `MillerRegister()` calls vs a scaffolder-maintained manifest file?
- Golden snapshots: where do they live + what's the one-command update ritual?
- Should the Stop-hook gate be hard-fail on `error` or warn-only until the back-catalog
  of existing Millers is clean?

See also: [[TESTING_INFRASTRUCTURE_EXPANSIONS]], [[SPOTIFY_SCRAPING_INTERNALS]],
[[AHK_FUNCTION_TESTING]].
