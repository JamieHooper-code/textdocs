---
tags: [design, caster, voice, materializer, per-context, build-spec]
related: ["[[VOICE_COMMAND_SYSTEM]]"]
status: BUILT-2026-06-24-pending-one-reboot-live-test
updated: 2026-06-23
---

# Per-Context Materializer — focused build spec

**This doc is self-contained. Build from THIS, not the 1500-line VOICE_COMMAND_SYSTEM.md.**
It exists because that monolith is too big to read reliably (a triage misread it twice).

---

## 1. What it is / why (the one-paragraph version)

The **generic command store** (`rules/generic_command_store.py`) lets a new voice command be ONE
appended JSON row — no Python, no new file. The **global materializer**
(`rules/generic_materializer_commands.py`) turns each `context.kind == "global"` row into a live
Dragonfly grammar at load (and hot-reloads on a marker bump, so new rows go live in ~5s, no reboot).
**DONE 2026-06-22.**

The **per-context materializer is NOT built.** Store rows reserve `context.kind == "context"` but the
global materializer **skips them** (`generic_materializer_commands.py:109-110`). So a command scoped to
"only in VS Code" / "only on YouTube" can't go live, and the **Voice Command Editor's global/context
scope step is blocked** (VOICE_COMMAND_SYSTEM §15.7 phase 5). That scope choice is the visible
GUI gap Jamie noticed.

**The bigger design goal (Jamie, 2026-06-24):** this isn't just "let context rows materialize." The
end state is (a) adding a new context is one JSON file + one reboot; (b) the add-voice-command flow
**auto-detects the context you're in** and ranks it FIRST in the scope picker (then its parent, then
global, then the rest); (c) a Miller **context manager** to favorite / hide / remove contexts, like
the other registries. The `caster_scope`-on-the-JSON design below is the backbone for all three — see
§4 (intrinsic vs view-state split), §8 (ranking + capture-at-entry), §12 (context manager).

---

## 2. The hard constraint (why this needs design at all)

**Caster scopes per RULE**, via `RuleDetails(name=..., executable=..., title=...)`. Every context rule
in the codebase works this way:
- `google_docs_commands.py`: `RuleDetails(name="GoogleDocs rules", executable="Chrome", title="Google Docs")`
- `command_line_commands.py`: `RuleDetails(name="Command Line Rules", executable="WindowsTerminal")`
- `chrome_commands.py`: `RuleDetails(name="chrome rules", executable="chrome")`

`executable` = the process name (substring, case-insensitive; may be a list). `title` = optional window-
title substring. Each rule is enabled by name in `settings/rules.toml → _enabled_ordered`.

There is **no per-COMMAND context** inside one rule — the whole rule shares one context. So to scope
commands per app, you need **one rule per context**. A single global materializer rule cannot do it.

---

## 3. Architecture — CHOSEN: one generated materializer file per context

A small **generator** emits one near-clone of the global materializer per context. Each generated file:
- filters store rows to `context.kind == "context"` AND `context.token == <thisToken>`,
- registers a `MappingRule` scoped via `RuleDetails(name="generic materializer <token>", executable=…, title=…)`,
- builds actions/extras with the SAME logic as the global one.

**Cost model (identical to the global materializer's, which Jamie already accepted):**
- New COMMAND in an existing context → append a row + bump markers → **live in ~5s, no reboot.**
- New CONTEXT (first command for a never-seen app) → run generator + add to `rules.toml` → **ONE reboot.**

Rejected alternatives (don't revisit without reason):
- *One file registering N rules dynamically* — Caster's loader calls `get_rule()` once per file (returns a
  single `(Rule, RuleDetails)`); multi-rule-per-file is not the supported shape. (If a future Caster API
  makes a file yield many rules cleanly, this collapses to one file — nice-to-have, not v1.)
- *Dragonfly `AppContext` per command inside one rule* — Caster wraps one context per `RuleDetails`; per-
  command context isn't natural. Reject.

This is exactly VOICE_COMMAND_SYSTEM §9's stated plan: "one materializer rule per distinct context …,
each reading its slice of the store … new files … need one reboot."

---

## 4. Context identity — CHOSEN: a `caster_scope` FIELD on the existing context (UNIFIED, Jamie's call)

A `context`-kind row names its context by the **existing context token** (the `INIDATA/Contexts/<token>.json`
filename stem): `"context": {"kind": "context", "token": "vscode"}`. **No parallel registry** — the contexts
registry is the single source of truth.

Scoping lives as a NEW opt-in field **`caster_scope`** on the context JSON, holding the Caster-native
`RuleDetails` values explicitly (NOT derived from the AHK match fields):
```json
// INIDATA/Contexts/vscode.json
{
  "match": { "exe": "Code.exe", ... },                     // AHK-side matching (existing, untouched)
  "caster_scope": { "executable": "Code", "title": "", "display": "VS Code" }   // NEW: Caster scope
}
```
Read by BOTH the generator AND the Voice Command Editor scope picker. A context WITHOUT `caster_scope`
simply gets no materializer file (opt-in).

**Why a dedicated field, not derived:** AHK matches on exe / win_class / `title_regex` (regex) / url; Caster's
`RuleDetails` wants a process name + a plain title SUBSTRING. The shapes differ, so `caster_scope` carries the
Caster values EXPLICITLY rather than trying to translate a regex into a substring. (Optional nicety, not v1:
the generator MAY default `caster_scope.executable` from the context's `program` block exe when omitted —
e.g. `program.exe == "Code.exe"` → `"Code"` — so app-level contexts need only add a title or nothing.)

**Readers tolerate the new field:** AHK `JsonParse` (ContextFunctions) and registries.py read the whole object
and ignore unknown keys, so adding `caster_scope` is additive and safe for every existing context consumer.

**Intrinsic-vs-view-state split (Jamie, 2026-06-24) — DO NOT BLUR:**
- **Intrinsic context data** (`match`, `program`, `parent`, `scroll`, `caster_scope`) → lives ON the context
  JSON. It defines what the context *is*.
- **User view-preferences** (favorite, hidden) → do NOT go on the JSON. They reuse the EXISTING prefs sidecar
  `INIDATA/registry_prefs.json` via `set_pref(scope="contexts", key=<token>, "fav"|"hidden")` — the same mechanism
  every other Miller registry manager uses (floats favs to top, sinks hidden to bottom). Putting `hidden` on
  `youtube.json` would mutate intrinsic data to express a view choice and diverge from the directories/sites
  managers. Keep view-state in the sidecar.
- **Remove a context** = delete its JSON file (already how contexts die) + regenerate (drops its materializer file).

---

## 5. DRY the materializer core first

Before generating N copies, factor the shared logic out of `generic_materializer_commands.py` into
**`rules/_materializer_core.py`**: `_build_action`, the per-slot extras builder (`_build_rule_parts` minus
the global-only filter), the safe-fail loop, the sidecar logger, and the `importlib.reload(generic_command_store)`
guard. Parameterize by a `row_filter(row) -> bool`. Then:
- `generic_materializer_commands.py` calls the core with `filter = kind=="global"` (unchanged behavior).
- each generated per-context file calls the core with `filter = kind=="context" and token==X`.

This avoids drift across N files.

---

## 6. Store changes (`generic_command_store.py`)

- `validate_row`: accept `context.kind == "context"` **iff** `context.token` is a non-empty string that
  resolves to an existing `INIDATA/Contexts/<token>.json` carrying a `caster_scope` field (else error — keeps
  dead/unscopable-context rows out). Global rows unchanged.
- **Marker bump must reach EVERY materializer file.** Today `bump_materializer_marker()` bumps only
  `generic_materializer_commands.py`. Change it to bump the global file AND every
  `generic_materializer_*_commands.py` present, so any store edit hot-reloads all of them. (Cheap; ~5s each,
  Caster reloads in parallel.) ALT considered: a single shared trigger file all materializers `#`-watch —
  more plumbing; do the bump-all loop for v1.

---

## 7. The generator — `rules/gen_context_materializers.py`

Globs `INIDATA/Contexts/*.json`; for every context with a `caster_scope` field, writes
`rules/generic_materializer_<token>_commands.py` from a template (mirrors the global file but: imports
`_materializer_core`, passes the token filter, and `get_rule()` returns `RuleDetails(name="generic
materializer <token>", executable=<caster_scope.executable>, title=<caster_scope.title>)`). Idempotent
(regenerates in place; removes files for contexts whose `caster_scope` was deleted). Prints the rule names to
add to `_enabled_ordered` (or patches `settings/rules.toml` directly + warns a reboot is needed for newly-added files).

---

## 8. Voice Command Editor scope step (unblocks §15.7 phase 5)

The add/edit flow's scope picker (currently always writes `global`):
- Offer **Global** + one row per context that has a `caster_scope` field (show `caster_scope.display`).
- Picking a context writes `context = {kind:"context", token}`.
- If Jamie wants a context with NO `caster_scope` yet → offer "enable scoping here" (add `caster_scope` to that
  context JSON + run generator + register), flagged **"needs one reboot to go live."** Until the reboot the row
  is stored but its rule isn't registered (the "stored + flagged not-live-yet" pattern the doc already anticipates).

**RANKING — the scope picker is auto-identified, not alphabetical (Jamie's explicit ask).** When she promotes a
function to a voice command from inside a context (e.g. a YouTube function), order the choices:
1. **The context she's in** — the LEAF of the live context chain (`DetectContextChain`). (YouTube → "YouTube" first.)
2. **Its parent chain — walked RECURSIVELY (Jamie, 2026-06-24)**, following `parent` upward through every level,
   not just the immediate parent. A deep chain `foo → bar → chrome` yields slots `bar, chrome` in order. Only
   include ancestors that have a `caster_scope` (skip un-scoped intermediate contexts but KEEP climbing past them).
   Guard against parent cycles (track visited tokens; stop on repeat). Dedup against the leaf.
3. **Global.**
4. **Every other `caster_scope` context**, below — later ordered by the prefs sidecar (favs up, hidden out, §12).
Skip slot 2 entirely when the context has no `caster_scope` ancestors (app-level like VS Code → just [VS Code,
Global, …others]).

**CAPTURE-AT-ENTRY (hard requirement).** Snapshot the active context chain **at the moment the add-flow is
invoked, BEFORE the editor GUI takes foreground** — otherwise the detected "current context" is the editor
window itself. Pass the snapshot into the scope picker. This mirrors `AddDirectorySmart`, which already seeds the
path from the foreground (Explorer/terminal/selection) before its own GUI opens. The chain snapshot is the single
new piece of plumbing the ranking needs; everything else (parents, caster_scope) is already on the JSON.

---

## 9. Build order (do in this sequence)

1. Add a `caster_scope` field to the `INIDATA/Contexts/<token>.json` of Jamie's v1 contexts (DECISION 2 below).
2. `_materializer_core.py` (extract shared logic) + refactor `generic_materializer_commands.py` to use it; verify global still works (debug entry + an existing global row fire).
3. `generic_command_store.py`: `validate_row` accepts context+token; `bump_materializer_marker` bumps all materializer files.
4. `gen_context_materializers.py`; generate the per-context files; add their rule names to `rules.toml _enabled_ordered`; **reboot once**.
5. Verify: append a `context.token=="vscode"` row, confirm the command fires in VS Code and NOT elsewhere; append a second context to prove isolation.
6. Wire the Voice Command Editor scope step (§8). Verify end-to-end by voice: add a context-scoped command in the editor, no code, no reboot (context already had a file).

---

## 10. Gotchas (hard-won — bake in, don't relearn)

- **Cached-helper trap:** every materializer file (global + per-context, and `_materializer_core`) must
  `importlib.reload(generic_command_store)` on load, or a store *code* change (not just data) leaves the
  in-process store stale and the rule silently empties. This already bit the global build (VOICE_COMMAND_SYSTEM
  §12b "cached-helper trap"). The core module ITSELF is a cached helper — reload it too from each file, or
  keep the core import-light and reload the store inside it.
- **Safe-fail per row:** one bad row must never break a rule's load (collect + log to a sidecar, skip the row).
- **MappingRule, NOT MergeRule** (non-CCR). MergeRule needs a ccrtype; `validate_rule.py` CI flags the mismatch.
- **rules.toml:** each generated rule name must be in `_enabled_ordered` or it won't load. New files need a reboot.
- **READ THE LOGS:** materializer load problems go to `~/.claude/context/generic_materializer*.log` and the
  Caster message log. Tail them FIRST if a command doesn't register.
- **Grammar dump refresh:** the global file calls `grammar_dump.schedule_dump()` on reload so collision checks
  see new commands; per-context files should too (or the dump goes stale for context commands).

---

## 11. DECISIONS — LOCKED 2026-06-23

1. **Architecture = generated per-context files (§3).** ✅ LOCKED yes.
2. **v1 context list** — ✅ LOCKED: EIGHT (Jamie dropped Anki + Discord 2026-06-24). Tokens + known-good Caster
   values verified against the actual `INIDATA/Contexts/<token>.json` files (executable = case-insensitive
   process-name substring; title = optional plain window-title substring):

   | display       | token         | caster_scope.executable | caster_scope.title | notes |
   |---------------|---------------|-------------------------|--------------------|-------|
   | VS Code       | `code`        | `Code`                  | —                  | app-level; `program.exe == Code.exe` |
   | Terminal      | `terminal`    | `WindowsTerminal`       | —                  | app-level; exe `WindowsTerminal.exe` |
   | Chrome        | `chrome`      | `chrome`                | —                  | BROAD (fires on all Chrome incl. the page-level ones below — intended) |
   | YouTube       | `youtube`     | `chrome`                | `YouTube`          | page-level; `parent: chrome`, url `youtube.com` |
   | Google Docs   | `google_docs` | `chrome`                | `Google Docs`      | page-level; `parent: chrome`, url `docs.google.com/document` |
   | Spotify (web) | `spotify`     | `chrome`                | `Spotify`          | page-level web player; `parent: chrome`. NOTE: real title_regex is `(Spotify\|.+ • .+) - Google Chrome$`; a plain `Spotify` substring catches Library/Home/search but NOT "Artist • Song" now-playing tabs. Acceptable for v1 (most Spotify voice commands are library-side). Revisit if she wants now-playing scope. |
   | Notepad++     | `notepad_pp`  | `Notepad++`             | —                  | app-level |
   | Kindle        | `kindle`      | `kindle`                | —                  | app-level |

   BUILD NOTE: add `caster_scope` to each token's existing JSON (all eight files exist). Overlapping scopes are
   fine: a broad `chrome` command AND a `youtube` command both fire on a YouTube tab (Caster activates every
   matching rule) — that's correct (general + specific). Anki/Discord deferred, not deleted — add later by the
   same one-JSON-field + one-reboot path.
3. **Context identity = the existing context token; scoping lives as a `caster_scope` FIELD on
   `INIDATA/Contexts/<token>.json` (§4).** ✅ LOCKED (Jamie's unification call — one registry, no parallel map).
4. **The `caster_scope` field holds explicit Caster-native values** (executable/title/display), NOT derived from
   the AHK match regex; generator MAY default `executable` from the `program` block exe. ✅ LOCKED.
5. **Marker bump = bump ALL materializer files on any store change (§6).** ✅ LOCKED yes.
6. **v1 = 8 contexts (Anki/Discord dropped); fav/hidden live in the prefs sidecar, NOT the JSON; scope picker is
   auto-ranked (leaf → parents → global → rest) with capture-at-entry.** ✅ LOCKED 2026-06-24 (Jamie's design review).

---

## 12. Context manager in Miller (POST-v1, backend ready now)

Jamie wants a Miller context manager to favorite / hide / remove contexts, like the directories/sites managers.
**No new backend** beyond what v1 builds — it rides the existing prefs sidecar:
- **Favorite / hide** → `set_pref(scope="contexts", key=<token>, "fav"|"hidden")` in `INIDATA/registry_prefs.json`
  (registries.py already has `set_pref`/`_load_prefs`; just add a `"contexts"` scope). Miller floats favs to the
  top, sinks hidden to the bottom — identical to every other registry view.
- **Remove** → delete `INIDATA/Contexts/<token>.json` + rerun the generator (drops its `generic_materializer_<token>_commands.py`
  and its `rules.toml` line). A removed scoped context's stored rows stay in the store but stop materializing
  (safe — they just go inert, same as a disabled rule).
- **The scope picker's "rest" ordering (§8 rank 4) reads this same prefs scope** — so favoriting a context also
  promotes it in the add-flow picker. One prefs source, two consumers (manager + picker).

This is why fav/hidden MUST stay in the sidecar (§4): the picker and the manager share it. Build the manager
whenever; v1's `caster_scope` + the `"contexts"` prefs scope are the only prerequisites, and v1 delivers both.

---

## 14. BUILD LOG — 2026-06-24 (what shipped)

All code built + statically verified (py_compile, store validate, ahk.py validate, generator
idempotency). **Only the one-time reboot + live voice test remains.** Files:

- **8 context JSONs** — added `caster_scope` (BOM preserved). `code, terminal, chrome, youtube,
  google_docs, spotify, notepad_pp, kindle`.
- **`rules/_materializer_core.py`** (NEW) — shared engine (`build_rule_parts(row_filter, debug_phrase,
  debug_text, sidecar)`, `build_action`, `refresh_grammar_dump`, store-reload). Reloaded by every
  materializer file (cached-helper trap).
- **`rules/generic_materializer_commands.py`** — refactored to a thin GLOBAL caller of the core.
- **`rules/generic_command_store.py`** — `context_caster_scope`, `list_scoped_contexts`,
  `_context_parent`, `scope_rank` (leaf→recursive parents→global→rest), `context_of_phrase`;
  `validate_row` accepts `kind=="context"`+token (rejects unscoped); `bump_materializer_marker` →
  `materializer_files()` bumps ALL 9; CLIs `scopes`, `scope-rank`, `context-of`, `add --context`.
- **`rules/gen_context_materializers.py`** (NEW) — generator; emits `generic_materializer_<token>_commands.py`
  per scoped context, idempotent, cleans up stale files, dual-registers each CLASS name in BOTH
  `rules.toml _enabled_ordered` AND the `[whitelisted]` boolean table.
- **8 generated `generic_materializer_<token>_commands.py`** — registered in rules.toml.
- **`~/.claude/helpers/voice_wizard.py`** — `assign()` now writes `{kind:context, token}` (token-based,
  matches the store; the old exe/title-on-row form is gone), resolves token→caster_scope for collision
  scoping, `live` via per-context-file existence; CLIs `--context`, `scope-rank`.
- **`Helpers/VoiceCommandEditorMenu.ahk`** — `VceLeafContext := DetectContext()` captured at host boot
  (pre-GUI, first-open only); `_VceAdd` passes it to the wizard.
- **`Helpers/Gui/VoiceWizardAssign.ahk`** — `AssignVoiceToFunction(fn, replacePhrase, leafContext)`;
  `_VWPickScope` (ranked numpad picker, Enter=current context); scope threaded into options+assign;
  rephrase PRESERVES scope via `_VWScopeOfPhrase` (`context-of`).

### Live test plan (after ONE Caster reboot registers the 8 new rule files)
1. **Isolation via debug entries (no store changes):** in YouTube say **"materializer youtube test"** →
   types "Materializer YouTube Working". Say it in VS Code → nothing. In VS Code say
   **"materializer code test"** → works; in YouTube → nothing. (Each generated file has its own
   `materializer <token> test` debug entry.)
2. **Store→context path:** add a youtube-scoped row, e.g.
   `py generic_command_store.py add "scope probe" SomeSafeFn --context youtube`, then say "scope probe"
   in a YouTube tab (works) and elsewhere (silent). Remove with `... remove "scope probe"`.
3. **Editor scope picker:** while in YouTube, "show fun" → pick a function → "add voice command" → the
   scope picker shows **YouTube** as option 1, **Chrome** as 2, **Global** as 3. Assign to YouTube,
   confirm it's live there only.

---

## 13. Build order delta (folds §8/§12 into §9)

§9 still holds. Two additions so the backend lands ready for the GUI work:
- **Step 3** also: confirm `set_pref` accepts a `"contexts"` scope (it's scope-agnostic today — verify, no code likely).
- **Step 6** (editor scope step) now includes: capture-at-entry context snapshot + the leaf→parents→global→rest
  ranking (§8). The Miller context manager (§12) is its own later task, not part of v1.
