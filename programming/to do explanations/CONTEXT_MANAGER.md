---
tags: [design, caster, voice, contexts, miller, context-manager, build-spec]
related: ["[[PER_CONTEXT_MATERIALIZER]]", "[[VOICE_COMMAND_SYSTEM]]"]
status: BUILT — all 9 facets live + deep-self-tested; left: deep-link phrases, cross-Miller mounts, engine backlog
updated: 2026-06-26
---

# Context Manager — design spec

## 0. The big idea (Jamie's organizing principle)

The system feels scatterbrained because every context facet has its own little tool
(`AddContext`, `AddProgramByVoice`, `AddListNavSite`, hand-edited JSON for scroll/noise…).
**Collapse them into ONE unified Miller that is the single home for everything about a context.**
This is part of a broader direction — a *few* unified Millers (Registry Editor, Context Editor,
Voice Command Editor), all sharing the same node machinery, fav/hide cycle, search, and **deep-links**.
Rule going forward: **anything new about contexts mounts HERE**, never as a new standalone tool.

Full context schema + which subsystem owns each field: ahk-functions `references/directory-program-registry.md`.
Don't re-document the schema here — this doc is about *editing* it.

---

## 1. A context's facets (what we're editing)

`Edit` = inline in this editor; `Route` = the data lives elsewhere (UIA dump / bindings.json) so
the node links to the owner-flow instead of hand-typing. Default is inline; only two honest Routes.

| Facet | Fields | Powers | Edit mode | v1 |
|---|---|---|---|---|
| **Identity** | `display`, `parent` | the chain + inheritance | inline (text; parent = context picker) | ✅ |
| **Matching** | `match.exe/class/title_regex[]/url_contains/url_regex` | foreground detection (every gate) | inline (text; title_regex = add/remove rows) | ✅ |
| **Program** | `program.open_fn/launch_fn/default_snap` | `open <program>`, launch+snap | inline (pickers: valid `Open*` fns + snap aliases) | ✅ |
| **Voice scoping** | `caster_scope` | per-context voice commands | inline — **BUILT** | ✅ |
| **Scroll** | `scroll.amount_down/up/x/y/enabled` | context-aware scroll | inline (number + toggle) | ✅ |
| **Noise** | `noise` | ignore transient windows | inline (toggle) | ✅ |
| **Routing** | `destination.store` (`registry:<id>` / `links`) | "add site" classification | inline (picker: registry or links) | ~ |
| **ListNav** | `nav_mode`, `selectors`, `geometry`, `post_click_function` | `jump <n>`/`next`/`focus me` | **Route** → UIA-dump flow (`AddListNavSite`); `nav_mode` inline | route |
| **Bindings** | (live in `bindings.json`, keyed by token) | macros active in this context | **Route** → the macro/binding editor; show count | route |

---

## 2. The editor = a NESTED tree of per-context FACET HUBs

**The tree is BY PARENT** (Jamie 2026-06-26): the root shows only top-level contexts; a context
drills to its **child contexts (nested)** then a divider then **its own facet hub**. So
`chrome ▸ youtube ▸ youtube_video`, children resolving inside their parents — not a flat list of 56.
(Nested paths drive drill/reveal/reopen; cycle-guarded against malformed parent data.)

A context's facet hub: its children are its facets. **Present** facets show status+value;
**absent** facets show `+ add <facet>` (so "make launchable", "enable ListNav", "enable scoping"
are all the same gesture — adding a facet). Drilling a facet opens its inline field-editor, or (for
the two Routes) the owner-flow. Editing and *adding* a facet are the same path — that's how
"everything goes through here" stays true.

```
<context>  "YouTube"   chrome ▸ youtube · scoped · 3 cmds · parent: chrome   (branch)
   Identity        display "YouTube" · parent chrome        → fields
   Matching        exe chrome.exe · url youtube.com          → fields
   Program         (not launchable)  + make launchable       → fields / add
   Voice scoping   ON  chrome ▸ YouTube                       → enable/edit/disable   [BUILT]
   Scroll          custom (9/11 @ .92,.5)                     → fields
   Routing         registry:youtube_videos                   → picker
   ListNav         off   + configure (UIA)                    → routes out
   Bindings        2 macros here                              → routes to macro editor
   Noise           false                                      → toggle
   ── Pin (cycle) ──   ── Remove ──
```

---

## 3. Inline editing model

Each facet is a branch of **field rows**; Enter on a row opens a single-field input or a picker,
writes via `WriteContextProfile` (passthrough-safe — it recursively encodes nested objects like
`caster_scope`), then reopens at the same node. Pickers (not free-text) where a value is constrained:
- **parent** → drill the contexts list.
- **program fns** → a picker of valid `Open*`/launch fns (signature-checked).
- **routing** → pick a registry id or `links`.
- **title_regex** → an array facet: rows of patterns with add/remove.
Simple facets (noise toggle, scroll numbers, display text) are one input. No giant flat form.

---

## 4. Deep-linking — the unifier (mechanism already exists)

The Miller takes an `initial_path`, so any node is addressable. A thin voice command opens THIS
editor landed on a node: `"open nav"` → `OpenContextEditor(currentToken, ["listnav"])`,
`"open scoping"` → `[token, "scoping"]`, etc. N small phrases, ONE editor — that replaces the
scattered standalone tools with entry points into the unified Miller. (Example only — wire the
specific phrases later; the capability is free.)

---

## 5. Add a context

`+ Add context` → one prompt (token) → drill straight into the hub (§2) with foreground-seeded
defaults (display + match.exe from the active window). Absorbs/retires `AddContext`; over time
`AddProgramByVoice` becomes "+ make launchable" inside the hub.

---

## 6. Relationship to the Registry Editor (the redundancy resolution)

Same Miller **pattern** (good — shared machinery), different **domain**: the Registry Editor edits
flat **dictation value-lists** (sites/docs/contacts — key→value); the Context Editor edits rich
**app/page profiles** (9 facets). A context is too structured to be a 57th flat registry. So they're
**siblings in one navigator**, cross-linking only on `destination` (a context routes "add site" into a
registry). Not redundant — the lookalike feel is consistency.

---

## 7. Safety + reboot

- **Remove / disable-scoping:** confirm modal lists dependents (program / scoping / N scoped commands /
  M sub-contexts); offers **rescope-its-commands-to-global** so nothing silently goes inert. v1 Remove
  deletes the whole file (program-only-strip later). **BUILT.**
- **Reboot:** field edits → none (`WriteContextProfile` invalidates cache + touches the Caster rule).
  Enable/disable/edit **scoping** → one reboot (rule file add/remove/RuleDetails change) — flagged in GUI.

---

## 8. Reuse map (condensed)

Reuse: `_MillerColumnPickGui` + `MlLeaf/MlBranch`, `_RegPartitionByPref`/`_RegPrefCycleRow` (shared
fav/hide CYCLE), `_ConfirmationModalGui`, `_MultiFieldInputGui`, `WriteContextProfile`/`DeleteContextProfile`/
`GetContext`/`LoadContexts`, the whole scoping backend ([[PER_CONTEXT_MATERIALIZER]]). Prefs: `set_pref`
scope `"contexts"`. New: `ContextEditorMenu.ahk` + the `contexts` data provider in `generic_command_store.py`.

---

## 9. Status

**BUILT + self-tested (deep tree valid, 56 contexts):** data provider (`contexts` + `context-commands`);
**nested-by-parent tree** (chrome ▸ youtube ▸ …, cycle-guarded, nested reopen paths); single **Pin cycle**;
**Identity** (display/parent) + **Matching** (exe/class/title_regex/url_*) inline field-editors; **Add** flow
(drills into the hub; replaces `AddContext`); **Voice scoping** now **inline** (executable/title/display as
editable rows, each re-runs the generator; Enable seeds defaults, no popup) + **scoped-commands view now
EDITABLE** (each command drills into the shared `_CmdActionNodes` cluster — Remove / Rephrase / Rescope,
identical to "show fun"); **Remove** with dependency safety; entry **"open context"** (silent nested
drill-to-current).

**ALL NINE FACETS BUILT 2026-06-26** — the full §2 hub is live + deep-self-tested (56 contexts):
- **Scroll** = `enabled` toggle + `amount_down/up` (int) + `x/y` (float) rows.
- **Noise** = top-level toggle (off clears the key).
- **Program** = `open_fn`/`launch_fn` free-text rows with a fn-index typo guard (warn-but-allow; the index
  can lag a new fn) + `default_snap` fixed picker (top/bot/full/left/right/main). Setting open/launch fn on a
  context with no program block creates it — "make launchable" is just filling a field (Add == Edit).
- **Routing** = `destination.store` picker (none / links / every loaded registry → `registry:<id>`).
- **ListNav** = `nav_mode` picker (list/grid/none/custom) + `post_click_function` free-text inline;
  `selectors`/`geometry` shown READ-ONLY (UIA-derived; pointer to AddListNavSite).
- **Bindings** = read-only count of macros scoped here (bindings.json `scope == token`) + a pointer to the
  macro editor.

Type-safety: numbers/bools write real Integer/Float/`1`/`0` via `_CtxSetRaw` / `_CtxClearPath` / `_CtxTruthy`
(0 is a real value, never the string-clearer). **Latent encoder bug fixed along the way:** `_JsonEncode`'s
`String(Float)` leaked full IEEE-754 precision (`0.92` → `0.92000000000000004`), corrupting scroll coords on
EVERY `WriteContextProfile`; now `Format("{:.15g}", v)`.

**Cross-Miller mounts BUILT 2026-06-26 (§6):** `_ContextEditorAsNode()` (key `ctxmgr`, label "Context
Manager" — `contexts` collides with an existing registry id) now mounts in BOTH the Voice Command Editor
("show fun") and the Registry Editor roots, each a sibling of the Registries mount. It drills the SAME tree
as "open context" and folds every context into the host's recursive search (`_CtxSearchIndex`). Self-arms
its data via `_CtxEnsureData()` in the root + search builders (host preloads nothing). Verified headlessly:
both roots carry the mount, no dup keys, mounted subtree deep-valid. **Seam:** editing a context from inside
a host reopens the STANDALONE Context Manager (actions reopen via `OpenContextEditor`) — browse/search is
seamless; the in-place-refresh engine work removes the reopen.

**LEFT (not facet-editing):** deep-link entry phrases (§4, e.g. "open nav" → editor at the ListNav node);
absorb `AddProgramByVoice`/`AddListNavSite` as the canonical add-paths over time; the shared-engine
nav-memory + in-place-refresh work ([[UNIFIED_MILLERS]] backlog).

**Scoped-commands view is now EDITABLE (portable-node convention — [[UNIFIED_MILLERS]]) — BUILT 2026-06-26.**
Shared cluster `_CmdActionNodes(phrase, fn, reopenFn, opts)` in `Helpers/CommandActionNodes.ahk`: Remove /
Rephrase / Rescope, all store-CLI-backed (live, no reboot). The scoped-command rows are now **branches** that
drill into it (`reopenFn = _CtxReopen(token,"scoping")`, `opts.defaultContext = token`); the Voice Command
Editor's `_VceCommandActions` delegates to the SAME cluster (so a command is managed identically from either
Miller). Rephrase IS included (the wizard is host-agnostic — both run in the GuiHost closure). This is the
reference case for "drop a node-cluster from another Miller in." 5 dead VCE copies deleted in the migration.

---

## 10. Decisions — ALL LOCKED 2026-06-24 (Jamie: "0.00")

- Facet-hub model; **inline-default** — the ONLY routes are ListNav (UIA) + Bindings.
- **Routing facet: inline** (picker: registry id / links).
- **Bindings: a read-only node** here (count + deep-link to the macro editor) — not omitted.
- **Program: inline** with fn pickers now (absorb `AddProgramByVoice` → "+ make launchable").
- Unified — everything contexts goes through this editor; siblings-with-Registry-Editor (cross-link on
  `destination`); Pin = one cycle; "open context" silent drill (no flash); remove offers rescope-to-global;
  scoping changes flag a reboot.
- The "collapse into a few unified Millers" principle gets its own short note: [[UNIFIED_MILLERS]].

## 11. Build order (LOCKED — my call, foundation-first / pain-first)

1. ✅ **Identity + Matching inline editors** — core fields every context has; also IS the Add core.
2. ✅ **Add flow** (§5): create token → drill into the hub; retire `AddContext`.
3. ✅ **Program facet** (open/launch free-text + typo guard, snap picker). `AddProgramByVoice` absorb: later.
4. ✅ **Scroll + Noise + Routing** (numbers / toggle / picker). + the `_JsonEncode` float-precision fix.
5. ✅ **ListNav** (`nav_mode`/`post_click` inline, selectors/geometry read-only) + **Bindings** (count + pointer).
   ↑ ALL DONE 2026-06-26 — also the portable `_CmdActionNodes` cluster (scoped commands editable).
6. **Deep-link entry points** (§4): "open nav" / "open scoping" / … → editor at that node.  ← next
7. ✅ **Mounts** into Registry Editor + "show fun" (§6) — `_ContextEditorAsNode` keyed `ctxmgr`, in both roots.
