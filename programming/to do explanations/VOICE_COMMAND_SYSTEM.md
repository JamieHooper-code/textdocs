---
tags: [design, caster, voice-commands, autohotkey, search, macro-wizard, planned]
created: 2026-06-22
updated: 2026-06-23
status: design
---

# Voice Command System — Unified Catalog, Search, Collision & Creation

The full design for reworking how Jamie's Caster voice commands are **catalogued,
searched, checked for collisions, and created**. The end goal: be as imprecise as
possible when searching, never create a colliding/off-convention command, and add
new commands as **pure data** (no Python) — with a **function-first creation wizard**
built last on top of stable interfaces.

This doc is the scoped plan agreed before any building. Build phases are at the bottom.

Related: [[reference_computer_layout]] (memory), [[qmd-warm-search-gpu]], [[CLAUDE_INTERACTION_EFFICIENCY]], [[macro_system_design]] (AHK docs), the `caster-voice` and `ahk-functions` skills.

---

## 1. Goals & non-goals

**Goals**
- Imprecise search that still finds the right command (drop/reorder/skip words).
- A catalog that is **exhaustive** — every command, including JSON/INIDATA-backed ones.
- **Context-aware collision detection** run on every command creation (before *and* after).
- Adding a command is **data-only** in the overwhelming majority of cases.
- A **function-first** creation flow: from any surface showing a function, press a button → assign it a voice phrase.
- Phrase **norms** that are checkable, not just advice.
- rdescript enrichment for searchability via the **local LLM**, not Claude.

**Non-goals (deliberately deferred)**
- Migrating voice commands into `bindings.json` / full trigger unification with the macro wizard.
- Dynamic JSON-driven grammars for *all* commands (only a contained generic lane — see §9).
- Ingesting `bindings.json` into the catalog now (design the shape for it; build later).
- Subsequence/fuzzy tier-3 search and phonetic collision lint (post-v1).
- Auto-creating commands without confirmation. **Every command is confirmed. Always.**

---

## 2. Current-state audit (the root causes)

| Symptom | Root cause | Evidence |
|---|---|---|
| "show voice toggle monitor" misses "toggle debug monitor" | Search is a single **contiguous-substring** `InStr` match | `Helpers/Gui/PersistentLoopGui.ahk:545` (left pane), `:704` (secondary pane) |
| Picker takes a couple seconds to open (didn't used to) | Two **O(n²) insertion sorts** over a **5,882-row** catalog, rebuilt every open in the keystroke-spawned AHK process | `Helpers/VoiceCommandBrowserFunctions.ahk` `_VCB_SortByGenericFirst` / `_VCB_SortStrings` |
| Claude can't find Jamie's JSON/data commands → creates collisions | Catalog expands JSON stores via a **hand-maintained allowlist** + one narrow `hardcoded_*_slots.json` convention; ~14 command-bearing stores are invisible | `Scripts/codebase_tools/voice_commands_by_tag.py:80-135` (`EXPLICIT_CHOICE_MAP`, `_build_choice_maps`) |

**Invisible-to-catalog stores** (in `INIDATA/VoiceChoices/`, 41 total): `searches.json`,
`watch_shows.json`, `reading_books.json`, `remote_connections.json`,
`remote_password_slots.json`, `project_aliases.json`, `playlists.json`, `list_tags.json`,
`switch_desk_modes.json`, `pedal_modes.json`, `completion_friends.json`, … — these back real
spoken commands but never reach the catalog, so neither qmd, `ahk_search.py --voice`, nor any
collision check sees them.

**Context scoping is by exe + title** (and `function_context`), already used heavily:
`youtube_commands.py` → `executable=["chrome"], title="YouTube"`; same for Netflix, Spotify,
Google Voice (`title="Voice - "`), Google Docs. So context-scoped phrases genuinely don't exist
outside their context — faster recognition, no collisions. Collisions therefore only matter
**within overlapping contexts** (see §6).

**The dispatch feed already exists and is rich.** `ahk_event.jsonl` logs every MAINFUN dispatch
with `fn`, `args`, `src` (`keyboard:` / `voice:` / `streamdeck:`), `dur_ms`, `file`, and the
**`win` context (exe+title) at call time**. `Scripts/codebase_tools/macro_setup_usage_audit.py`
already derives usage / cold-lists from it. Voice usage is already counted here (function-backed
commands only — inline `Key()`/`Text()` commands never hit MAINFUN, so they're invisible; another
reason to push logic to AHK).

---

## 3. Architecture overview

Every "assign something to a trigger" flow is the same pipeline with interchangeable ends:

```
        PICK A FUNCTION              CHOOSE SCOPE            ATTACH A TRIGGER
   ┌────────────────────────┐   ┌──────────────────┐   ┌──────────────────────┐
   │ ahk_event.jsonl recent │   │ global           │   │ key+slot → bindings  │ ← macro wizard (today)
   │  (ShowRecentFunctions) │ → │   or             │ → │  .json (ResolveBinding)│
   │ + qmd/ahk_search full  │   │ a context        │   │ phrase+ctx → JSON     │ ← voice wizard (final)
   │   inventory            │   │ (exe/title)      │   │  store (materializer) │
   └────────────────────────┘   └──────────────────┘   └──────────────────────┘
       SHARED                       SHARED (linked)          DIFFERENT ends
```

The **left two stages are identical** for macro keys and voice. Only the trigger end differs.
Two stable interfaces make everything plug in (see §11):

1. **`(name, args, meta) → assign-voice`** — any function-bearing surface calls this.
2. **`trigger record → catalog`** — any trigger source feeds the one index.

Build to those two contracts and current/future surfaces (including the eventual
ShowRecentFunctions upgrade) plug in without touching the core.

---

## 4. The Command Record schema

One shape, trigger-agnostic, that every flow produces and the catalog stores:

```jsonc
{
  "action":  { "function": "ToggleDebugMonitor", "args": [], "kind": "ahk_fn" },
  "scope":   { "kind": "global" | "context", "context": "youtube" | null,
               "caster": { "exe": ["chrome"], "title": "YouTube" } | null },
  "trigger": { "kind": "voice" | "key" | "pedal" | "streamdeck",
               "phrase": "toggle debug monitor",         // voice
               "device": null, "slot": null, "layer": null },  // key/pedal
  "meta":    { "tags": ["debug","window"],
               "source": "rules/...py:NN"  |  "INIDATA/VoiceChoices/sites.json",
               "search_terms": ["telemetry","dispatch tooltip","overlay"],
               "src_of_truth": "live_grammar" | "static" }
}
```

`bindings.json` already stores the key/pedal variant. Voice rules + JSON stores are the voice
variant in different physical form. The catalog ingests both into this shape. **Provenance
(`meta.source`) is mandatory** — it's the direct fix for "I can't find the JSON commands."

---

## 5. The catalog layer (the trunk — replaces today's generator)

This single milestone fixes **both** slowness *and* incompleteness, and adds enrichment.

**5a. Derive from the live grammar, not static parsing.** A Caster-side exporter walks the
actually-loaded Dragonfly rules and dumps every registered spec + its exact context (exe/title)
+ Choice keys. This is **ground truth** — no allowlist, no Python re-parsing, captures
JSON-expanded Choices exactly as Dragon hears them. Static file parsing survives only as an
offline fallback. *Rationale: our planned "auto-discover stores + parse the consuming rule"
just relocates the silent-incompleteness bug; the live grammar eliminates it.*

> **Feasibility spike — DONE 2026-06-22 (confirmed feasible, no Caster perf impact).** Dragonfly
> install: `…\Python310-32\Lib\site-packages\dragonfly\`. API verified: `engine.grammars`
> (`engines/base/engine.py:95`) → `Grammar.context` (`grammar/grammar_base.py:190`) +
> `Grammar.rules` → `MappingRule.specs` (`grammar/rule_mapping.py:159`); Choice values in the
> rule extras. **Performance:** the dump is a read-only walk of in-memory objects — `AppContext`
> stores exe/title as plain attributes (`grammar/context.py:251`), so reads do **no** window
> query/I/O, and enumeration adds **no** per-utterance recognition hook. Cost ~sub-100ms, one-shot.
> **Two rules to keep it free:** (1) trigger on-demand / after-reload, never via a per-utterance
> RecognitionObserver; (2) keep sort/cluster/LLM-enrichment OUT of the Natlink process (in-process
> part only walks grammars → writes raw JSON; external Stop-hook Python does the rest).
> **Constraint:** must run *inside* the Natlink/Dragon process (a standalone `python -c` sees no
> loaded grammars) → it's a small Caster-side module; cleanest trigger = after a rule reload completes.
>
> **Dumper BUILT + RUNTIME-VALIDATED 2026-06-22:** reloaded + ran "make catalog" → `ok:true`,
> **83 grammars / 82 rules / 1,441 templates + 4,093 Choice keys ≈ 5,362 effective concrete commands**
> (vs 2,723 in the static catalog — captures ~2× more, including the previously-invisible JSON stores:
> `<spot_slot>`=182, gv `<name>`=69, `<send_choice>`=43, etc.). Contexts captured correctly
> (youtube/chrome, netflix, spotify, google docs, vscode/code, …). Read-only walk ran instantly during
> reload — no recognition impact. Fixed post-test: version detection now uses
> `importlib.metadata.version("dragonfly2")` (top-level `__version__` doesn't exist) so the guard is
> silent on 0.35.0 and only fires on a real change. Build details below. ⬇
>
> **Dumper files:** `rules/grammar_dump.py`
> (`dump_grammar()` → `.claude/context/voice_grammar_dump.json`) + trigger `rules/grammar_dump_commands.py`
> ("make catalog", `GrammarDumpRule`, enabled in `rules.toml`). Compiles on Python310-32 + passes the
> Caster validator. **Update-safety:** lives in the user rules dir (NOT `site-packages\dragonfly`), so
> `pip -U` can't overwrite it. Reads 4 private dragonfly attrs (`AppContext._executable/_title`,
> `MappingRule._extras`, `Choice._choices`) — guarded by `getattr` + per-grammar/per-rule try/except +
> a `DRAGONFLY_TESTED_VERSION = "0.35.0"` mismatch warning in the output. After any dragonfly update:
> run "make catalog", check `warnings`/`command_count`, re-verify the 4 attrs against the header, bump
> the version constant. TODO: external generator should fall back to the static catalog if the dump is
> empty/failed. NEXT: reload → say "make catalog" → inspect the JSON (esp. that JSON-backed commands now appear).

**5b. Provenance via load-time stamping.** The live export knows phrase/context/Choice-keys but
not "this Choice came from sites.json." Rules stamp their source at load (or a store→rule map
joins it in), so each record carries `meta.source`. Nothing gets *less* detailed than today.

**5c. Pre-built, pre-sorted JSON output.** All clustering + sorting moves into Python (Stop hook),
emitting a ready-to-render catalog. The AHK picker just loads & renders — **kills the O(n²) sort
in the keystroke-spawned process.**

> **Dump → picker Choice expansion DONE 2026-06-22.** The "show voice" picker reads the generated
> TSV, whose Choice expansions came from the brittle `CHOICE_JSON_MAP` allowlist (keyed on var name)
> — so `text <name>` (gv contacts, var `name` ≠ allowlist's `contact`) never expanded and "text
> Spanish" was invisible. Fix: `voice_commands_by_tag.py` `expand_entry` now pulls per-phrase Choice
> keys from the **live dump** (`_load_dump_slots`, ground truth) first, allowlist as fallback. TSV
> 3,274 → 6,534 rows; "text Spanish"/"call Spanish" + all runtime-assembled Choices now appear.
> Verified via screenshot. (Full dump→catalog *join* — replacing the static command list with the
> dump for desc/tags too — is the larger remaining rebuild; this fixed the user-visible expansion gap.)

**5d. Incremental LLM enrichment.** A `local_llm` task (`voice_enrich`) emits `search_terms`
(synonyms, alt phrasings, domain words) per command. **Cached by a `phrase+desc` hash; only
new/changed commands are enriched** — never re-run qwen over 5,882 rows every Stop. Output
constrained (short, lowercased, deduped vs a stoplist) to avoid polluting search.

**Coverage validation:** the generator must emit a list of any commands whose context/source it
*could not* resolve, so gaps are visible, never silently dropped.

Outputs stay at `C:\Users\jamie\.claude\context\voice_commands_by_tag.{md,tsv,json}` (consumed by
the AHK reader, qmd, `ahk_search.py --voice`, and `voice_index`). Fixing completeness here fixes
all of those at once.

---

## 6. Search & collision — the `voice_index` module

One module, three subcommands, all reading the exhaustive catalog (the read-half of the same
loader). Consolidated name (no separate `voice_check`).

- **`voice_index search "<terms>"`** — discovery. Fast/offline, token-subset + prefix + the
  `search_terms` field. The `ahk_search.py` analog for commands.
- **`voice_index check "<phrase>" --context <global|exe/title>`** — collision + lint. **Run before
  writing any command and again after, every time.** Low-token compact output.
- **`voice_index coverage`** — a *join* over the existing usage feed + trigger index: functions
  used-but-unbound (→ create), commands defined-but-never-used (→ refactor/remove), functions never
  dispatched (cold). Reuses `macro_setup_usage_audit.py`; builds no new data source.

**Collision semantics (must be precise):**
- **Context overlap**, not equality. **Global overlaps *everything*** (a global phrase collides with
  any context phrase, since both are active in-context). **Title match is substring** → `title="Google"`
  overlaps `title="Google Docs"`; compute overlap accordingly, don't assume disjoint.
- **Three collision kinds**, not one: (a) identical phrase, (b) **prefix-shadowing** siblings
  (one phrase is a token-prefix of another), (c) **homophones** (Dragon hears "see"/"ski",
  "to"/"two") — textual at v1, phonetic-proximity warning deferred to v2 (shares the §7 phonetic model).

**In-GUI fuzzy matching** (the live picker filter) replaces the `InStr` at PersistentLoopGui
`:545`/`:704`: token-subset (each filter word matches somewhere, order-independent) + word-prefix,
ranked. Tier-3 subsequence deferred. **Cross-decision bug:** ranking reorders rows, which breaks
`numbering:"stable"` (digit N ≠ catalog index N). When an alpha filter is active, switch to
**session numbering** (1..N of current matches) so numpad picks stay correct.

---

## 7. Phrase norms (checkable, not just advice)

**Grammar:** verb-first, two words, one-syllable verb. Domain-first **only** at genuine scale
(hundreds — Spotify, searches). (Flips the skill's old "3+ siblings" rule to Jamie's actual pref.)

**Verb lexicon (role-driven):**

| Verb | Onset | Role | Example |
|---|---|---|---|
| open | vowel | persistent open, leave it up | open chrome |
| see | /s/ | transient pull-up / glance, not kept | see calendar |
| make | /m/ | perform / create / do | make commit |
| show | /sh/ | reveal a panel / list | show voice |
| add | vowel | **append to a list/store** | add site |
| start | /st/ | start (restart if already open) | start stream |
| send | /s/ | dispatch outward | send draft |
| save | /s/ | persist current state | save slot |
| hide | /h/ | dismiss without closing | hide messages |

**Sound rules:**
- Prefer soft onsets: s, sh, m, n, f, l, h, w, v (carry into the next word).
- Avoid hard plosive onsets, esp. /k/ and hard-c (kill, copy, cut, kit). An /s/-led cluster
  softens an unavoidable /k/ ("skit" > "kit"). `/st/` clusters (start, stop) are fine — sibilant-led.
- Avoid bare one-word commands, monosyllables especially — Dragon needs the second syllable to lock on.

**Lint rule — verb+value as a whole** (Jamie's formulation, adopted):
- **Structural** (must carry a verb): applied to the *full materialized phrase*. A bare-noun Choice
  command with no verb template is flagged ("the office" alone fails; "watch the office" passes —
  verb = template's "watch").
- **Phonetic-onset**: applied to the **verb only**. Proper-noun values ("Atlantic Logistics") are
  never held to onset/syllable rules — you can't re-sound them.

Lives in `caster-voice` skill: a compressed block in `SKILL.md` (loaded every voice task) + a fuller
`references/phrase-design.md`.

---

## 8. rdescript enrichment

No rdescript *generator* exists today — rdescripts are hand-authored `[phrase] -> chain | desc`,
and `caster_rule_postwrite.py` only auto-brackets + bumps the reload marker. Enrichment goes in the
**derived index, not the source line** (the rdescript is logged on every recognition — keep it terse).
Mechanism = §5d (`voice_enrich` local_llm task → `search_terms` on the catalog record). The
local-LLM gateway already exists: `Scripts/local_llm/local_llm.py` + `tasks.json` (task→model router,
qwen2.5:7b-instruct, `keep_alive:0`). Add one task entry; no code change.

---

## 9. Generic command store + materializer (zero-code adds — core architecture)

Two tiers of data-entry so new commands almost never touch Python:

- **Domain Choice stores** (existing): `sites.json`, `watch_shows.json`, … — structured value sets
  (url+title, service+url). Keep. Adds go through `add_voice_choice.py` (already supports add /
  `--remove` / `--replace-phrase` rename-collapse / `--peek`).
- **One generic command store** (new): typed JSON rows `{phrase, function, args, arg_kinds, context,
  tags}`. A small set of **dynamic "materializer" rules — one per context** — read the store and
  register grammars at load.

**Net:** adding a command = appending one JSON row. The bridge-variant code-gen we dreaded
(`mainfun_action` vs `named_mainfun_action_with_args` vs `_with_two_extras` vs `_with_number` vs
`to_snake_case`) is solved **once** inside the materializer's `arg_kinds` interpreter, not
regenerated per command. The wizard then only ever writes data.

**Caveats baked in:**
- Materializer must **validate + safe-fail per row** — one bad entry cannot break a grammar's load.
- Caster scopes per *rule*, so one materializer rule per distinct context (global, YouTube, Spotify, …),
  each reading its slice of the store.
- Storage stays **native** (its own JSON), *not* `bindings.json` — zero-code adds without the risky
  full trigger-unification we deferred.
- The genuinely-novel grammar shape (rare) still needs hand-written Python.

---

## 10. Refactor-audit tool

Scans all rules + the catalog, flags two checklists for Jamie to walk through (no auto-changes):
- **Convention violators** — domain-first that shouldn't be, harsh onsets, bare one-word commands
  (uses the §7 lint).
- **Caster-hardcoded logic** — rules with inline multi-step `Key()`/`Playback()` chains that should
  be AHK functions. Single keystroke/text inline stays; >2-step gets flagged to refactor or remove.

Also surfaces `voice_index coverage`'s dead-command list (defined-but-never-used). Supports both
JSON-store and rule-file renames/deletes (`add_voice_choice.py --replace-phrase` for stores; guided
edit for rule files). **First worked refactor:** collapse `stream kill/start/restart` → **`start stream`**
(start = full restart if already open).

---

## 11. The two stable interfaces + the function-first wizard

**Interface 1 — `(name, args, meta) → assign-voice`.** The wizard is **not** a thing Jamie opens and
then speaks into. It's an **action attached to any surface showing a function**: ShowRecentFunctions
rows, the macro wizard, command browsers, bindings views. The function is *already selected* (it's what
she's looking at), so the flow shrinks to:

```
phrase entry → voice_index check (collision + lint) → scope (global / context,
  context auto-suggested via §12 link) → optional args → write (JSON store, §9)
  → enable/reload → record to catalog
```

**Interface 2 — `trigger record → catalog`.** Any trigger source feeds the one index (§4).

Because both the catalog and the assign-action consume the stable `(name,args,meta)` contract — never
the picker's internals — the planned ShowRecentFunctions upgrade is transparent: as long as it can hand
a function identity to the assign-action, nothing downstream changes.

**Prefer data-append over code-gen:** the wizard's first decision is "does this fit a Choice template
or the generic store?" → if yes, append JSON (safe, light file-touch reload); Python code-gen only for
genuinely-new structure. Likely ~80% of creates never touch code.

---

## 12. Relationship to the macro wizard

| Stage | Macro wizard (today) | Voice wizard (final) |
|---|---|---|
| Pick function | `ahk_event.jsonl` recent + bindings for slot | **same** recent feed + qmd/inventory search |
| Scope | global / AHK context (`DetectContextChain`) | global / **Caster** exe+title context |
| Trigger | press key → `bindings.json` | say/type phrase → JSON store (materializer) |
| Resolve | `ResolveBinding` | Dragonfly grammar |
| Collision/discovery | reads the unified index | reads the **same** index |

They're the **same wizard with two trigger front-ends**. The only seam is scope naming: macro uses
the AHK Contexts registry; voice uses Caster exe/title. **Derive the link, don't hand-maintain a map**
(a Caster rule already declares exe+title; match against the AHK context's title/url fields, manual
override only — avoids re-creating the allowlist disease we just cured). Eventual convergence (one
"assign trigger → key/phrase/pedal/SD" flow) is deferred; the record shape already accommodates it.

**Temporary / runtime scoping** (the standalone Macros.md TODO — "only VS Code commands",
"scope to git", "unscope") belongs to this system: a spoken toggle that narrows the *active* grammar
set to one context on demand. It rides on the same context model below and the catalog's per-command
context field. Fold it in here rather than building it separately.

**`function_context` escape-hatch:** for contexts a window *title* can't express (a specific URL/channel),
Caster's `function_context=` scopes the *grammar itself* via a predicate that can call
`IsForegroundContext("youtube")`. This is **not** action-gating (rejected) — the grammar isn't active out
of context, so it stays fast and collision-free. Offer it as the advanced scope option when title is
insufficient.

---

## 12b. Command placement, live-loading & auto-refresh

**The reload constraint (verified in Caster source 2026-06-22).** Caster's file watcher
(`timer_reload_observable`, `reload_timer_seconds = 5`) only tracks files registered at startup:
`BaseReloadObservable._update()` loops over known file hashes, detecting content changes + deletions.
The activator's own docstring: *"Rule reloading does not watch for new files."* Consequences:
- **Editing an existing rule file → live hot-reload, NO reboot.** ✓
- **A brand-new rule file → invisible until a full reload** (which sometimes crashes Jamie's system).
- **Deleting/renaming a file → also needs a reboot** ("Please reboot Caster to re-track").

**Placement conventions (this is what avoids reboots, not just tidiness):**
1. **Put new commands in the existing `<domain>_commands.py` whose domain matches** — by app/context,
   by topic (dev/maintenance → `programming_general_commands.py`; binding tooling → `macro_commands.py`),
   or by the backing AHK function's domain. Editing an existing file hot-reloads live.
2. **Never create a one-off file for a single command** (the `grammar_dump_commands.py` mistake — it
   forced exactly the risky reload Jamie wants to avoid). New files only for a genuinely new domain
   expected to hold several commands — and accept that those need one reboot.
3. **The generic materializer (zero-code adds) is the ultimate fix**: data-driven commands ride ONE
   already-loaded rule's hot-reload → add commands forever, never a new file, never a reboot.
4. **Top-3 placement auto-detect** (to build): when adding a command, score candidate files by
   (a) backing function tags/file, (b) context exe/title → that app's rule file, (c) phrase-keyword
   match to file domains, (d) sibling verb ("make X" lives with other "make" commands). Surface top 3.

**Auto-refresh of the catalog — EVENT-DRIVEN, not a poller (BUILT 2026-06-22, runtime-pending).**
Regenerates *because a command file changed*, riding Caster's existing hot-reload — no second timer
running in steady state. Chain: `programming_general_commands.py` is the **trigger file** — on (re)load
it calls `grammar_dump.schedule_dump()`, which arms ONE one-shot Dragonfly timer (`repeating=False`,
~2s, coalesced) that dumps once. The `caster_rule_postwrite` hook bumps the trigger file's reload marker
on *any* rule edit, so editing a command → Caster hot-reloads the trigger → one dump ~2s later (delay
lets same-cycle reloads of the other changed files settle first). `add_voice_choice` will bump it too
(wired with the wizard) for JSON-store adds. Reboot-free. Coverage: Claude edits (via the hook) now;
wizard/JSON adds (via add_voice_choice) next; pure hand-edits fall back to manual `make catalog`.
Verified: `_bump_trigger` bumps the marker + skips self; the one-shot path runs in the Dragon process.
Rejected the polling variant (Jamie: no background poller). **"make catalog" relocated** to
`programming_general_commands.py` (sibling to "make dump"); `grammar_dump_commands.py` gutted + dropped
from `rules.toml` (delete file next reboot).

**Helper-cache caveat:** Caster hot-reloads rule files but NOT cached helper modules, so changes to
`grammar_dump.py` (schedule_dump, the version fix) only take effect on the next full reboot. The trigger
import is defensive (`try/except`) so a live edit never breaks the rule — auto-refresh simply begins
working after the next reboot. Until then, `make catalog` is the manual refresh.

---

## 13. Build phases (sequencing)

1. **Fuzzy search** — token-subset + word-prefix in `PersistentLoopGui` (replaces `InStr` at `:545`/`:704`)
   + **session-numbering fix**. Independent, ship first, fixes today's pain across *every* picker.
   **✅ DONE 2026-06-22:** `_PLG_AllTokensMatch` added; wired into the left pane, secondary pane, and
   group auto-expand. No reordering → stable numbering untouched (session-numbering only needed once
   score-ranking lands, still deferred). Verified: "toggle monitor" now surfaces "toggle debug monitor".
2. **Catalog-layer rebuild (the trunk)** — live-grammar export → provenance-stamped, pre-sorted JSON +
   incremental `search_terms` enrichment + coverage-gap report. Kills slowness *and* incompleteness together.
   **⚡ Slowness half DONE 2026-06-22 (separately):** measurement showed the ~couple-second open was
   *entirely* an O(n²) insertion sort in `_VCB_LoadAllVoiceCommands` (sort=3656ms of 3688ms total over
   2,723 commands). Replaced with a stable O(n log n) merge sort (`_VCB_MergeSort` in
   `VoiceCommandBrowserFunctions.ahk`) → **sort 3656ms→78ms, total build 3688ms→125ms (~30×)**. This was
   a contained ~40-line AHK change, *not* the Python rebuild — speed never justified the rebuild; only
   **completeness** does. **Remaining (the real Phase 2):** live-grammar export + provenance + pre-built
   JSON + enrichment, to fix the invisible-JSON-commands problem (§2, §5). The merge sort is a temporary
   bridge that disappears once sorting moves to Python.
3. **`voice_index`** (`search` / `check` / `coverage`) + the protocol Claude follows (check before & after
   every create). Depends on phase 2.
   **✅ search + check DONE 2026-06-22:** `~/.claude/helpers/voice_index.py` reads the live dump directly.
   `check` compiles each Dragonfly spec → regex and reports whether a command already fires on the
   candidate (exact / Choice-key / dictation-shadow / optional / alternation), context-filtered
   (global overlaps all; disjoint exe/title don't); verdict COLLISION/WARN/CLEAR + lint (verb-first,
   hard-/k/ onset). Tested vs the real dump: `make catalog`→self, `spot Charlie`→`spot <spot_slot>`+`spot <textnv>`,
   `jump line 5`→vscode line cmd only in exe=code (drops in chrome), number templates no longer false-match
   word phrases. `search` = token-subset over phrase+rule+context+Choice keys. Protocol wired into the
   caster-voice SKILL.md ("Collision check — MANDATORY before creating any voice command").
   **Remaining:** `coverage` (needs the usage-feed join, deferred); incremental `search_terms` enrichment.
4. **Norms** (`SKILL.md` block + `references/phrase-design.md`, feeds the lint) + **refactor-audit tool**
   (needs catalog + norms). First refactor: `start stream`.
   **✅ DONE 2026-06-22 (Big Build 1):** `references/phrase-design.md` written (verb lexicon + sound rules
   + verb+value lint formulation); SKILL.md phrase-naming section rewritten to match (verb-first/2-word/soft
   onset, `add`/`start`, domain-first only at scale). **Refactor-audit** = `~/.claude/helpers/voice_audit.py`:
   AST-scans rule files → **HARDCODED LOGIC** list (commands with >2 inline Key/Playback steps that should be
   AHK fns — found **101**, ranked worst-first w/ file:line + snippet; mainfun/Function dispatches never
   flagged) + **CONVENTION** list (low-noise: one-word commands + hard-/k/ onsets — **126** flags; debug
   entries skipped). UTF-8 stdout. **Remaining Build-1 follow-ons (lighter):** `voice_index coverage`
   (catalog `action` ↔ `ahk_event.jsonl` usage join — catalog carries `action`, feasible) + full dump→catalog
   join (dump-only commands get desc/tags). `start stream` refactor still pending.
5. **Generic command store + materializer** + **function-first wizard** (built on the two interfaces;
   data-append first, code-gen only for novel structure). Last and hardest.
   **✅ STORE + GLOBAL MATERIALIZER DONE 2026-06-22 (Big Build 2, safe half) — awaiting ONE reboot to go live.**
   - `rules/generic_command_store.py` — typed JSON store (`INIDATA/VoiceChoices/generic_commands.json`,
     co-located with the other 41 stores). `validate_row` / `load_rows` (safe-fail: bad rows collected,
     never raised) / `add_row` / `remove_phrase` / `bump_materializer_marker`. The **arg_kinds→bridge
     map is solved once here** (`[]`→`mainfun_action` or fixed-args `run_mainfun_args`; `["text"]`→
     `named_mainfun_action_with_args`; `["text_snake"]`→ +`to_snake_case`; `["number"]`→
     `named_mainfun_action_with_number`; `["text","text"]`→`…with_two_extras`). Stdlib-only (import-safe
     in the Natlink process) + CLI (`list|add|validate|remove`). **Tested:** valid no-arg/text/number rows
     accepted; slot↔arg_kinds mismatch, bad function name, and fixed-args-with-slots all rejected; marker
     bump 0→3 verified; store reset to `[]`.
   - `rules/generic_materializer_commands.py` — **global** `MergeRule` (`GenericMaterializerRule`, registered
     in `rules.toml` _enabled_ordered + whitelisted). Reads the store's global slice at load, builds one
     spec per row via `_build_action`, **safe-fails per row** (skips + logs to
     `.claude/context/generic_materializer.log`). Debug entry first; extras pool `text/text2/number`.
     Compiles clean on Py310-32 syntax. Context-kind rows skipped (per-context materializers = later build).
   - **No-reboot adds:** appending a row bumps this file's `# reload-marker`, so Caster's TimerReloadObservable
     hot-reloads it and re-reads the store (~5s, no reboot). First registration still needs ONE reboot (new file).
   - **Auto-flows into the catalog:** once live, materialized specs are real registered grammars, so the
     live-grammar dump captures them with no extra wiring. Catalog generator unaffected (it only globs
     `hardcoded_*_slots.json` + its allowlist; the list-shaped store is ignored).
   - **LIVE-CASTER CHECKPOINT (Jamie controls the reboot):** can't runtime-test grammar registration without
     it. After the next reboot: say **"Generic Materializer Commands"** → expect "Generic Materializer Rules
     Are Working". Then `generic_command_store.py add …` a real command and confirm it goes live without a
     second reboot. **Build 3 (function-first wizard) builds on this** — it only ever writes rows + bumps.
   - **✅ LIVE + PROVEN 2026-06-22 (2nd reboot):** first reboot rejected the rule — it was written as a
     `MergeRule` with no `ccrtype` (Caster: *"MergeRules must have a ccrtype"*); fixed to `MappingRule`
     (every non-CCR rule here is a MappingRule; the enable/disable trigger derives from `RuleDetails.name`).
     The combo error only fires at Dragon load (rejected rules are never put under file-watch —
     `grammar_manager.py:118` early-returns before `register_watched_file`), so it needed a 2nd reboot.
     **Hardened CI:** `validate_rule.py` now statically flags `MergeRule`-without-`ccrtype` (and the inverse)
     so this can't recur. After the fix the materializer loaded + the debug entry confirmed. **CORRECTION
     (2026-06-22): the "make mouse spot went live" claim here was NOT voice-confirmed at the time** — and a
     latent bug meant the no-reboot path silently broke the moment the store's *code* changed (see the
     cached-helper fix in Build 3b below). The marker-bump→hot-reload path is now genuinely proven by voice
     (Jamie confirmed "make mouse spot" + "make alarm morning" both fire) only AFTER that fix.

   **✅ BUILD 3a DONE 2026-06-22 — wizard backend (`~/.claude/helpers/voice_wizard.py`).** The Interface-1
   assign-action (sec 11), testable + CLI:
   - `assign(function, phrase, arg_kinds, scope, exe, title, tags, force, dry_run)` → validate row shape
     (`generic_command_store.validate_row`) → **collision + lint** (`voice_index.check` over the live dump)
     → write row + bump marker. Stops on COLLISION unless `--force`; WARN/CLEAR write through. Returns a
     structured dict (`ok/stage/verdict/live/row`) so a front-end can show the collision report + lint notes
     before confirming. Meaningful exit codes (0 ok / 1 invalid / 2 collision).
   - `suggest_phrase(fn)` — naive verb-first phrase from a PascalCase/camel/snake name (`CopyMousePosRelative`
     → "make mouse pos", `OpenScratchpad` → "open scratchpad"). A *starting point* the human/LLM refines —
     the smart suggestion is Claude's job, never authoritative here.
   - **Tested:** suggest across 4 names; dry-run COLLISION (blocks "mouse relative"); WARN (siblings, hard-/k/
     onset); invalid-row rejection. Full verify suite PASS.
   - **The Claude-driven front-end works NOW:** "bind X to function Y" → I run `voice_wizard assign` (mandatory
     check built in) → live in ~5s, no reboot.
   - **✅ SUGGESTION BRAIN DONE 2026-06-22 (Jamie: "I want some sort of brain behind it… several options… never
     auto-accept").** `voice_wizard.py options <Function> [--purpose …]` → `suggest_phrases()`: calls the local
     LLM (`local_llm` task **`voice_suggest`**, norms baked into the system prompt — verb lexicon + soft-onset +
     2-word) for ~5 candidates, **collision-filters** them (drops any COLLISION — an unusable suggestion isn't a
     suggestion) and **norm-ranks** (CLEAR before WARN, fewer lint notes, known-verb-led, shorter). Naive split
     always included as an offline fallback. Tested: `ToggleDebugMonitor` → "make debug visible / toggle debug
     view / make debug monitor / show debug monitor / enable debug info". **Behavioral rule encoded in SKILL.md:**
     never auto-accept — always run `options`, sanity-check, consider better, present Jamie several; she picks/edits.
   - **✅ SLOT COMMANDS + SIGNATURE-AWARENESS DONE 2026-06-22 (Jamie: "aware of commands like `text <blank>` /
     `name chat <blank>` … name variables like that … import the variable from AutoHotkey appropriately").**
     - **Free-form named slots** (store + materializer refactor): a phrase's slot name is the AHK *variable*
       (`text <message>`, `wait <secs>`, `move <from> to <to>`) — `generic_command_store.slot_plan` pairs each
       name with its kind (positional from `arg_kinds`); the materializer builds a **union extras list keyed by
       slot name** (`Dictation(name)` / `IntegerRef(name)`) so the spoken value imports into the AHK call under
       that name. `validate_row` enforces slot-count == arg_kinds and unique lowercase-identifier names.
     - **AHK signature detection** (`voice_wizard.function_signature`): finds `Fn(params)` in the Helpers source,
       parses params (byref/variadic/optional/defaults), infers kind (numeric-ish name → `number`, else `text`),
       labels the slot from the param. Tested: `AlarmClockAddAlarm(hour,minute)`→2×number, `AddPreset(name)`→text,
       `CopyMousePosRelative()`→none.
     - **Brain is slot-aware:** a function that takes an arg yields slot-phrase candidates (`make <name>`,
       `start <name>`), verb from the LLM + slot from the param; no-arg functions get plain phrases. `assign`
       **auto-detects `arg_kinds` from the signature** when the phrase has slots (no `--arg-kinds` needed).
     - **Collision handles slots:** proposing `text <message>` is correctly BLOCKED by the existing `text <textnv>`
       (Google Voice) — the system is aware of the existing `text <blank>` family. Store/materializer compile +
       full verify PASS.
   - **✅ NAMESPACE ECONOMY / SLOT FOOTPRINT DONE 2026-06-22 (Jamie: "make <blank> is infinitely more prime than
     make alarm … it needs to differentiate … feed the LLM better examples").** Frequency-from-usage-log was
     considered and REJECTED as the driver (a brand-new command has no history — guessing). The real fix is a
     **measurable footprint + a specific-default**, no frequency dependence:
     - `voice_index.slot_footprint(phrase, commands)` — a wildcard's cost = how few/common the FIXED words before
       it are. `make <x>` (1 fixed token, verb family=20) = **greedy**; `make alarm <x>` (2 fixed) = confined.
       Greedy = lone fixed verb that is **busy** (`verb_family_size` ≥ `GREEDY_FAMILY_THRESHOLD`=3, corpus-measured)
       **OR** an intrinsically-prime **core lexicon verb** (`CORE_PRIME_VERBS` open/see/make/show/add/start/send/
       save/hide/find — so `start <x>` is greedy at family 0). A rare/dedicated verb (`frobnicate <x>`) is exempt.
     - **Not a rejection — an informational WARN + ranking penalty.** `check()` adds a greedy lint note + returns
       `footprint`; the wizard generates **confined candidates by default** (`_verb_and_nouns` → `add alarm <name>`
       / `add preset <name>` from the function name) and ranks confined above greedy (rank key:
       verdict > greedy > lint-count > known-verb > word-count — word-count is a LATE tiebreak so a clean
       namespaced 3-word beats a 2-word unknown-verb one). Greedy prime phrases are still offered (deliberate
       opt-in for a frequent command) but sink. Tested: `AlarmClockAddPreset` now leads with `pick <name>` /
       `pick alarm <name>` / `add alarm <name>`, with `make <name>`/`start <name>` flagged greedy + sunk.
     - **Examples baked in both places** (Jamie's ask): LLM `voice_suggest` prompt got a namespace-economy block +
       good/bad (`add alarm <name>` GOOD vs `make <name>` BAD); `phrase-design.md` got a "Namespace economy & slot
       footprint" section + good/bad table; SKILL.md got the principle + a hardened "LLM is raw material, Claude
       is the brain — don't accept a terrible phrase the little model produced; ask about frequency when it decides."
   - **⏳ BUILD 3b IN PROGRESS 2026-06-22 — the button/GUI surface (Jamie wants it on BOTH ShowRecentFunctions +
     the macro wizard).** Architecture: a **standalone detached runner** any surface spawns with a function name
     (same pattern as ShowRecentFunctionsViewer) so the modal picker survives the caller's `ExitApp` and gets the
     picker's includes (`PersistentLoopGui` has none of its own). Built + **`ahk.py validate` clean** (includes
     resolve, no dup definitions):
     - **Python JSON layer** — `voice_wizard.py options/assign --json` (tested): options emits
       `{function,found,params,arg_kinds,candidates:[{phrase,verdict,lint,greedy,arg_kinds}]}`; assign emits
       `{ok,stage,live,verdict,collisions,lint}` with meaningful exit codes.
     - **`Helpers\Gui\VoiceWizardAssign.ahk`** — `AssignVoiceToFunction(fnName)`: calls `options --json` →
       `_PersistentLoopPickGui` (numpad pick, `allowUnknown` = pick-a-number-OR-type-your-own, slot-aware) →
       `assign --json` → tooltip/MsgBox outcome. `PyVoiceWizard(args*)` = temp-file stdout capture (the
       `_GuiLayoutRun` pattern). Deps must be host-supplied (no self-`#Include`).
     - **`Scripts\VoiceWizardRunner.ahk`** — standalone entry (`#Include`s the picker+JSON deps + the orchestrator,
       mirrors the viewer's proven set), reads `A_Args[1]` = function name, calls `AssignVoiceToFunction`. **This
       is what surfaces spawn:** `Run(Format('"{}" "{}" "{}"', A_AhkPath, runnerPath, fnName))`.
     - **✅ RUNNER PROVEN END-TO-END BY VOICE 2026-06-22.** Jamie ran the runner for `AlarmClockAddPreset`, the
       picker rendered (numpad list, slot-aware), she picked the confined **`make alarm <name>`** → `assign`
       auto-detected `arg_kinds=["text"]` from the signature → wrote the store row → materialized → **she said
       "make alarm morning" and it fired `AlarmClockAddPreset("morning")`.** The full pipeline works: surface →
       brain → picker → pick → live voice command, zero code, no reboot.
     - **BUG FOUND + FIXED during the live test — the cached-helper trap (latent hole in the whole no-reboot
       promise).** First voice test failed ("not recognized"); the Caster log showed the materializer reloading
       but registering NOTHING. Cause: Caster hot-reloads rule files but NOT imported helper modules, so after the
       store's *code* changed (gained `slot_plan`, new `validate_row`), the materializer kept importing against the
       STALE in-process store; the missing `slot_plan` raised ImportError and the defensive `except` silently
       emptied the rule. Fix: `import generic_command_store as _gcs; importlib.reload(_gcs)` on every materializer
       load → store CODE changes now go live without a reboot, not just store DATA. Confirmed via a fresh grammar
       dump containing both commands. Documented in SKILL.md ("Cached-helper trap"). The READ-THE-LOGS habit caught
       this cold — implementation guesses would have missed it.
     - **Cleanup:** `→`/`✓`/`…` in `VoiceWizardAssign.ahk` → ASCII (they crashed `ahk.py check`'s cp1252 console
       printer; the AHK GUI renders unicode fine — this was a tooling-side bug, noted as a separate `ahk.py` issue).
     - **✅ MACRO WIZARD WIRED 2026-06-22.** `SetMacroWizardFunctions.ahk`: new numpad action **`13` = assign
       voice** (`_Smw_AssignVoiceFromPicker`) in the wizard picker's `numpad_dispatch`. Pick a function row, run
       action 13 (e.g. `3.13`) → spawns `VoiceWizardRunner.ahk` for that function (detached, survives the wizard's
       ExitApp) → the phrase picker pops → assign. The pressed key is NOT bound; a voice phrase is. Footer hint +
       handler added. `ahk.py validate MAINFUNCTIONS.ahk` clean (exit 0) + full suite PASS.
     - **✅ ShowRecentFunctions viewer WIRED 2026-06-22** (Jamie clarified: wire it, just skip the `v` *letter*
       hotkey — a letter would collide with the planned type-to-search). Added **numpad action `20` = assign
       voice** to `ShowRecentFunctionsViewer.ahk`'s `BuildListActions()` (2-digit so it can't collide with a row
       number; the BrowserGui dispatch checks the action map before row-jump, "actions win on collision").
       `AssignVoiceAction(ctl)` reads `ctl.FocusedRow()` → `Entries[row]["fn"]` → spawns the runner. Focus a
       function row, type `20`+Enter. The viewer is a freshly-spawned process each "show fun", so this is live
       immediately — no reload. `ahk.py validate` clean + full suite PASS. (The future type-to-search rework can
       replace/augment the trigger; the runner stays the shared entry.)
     - Demo commands (`make mouse spot`, `make alarm <name>`) removed; store back to `[]`.
     - **Build 3b effectively COMPLETE** for the in-scope surface (wizard). Viewer integration rides the future
       search rework. The whole §11 vision is now real: function-first, brain-suggested, footprint-aware,
       slot-aware, collision-checked, live-no-reboot voice-command creation.

**Deferred to post-v1:** `bindings.json` ingest, subsequence search tier, phonetic collision lint, full
trigger-unification with the macro wizard. (Design the shapes now; don't build yet.)

---

## 14. Key file map (for whoever builds this)

| Concern | Path |
|---|---|
| In-GUI fuzzy filter | `Helpers/Gui/PersistentLoopGui.ahk` (`:545`, `:704`) |
| Browser entry points | `Helpers/VoiceCommandBrowserFunctions.ahk`; rule `rules/voice_command_browser_commands.py` |
| Catalog generator (to rebuild) | `Scripts/codebase_tools/voice_commands_by_tag.py` |
| Catalog outputs | `C:\Users\jamie\.claude\context\voice_commands_by_tag.{md,tsv,json}` |
| Dispatch feed / usage | `ahk_event.jsonl`; `Scripts/codebase_tools/macro_setup_usage_audit.py` |
| JSON command stores | `INIDATA/VoiceChoices/*.json` (41); writer `Scripts/VoiceConfigManager/add_voice_choice.py` |
| Recent-functions picker | `ShowRecentFunctions()` (`rules/macro_commands.py` "show fun"); `Scripts/ShowRecentFunctionsViewer.ahk` |
| Macro wizard | `Helpers/SetMacroWizardFunctions.ahk` → `INIDATA/bindings.json`; `Helpers/BindingResolver.ahk` |
| Context scoping | `RuleDetails(executable=, title=)`; `rules/window_context_helpers.py` (`function_context`) |
| Local LLM gateway | `Scripts/local_llm/local_llm.py` + `tasks.json` |
| rdescript postwrite hook | `C:\Users\jamie\.claude\hooks\caster_rule_postwrite.py` |
| Rule enablement | `C:\Users\jamie\AppData\Local\caster\settings\rules.toml` (`_enabled_ordered`) |
| Skills | `~/.claude/skills/caster-voice/`, `~/.claude/skills/ahk-functions/` |

---

## 15. The Voice Command Editor (Miller-columns rework of "show fun")

**Status: PHASES 1/2/3/4 BUILT (2026-06-22); scope step blocked on per-context materializer.** A full
**function-first voice-command editor**. Jamie's framing: "adding a new command is just one facet —
I want a full editor." Decisions locked: **live commands only** (not meta-aliases) [1],
**replace "show fun"** [4], **voice-only right column for now** [7] but **architected so key/Stream-Deck
bindings can join later** [8, deferred], **design-doc-first** [9].

**As built (2026-06-22):**
- **Attribution coverage fix** ✅ (Jamie 2026-06-22, "a lot of these have voice commands but aren't picked
  up"). Two parts: (1) **The bracket bug** — rule rdescripts wrap the phrase in `[brackets]`
  (`[lockout] -> StartLockoutTimer()`) but the live dump does NOT (`lockout`); `_norm` was keeping the
  brackets, so EVERY bracketed rule command failed the dump join. Stripping a surrounding bracket pair
  in `_norm` took the index from **23 functions / 29 commands → 235 / 298** (and the displayed phrase now
  uses the clean dump form, not the bracketed rdescript). (2) **Manual override map** —
  `INIDATA/voice_function_overrides.json` (`{function: [phrases]}`), a TRUSTED hand-maintained map for
  commands no detector can attribute (runtime Choice dispatch like `open <site>` fanning out to a
  dedicated function, program/link-registry launches, dispatcher-named rdescripts). The join merges them
  as `tier:"manual"` read-only rows; the editor's info tooltip points back to the file. First entry:
  `OpenGoogleVoiceWithWait → "open google voice"`. **The remaining ~320 unresolved are genuinely
  function-less** (inline `Key`/`Text`, alternation phrases) or runtime-Choice multi-dispatch — add an
  override line for any real one that surfaces. (A future detector could parse the program/directory/site
  registries to auto-attribute the `open <X>` family.)
- **Phase 1 — join index** ✅ `Scripts/codebase_tools/voice_by_function.py` builds
  `context/voice_by_function.json` = `{functions:{FN:{tags,commands:[{phrase,context,source,tier,editable,live,file,line}]}}, unresolved:[…]}`.
  Joins the live grammar dump (liveness + context) × the tag catalog's `action` (function attribution)
  × the generic store (function direct). Iterates JAMIE'S commands (store + rdescripts), dump as a
  liveness gate, so `unresolved` stays her commands (inline-Key / multi-dispatch), not the ~1400
  framework built-ins. Regenerated on the Stop hook (called from `voice_commands_by_tag.main`) and at
  editor boot. CLI: `--function FN` (per-function query), `--stdout`.
- **Phase 2 — editor GUI** ✅ `Helpers/VoiceCommandEditorMenu.ahk` (`OpenVoiceCommandEditor`), a
  Miller via the scaffolder, GuiHost-hosted (survives later MAINFUN dispatches). Left = recents ∪
  functions-with-commands; right = `print` (paste block) / each live command READ-ONLY (phrase +
  `[scope]` + tier) / `+ add voice command` (spawns `VoiceWizardRunner`). Shared dispatch-read + paste
  block extracted to `Helpers/RecentFunctionsData.ahk` (`Rfd*`). Screenshot-verified.
- **Phase 4 — cutover** ✅ `macro_commands.py`: **"show fun" → OpenVoiceCommandEditor**;
  **"show function" → ShowRecentFunctions** (old viewer preserved, unchanged). Hot-reloads, no reboot.

- **Phase 3 — per-command edit** ✅ STORE-tier rows are **drillable branches** whose children are
  VISIBLE action rows — **Remove** / **Rephrase** (arrowing a command previews them on the right;
  drilling picks them). **Hard rule (Jamie 2026-06-22): never expose an action ONLY via an `N.M`
  compound — if it can be done, it shows as a row.** Remove: `_ConfirmationModalGui` →
  `generic_command_store remove` (live, no reboot) → reopen the editor drilled at the same function
  (`OpenVoiceCommandEditor([fn])` + `initial_path`). Rephrase: **non-destructive** — runs the add
  wizard carrying the old phrase; the wizard drops the old row ONLY after the new one is assigned
  (`AssignVoiceToFunction(fn, replacePhrase)`), so cancelling keeps the original. Rule/choice rows stay
  read-only leaves (Enter shows source + how to change).
- **Add/rephrase run INLINE in the editor host** (not the detached `VoiceWizardRunner`), so **Esc from
  the wizard returns to the editor** (Jamie 2026-06-22) — on exit (success or cancel) the editor reopens
  drilled at the function. `VoiceWizardAssign.ahk` is now in the MAINFUNCTIONS closure for this. The
  detached runner still serves the old viewer's `20` + the macro wizard's `13` (they ExitApp).
- **Collision UX fix** ✅ (Jamie 2026-06-22): the add/rephrase wizard no longer dumps an error box and
  quits on a collision/lint-reject — `AssignVoiceToFunction` now LOOPS, re-opening the same picker with
  the reason shown atop it (`_VWAssignFailNote`), so a collision reads as "pick another." Only success
  or an explicit cancel leaves.

**Still pending:** the **scope step (global/context) on add** is blocked on the **per-context
materializer** (a context-kind store row can't go live until that exists — building the scope UI now
would just store dead commands). That materializer is the next real subsystem (§15.7 phase 5).
**Dedup debt:** the old viewer (`ShowRecentFunctionsViewer.ahk`) still has its own copies of the
dispatch-read/paste-block logic now living in `RecentFunctionsData.ahk` — refactor it to use the shared
module when next touching it.

### 15.1 UI — Miller columns

```
┌─ filter: [type to search functions / phrases / descriptions] ─────────────┐
│  LEFT: functions                    │  RIGHT: options for selected fn       │
│  (last 25 dispatched, then search)  │                                       │
│  > CapitalizeFirstLine              │   1  print   (paste block — today's   │
│    OpenScratchpad                   │              Enter behavior)          │
│    SwitchTopDesktop                  │   ── live voice commands ──           │
│    …                                │   2  "capitalize first line"  [global]│
│                                     │   3  "make caps"      [global] (store)│
│                                     │   4  "make capital"   [global] (store)│
│                                     │   ── add ──                           │
│                                     │   5  add voice command (brain → pick) │
└─────────────────────────────────────┴───────────────────────────────────────┘
```

- **Left = functions.** Default: the **last 25 dispatched** (from `ahk_event.jsonl`, deduped — extends the
  current 15). Typing filters/searches (see 15.4). One row per function.
- **Right = that function's options.** Row 1 is always **`print`** = today's `PickAndPaste` (clipboard + paste
  the function block). Below: **the function's live voice commands** (view/edit/remove), then **add**.
- Numpad-first per `feedback_numpad_ui_design.md`: left rows pick a function; right rows are numbered actions.
  Likely `MillerColumnPickGui.ahk` (already exists) or an extended `BrowserGui`.

### 15.2 Data model — function → live commands (the core challenge)

The dump is authoritative for *what's live* (`phrase, rule, exe, title, context_kind, slots`) but **carries no
function**. The catalog (`voice_commands_by_tag.json`) carries **`action`** (the function) + `phrase` +
`file/line` + `source` + `description`, parsed from rule files. The **generic store** carries `function` per row.
So the editor's index is a **join**:

1. **Live set + context** ← the grammar dump (complete; includes JSON-store Choice commands).
2. **Function attribution** ← join each dump phrase to the catalog's `action` by normalized phrase (+context).
   Generic-store commands get their function directly from the store.
3. **Unresolved bucket** ← any live phrase whose function can't be attributed is grouped visibly under
   "(unresolved)", never dropped — the coverage-gap rule from §5.
4. **Editability tier** per command:
   - **store** — generic-store row → fully editable (rephrase / rescope / remove) via `generic_command_store`.
   - **rule** — hardcoded in a `*_commands.py` → shown read-only with file:line; edit options in 15.3.
   - **choice** — JSON-store Choice value (sites, shows…) → edited via `add_voice_choice.py`, linked out.

Built by extending the catalog generator (`voice_commands_by_tag.py`) to emit a **`by_function` index**
(`{function: {meta, commands:[{phrase,context,source,tier,editable}]}}`) consumed by the AHK editor. This is
the §6 "coverage"/join work, now concrete.

### 15.3 Right-column operations (voice-only for now)

- **print** — paste the function block (unchanged behavior; preserves muscle memory through the rename).
- **add voice command** — the proven wizard flow (`voice_wizard options` → footprint/slot-aware picker → `assign`)
  **+ a scope step** (global / context — the current gap Jamie hit; see 15.5).
- **per live command** — Jamie chose BOTH [3]+[4], tabled [5]:
  - **store tier** — fully editable inline: **rephrase** (collision-checked rewrite) / **rescope** / **remove**
    the row, live.
  - **rule/choice tier [3]** — shown **read-only** with its source (`file:line` / which JSON store); to change
    behavior you **add a new store command** for the function rather than touch the hardcoded one.
  - **rule/choice tier [4]** — **guided edit**: a "open at source" action jumps to the `*_commands.py` line (or
    `add_voice_choice --remove`/`--replace-phrase` for choice stores) for a manual change.
  - **(future [8])** key binding / Stream Deck button for this function, unified with the macro wizard.
- **migrate-to-store [5] — TABLED** (Jamie: "interested eventually"). Would re-create a `rule`-tier command as
  an editable `store` row + remove the original. Revisit later.

### 15.4 Search [BUILT — searches EVERY function, Jamie 2026-06-22]

Type-to-search over the left column, matching **function name + its live phrases**. **The left list is
ALL ~1017 dispatchable AHK functions** (recents pinned at the top, then alphabetical), not just
recents/ones-with-commands — Jamie: "I want to type and pull up anything, any voice command or any AHK
function." Built by seeding `VceFnOrder` from `INIDATA/fn_to_file_index.json` (Stop-hook regenerated;
leading-underscore helpers skipped). Each function node carries `search_skip:true` + its own `search`
text (name + command phrases), so the Miller recursive search matches across the whole set instantly
without live-walking each function's options. A command phrase is found via its function's search text;
Enter drills to that function's options. Open question 6 (meta.json aliases in the search text) still
open — would add capability-recall ("uppercase" → `CapitalizeFirstLine`); deferred.

### 15.4b Wizard resilience [BUILT — Jamie 2026-06-22]

The add/rephrase wizard is now crash-proof and never dead-ends:
- **Decode-crash fix:** `voice_wizard._llm_candidates` forced `encoding="utf-8", errors="replace"` —
  the default cp1252 decode crashed subprocess's reader THREAD on a non-decodable byte from the model,
  and that thread's traceback leaked into the captured output, breaking the JSON the picker parses
  (Jamie hit this on a rephrase: "No phrase candidates for SwitchTopDesktop" + a Python traceback).
  The `options` CLI path is also fully `try`-wrapped so it ALWAYS emits valid JSON (empty candidates on
  any failure), never a traceback.
- **Type-your-own fallback:** when `options` returns zero candidates (brain offline / all collided), the
  picker opens anyway with an empty list + a note, so Jamie types her own phrase (the assign step
  re-detects the signature server-side). No more error-box-and-quit.

### 15.5 Scope step (global / context) — fixes the gap Jamie hit

The current wizard always writes `global`. The editor's add/edit flow gets an explicit scope choice:
**global** vs **context** (exe/title, auto-suggested from the function's usage context in `ahk_event.jsonl` —
the `win.exe`/`title` recorded at dispatch). Requires the **per-context materializer** (deferred build) for a
context row to actually go live; until then a context choice is stored + flagged "not live yet".

### 15.6 Migration — coexist via rename (NOT a hard replace) [Jamie 2026-06-22]

Don't retire the current viewer. **Rename its trigger to "show function"** (today's ShowRecentFunctions viewer
stays fully available during + after the build), and point **"show fun" at the NEW editor**. So: `"show function"`
→ existing `ShowRecentFunctionsViewer.ahk`; `"show fun"` → the editor. `PickAndPaste` is preserved as the editor's
right-column `print` action. The numpad-`20` assign-voice hook folds into the editor's right column.
`VoiceWizardRunner.ahk` stays the shared assign entry; macro-wizard `13` unaffected.

### 15.7 Build phases

1. **`by_function` index** — extend `voice_commands_by_tag.py` to emit the join (15.2). Testable headless.
2. **Editor GUI** — Miller columns: left fn list (last 25 + search), right options. `print` first. Read-only
   command list to start.
3. **Add/edit/remove wired** — right-column actions call `voice_wizard`/`generic_command_store`; scope step.
4. **Replace "show fun"** + fold the `20` hook in.
5. **(later)** per-context materializer (unblocks context scope going live); bindings/SD column [8].

### 15.8 Open decisions for Jamie

Carried into the review round below (search-meta-aliases [6]; edit-hardcoded-commands tier; migrate-to-store).

---

## 16. The Registry Editor — unified, auto-discovering ("open registry")

**Status: DESIGN (2026-06-22), agreed before building.** Voice: **"open registry"**. A single Miller that
browses **every registry** and lets Jamie edit **individual entries** (a site, a contact, a context, a
send-prompt) line-by-line — instead of hand-editing JSON or remembering which per-registry wizard to run.
Mounts **standalone** AND as a **branch in show fun** (and a future master menu, via the `AsNode` hook the
editor already has). Jamie's framing: *"a bunch of our registries editable within one thing… build the
system unified."*

### 16.0 North star — auto-discovery, zero hardcoding, maximally searchable

Jamie's hard requirement: **"I don't want to hardcode choice maps and stuff — as automatically generative
as possible and as searchable as possible."** So the editor is **driven by the files on disk**, not by a
hand-maintained list. The current hardcoded `EXPLICIT_CHOICE_MAP` (≈30 entries in `voice_commands_by_tag.py`)
is the **anti-pattern this replaces** — the registry layer must *derive* what that map encodes (file ↔ slot
↔ phrase ↔ tags) automatically, and ideally `voice_commands_by_tag` consumes the same derivation so the
hardcoded map can eventually be retired (one source of truth for "what registries exist").

### 16.1 Registry discovery (no hardcoded list)

- **VoiceChoices family (~42):** glob `INIDATA/VoiceChoices/*.json`. Each file = one registry. Display name
  derived from the filename (`sites.json` → "Sites", `gv_contacts.json` → "GV Contacts").
- **Contexts family (~55):** the `INIDATA/Contexts/` **folder** = ONE registry whose entries are the files
  (one context per file). Plus `directories.json` / programs.
- New registry = drop a JSON file; it appears automatically. No code, no map edit.

### 16.2 Schema inference (the heterogeneity problem — the core challenge)

Registries are NOT uniform; the editor infers each one's shape by **sampling its entries** (confirmed shapes
2026-06-22):

| Shape | Examples | "An entry" is | Edit UI |
|---|---|---|---|
| `dict[str → str]` | send_prompts (phrase→text), gv_contacts (name→phone) | a key→string pair | SingleField (the value) |
| `dict[str → object]` | sites/google_docs (→{title,url}), directories (→{path,program}) | a key→{fields} | Form, fields **inferred from the entry's keys** |
| `dict[str → list]` | gv_broadcasts (name→[members]) | a key→list | list editor (add/remove items) |
| **one-object-per-file** | `Contexts/*.json` (each file = {exe,title,…}) | a **file** | Form over the file's fields |

Two orthogonal axes the model must carry: **entry granularity** (keys-in-a-file vs files-in-a-folder) and
**value shape** (str / object / list). Add/edit/remove are all driven off the inferred shape — a brand-new
registry of a known shape works with zero per-registry code. Genuinely novel shapes fall back to a raw-JSON
edit (honest escape hatch) rather than silently mis-rendering.

### 16.3 Auto-derived voice linkage + show-fun attribution (kills the hardcoded map)

The live grammar dump already records, per phrase, each **choice slot and its key set** (`slots: {site:
{kind:"choice", keys:[…]}}`). Matching a registry file's **key set** against the dump's slot key sets
**auto-derives** which phrase template a registry backs (`sites.json` ↔ `open <site>`) — no
`EXPLICIT_CHOICE_MAP`. Wins:
- The editor shows "powers: `open <site>`" per registry, derived.
- It **feeds show-fun attribution**: a registry entry whose dispatch resolves to a function attributes that
  command automatically — the proper, general version of the manual `voice_function_overrides.json`
  (which stays as the escape hatch for the truly underivable).
- `voice_commands_by_tag` can consume the same derivation → **retire `EXPLICIT_CHOICE_MAP`** (one source of
  truth). Best-effort: a registry with no dump match still edits fine; it just shows no linkage.

### 16.4 Searchability

A flat **`search_index`** (the Miller pattern, built once + cached) over **every entry of every registry**,
so typing finds any line instantly across thousands of entries — `"google voice"` surfaces the sites entry,
`"commit"` the send-prompt. Token-subset matching (already engine-wide) means word order doesn't matter.
Enter on a hit drills to that entry's edit actions; the registry list itself is also searchable.

### 16.5 Edit / add / remove / reload (uniform, shape-driven)

- **Edit / Add / Remove** are **visible drill rows** per entry (Jamie's hard rule: no action reachable only
  via `N.M`). Shape-driven editors (16.2). Reuse `add_voice_choice.py` / `set_json_key.py` where they fit;
  one uniform read/write API over both families otherwise.
- **Reload trigger:** a write must make the command live without a manual reboot. Different registries reload
  differently (VoiceChoices touch their backing rule so Caster re-reads; Contexts are read live by AHK). The
  layer derives the rule(s) to touch from the dump linkage (16.3), falling back to the known per-family
  behavior. **Write safety:** atomic temp-write + replace, and a backup before destructive edits (these are
  Jamie's real data).

### 16.6 Integration + convergence with show fun ✅ BUILT (2026-06-22)

- **Standalone:** "open registry" → the registry Miller.
- **In show fun:** ✅ a **"Registries"** branch at the bottom of the left column drilling into the SAME tree
  (registries → entries → Edit/Rename/Remove). Implemented as one `MlBranch("registries", …)` in
  `_VoiceCommandEditorRootNodes` whose `children` = `_RegistryEditorRootNodes` (the registry editor's own
  root builder) — the identical node tree, mounted one level deeper. No duplicated logic.
- **Registry entries are searchable + editable FROM show fun ✅ [Jamie 2026-06-22].** Typing `doc journal`
  in show fun surfaces the *specific* registry entry (the "journal" google-doc) under "more results", with
  its location path (Registries → Google Docs → Journal) previewed on the right; Enter drills into that
  entry's SAME action rows (Edit value / Rename / Remove) as the registry editor. **Mechanism:** the
  Registries branch carries a **per-node `search_index`** (`_RegSearchIndex`, the registry editor's own flat
  index) — the engine pulls it during the search walk with paths RELATIVE to the mount node, so the index's
  `[regId, "entry:"+key]` resolves to `["registries", regId, "entry:"+key]` in the live tree. Functions stay
  live-walked (each a `search_skip` branch); only the registry subtree uses the flat index, so search stays
  instant across both worlds. One backend (the registry layer), surfaced from both Millers.
- **"show fun `<text>`" ✅:** mirrors "open registry `<text>`" — `OpenVoiceCommandEditor(token)` treats an
  exact function name as `initial_path` (drill in) and any other text as an `initial_filter` (pre-fill the
  search box). Voice: `show fun <textnv>`. Internal reopens (after add/remove/rephrase) always pass a real
  function name, so they keep drilling in as before.
- **Editing a registry entry from show fun** reopens the **Registry Editor** at that registry (same GuiHost,
  no reboot) — the unified system's natural landing. (A future refinement could return to show fun instead.)
- **Master menu:** both expose `*AsNode()` so a future single root lists Functions + Registries + … .
- **Convert** the `OpenDirectoryConfigGui` Picker (`DirectoryRegistry.ahk`) **into** this Miller
  (directories/programs become two registries in it) — consolidation, not a parallel editor.

### 16.7 Build phases

1. **Headless registry layer** (`Scripts/.../registries.py`): discover (16.1) → infer schema (16.2) →
   `list` / `get` / `add` / `set` / `remove` per entry, uniform across families, atomic + reload-aware.
   The auto-derived linkage (16.3) lives here. Fully testable headless (mirrors `voice_by_function.py`).
2. **Registry Miller** (`Helpers/RegistryEditorMenu.ahk`, GuiHost) on the layer: Registries → entries →
   edit/add/remove rows; `search_index` across all entries; voice "open registry".
3. **show-fun integration** ✅: "Registries" branch (mounts `_RegistryEditorRootNodes`) + per-node
   `search_index` folding every entry into show fun's recursive search; "show fun `<text>`"; both `AsNode`.
4. **Retire hardcoding**: `voice_commands_by_tag` consumes the derived linkage → drop `EXPLICIT_CHOICE_MAP`;
   the registry layer feeds show-fun attribution (shrinks `voice_function_overrides.json` to true specials).
5. **Convert** `OpenDirectoryConfigGui` into the Miller; rewire its callers; delete the old Picker.

### 16.8 Serious design considerations (the hard parts, called out up front)

1. **Heterogeneity is the whole game** (16.2): two granularity axes × three value shapes. Get the inference +
   generic editors right and 97 registries "just work"; get it wrong and every registry is a special case.
   Raw-JSON fallback keeps unknown shapes honest.
2. **Reload reliability** (16.5): the #1 way this silently fails is a successful write that never goes live.
   Deriving the touch-target from the dump is elegant but must have a dependable fallback; verify live after
   write where possible.
3. **Write safety:** atomic writes + pre-edit backup; never corrupt a real registry on a bad edit.
4. **Linkage fragility** (16.3): key-set matching can mis-associate two registries with overlapping keys, or
   miss one assembled oddly at runtime. Treat linkage as best-effort enrichment, never a correctness gate for
   editing — a registry with no derived linkage still fully edits.
5. **Backward-compat:** `add_voice_choice.py`, `set_json_key.py`, and the 41 voice-config wizards must keep
   working (or be migrated deliberately, not broken). The layer wraps them where it can.
6. **Performance:** ~97 files / thousands of entries — read once, cache; the `search_index` is built once.
7. **Don't reduce capability:** some registries have bespoke add-flows (e.g. `AddGVBroadcast`'s member
   editor, context auto-capture from the foreground window). The generic editor must *defer to* those rich
   flows where they exist, not replace them with a worse generic form.

### 16.9 TABLED — Send-prompt system → JSON (future subsystem, Jamie 2026-06-22)

The send-prompts registry (`send_prompts.json`) maps a phrase → a **prompt-file NAME**; `SendClaudePrompt`
reads `INIDATA/ClaudePrompts/<name>.txt` and pastes (optionally auto-sends) it. So the *real* text isn't in
the JSON — it's in a `.txt`, and the registry editor can't edit it. Rather than bolt a ".txt editor" onto
the registry editor, Jamie wants to **reconfigure the whole send system to JSON**: each prompt becomes a
JSON entry carrying its **text inline**, plus per-entry **aliases**, a **behavior flag** (send vs.
paste-only), and possibly **contextual activation** (where the prompt is available). Once it's a normal
JSON registry, it edits through this editor for free (text + aliases + flags as fields). **Tabled** as its
own build; until then send-prompt *text* stays `.txt`-backed and uneditable here (key/rename/remove work).

### 16.10 Generator extensibility — extra registry sources (Jamie 2026-06-22)

Discovery must be **easy to point at more sources** than `VoiceChoices/` + `Contexts/`. Some data that acts
like a registry lives elsewhere — e.g. Jamie's **soft-added spot slots** live in the **media catalog
(E:\Media\catalog)**, not a VoiceChoices file, so the editor only shows the 8 hardcoded ones. The hook: a
small **EXTRA_SOURCES list** in `registries.py` (path/glob + family) so a new source is one declarative
line, no new code path. Wiring the media catalog specifically is deferred (rich, differently-shaped store),
but the extension point exists.

### 16.11 As-built increments (2026-06-22)

- **Speed:** the dump is cached to `~/.claude/context/registries_dump.json` (regenerated on the Stop hook +
  after every GUI write); the Miller reads the cache at boot instead of shelling Python (falls back to a
  live `dump` if the cache is missing). Trims the per-open Python+cmd spawn.
- **"open registry `<text>`":** `OpenRegistryEditor(arg)` treats a known registry id as `initial_path` and
  anything else as an `initial_filter` that pre-fills the Miller's search box (new engine opt
  `initial_filter`). Voice: `open registry <textnv>`.
- **Custom-editor links:** a registry with a bespoke editor surfaces an **"Open … editor"** action row that
  defers to it (e.g. `gv_broadcasts → AddGVBroadcast`), via a small `_RegCustomEditor` map — the "link to
  it" option (vs. replicating the rich flow in Miller).
- **show-fun ↔ registry unification (16.6):** `_VceLoadData` also calls `_RegLoadData` (best-effort);
  `_VoiceCommandEditorRootNodes` appends ONE `MlBranch("registries", …)` carrying `children =
  _RegistryEditorRootNodes` + a per-node `search_index = _RegSearchIndex`. Result: every registry entry is
  searchable from show fun and drills to the same Edit/Rename/Remove rows. Plus `OpenVoiceCommandEditor`
  gained the function-vs-free-text classification (`_VceIsKnownFunction`) so `show fun <text>` pre-fills the
  search box. Verified: search `doc journal` resolves to Registries → Google Docs → Journal; the Registries
  branch previews all 42 registries. (Drill-to-actions reuses the builders "open registry" already proves —
  Jamie's confirmed real-edit round-trip.)

---

## 17. Unified Destinations / Openers (design — Jamie 2026-06-23)

**The problem.** The "things I open" registries grew one at a time and diverged. `sites.json` and
`google_docs.json` carry **byte-for-byte the same fields** (`url, title, default_pos`); `directories.json`
is `path, program, snap` — the same "open a target at a position" shape with a different target kind. Each
has a **bespoke add-flow** (`AddGoogleDoc` InputBox chain, `AddDirectoryByVoice`/`AddProgramByVoice` forms,
`AddHardcodedSite`), so editing in the registry browser vs. "add doc" are different code paths, and "every
one of those interfaces is a little different." Position is a real shared concept but is named two ways
(`default_pos` vs `snap`), unpopulated, and rendered as free text. Jamie wants ONE schema-driven editor +
ONE opener, with position picked from a defined list, and the **same backend** whether reached from the
registry browser or an add command (edit one → edits both).

### 17.1 The model — a `type`-discriminated **Destination**

sites / google_docs / directories are the **same supertype**: *a named destination that opens to a target,
at a position.* They differ only by target kind + a couple variant fields + the open verb.

```
{ phrase,                       // the spoken key ("Journal")
  type,                         // url-doc | url-site | directory | program-ref
  target,                       // the URL, the path, or the program token
  position,                     // shared enum (snap_position): main|left|right|top|bot|full|twin
  open_mode,                    // shared enum: new_tab | focus | navigate  (URL types)
  open_fn,                      // OPTIONAL escape hatch — a specific AHK fn (enum from the function index)
  // variant, by type:
  program,                      // directory / program-ref: launcher token (resolves via Contexts)
  title }                       // url types: tab-match title
```

One dispatcher: **`OpenDestination(phrase, spokenPos)`** resolves and opens:
1. `open_fn` set → call **that function** `(target, resolvedPosition)`. Full control for special needs.
2. else → the **type-default mechanic** (Chrome tab / Explorer+program / launch), modulated by `open_mode`.

**Position precedence** (mirrors directories today): spoken `<snap_dest>` > `entry.position` >
`program.default_snap` > native. So `doc journal main` (one-off) and a saved `position: main` (persistent)
both work.

### 17.2 DECISIONS (Jamie: "your call" 2026-06-23 — locked)

- **Logical unification YES, physical mega-file NO.** sites/docs/directories share ONE typed schema, ONE
  editor, ONE opener — but stay in **separate JSON files**. A single `openers.json` is overkill: it buys
  phrase-collision-across-types, a grammar question (loses the "doc X"/"site X" namespacing), and migration
  risk, for ~no benefit once schema+editor+opener are already shared. (Optionally merge *only* sites+docs
  later if the grammar split stops earning its keep — not now.)
- **Keep the doc/site grammar split** (`doc <google_doc>` vs `site <site>`) — scoped namespaces are a
  feature (a doc and a site can both be "music").
- **Programs/Contexts stay their own system, referenced not absorbed.** A `directory`/`program-ref`
  destination *borrows* its launcher from the Contexts program registry (`program` token → `open_fn`/
  `launch_fn`/`default_snap`). url destinations touch Contexts not at all. The program registry is the
  "launcher library"; Destinations are "saved places." One reference direction only.
- **`open_fn` is the escape hatch** (NOT `open_mode`): optional per-entry AHK fn, signature
  `(target, resolvedPosition)`, matching the existing program-registry `open_fn` convention. Rendered in the
  editor as an **enum picked from the function index** (the list show fun already knows) — "pick from a
  defined list, don't type it." Blank = type default.
- **`open_mode`** stays the generic new/focus/navigate enum for the type-default path; ignored when
  `open_fn` is set.
- **`default_pos` + `snap` normalize to one field: `position`** (enum `snap_position`, sourced from
  `SnapByAlias` — single source of truth).
- **Typed field schema lives in `CHOICE_SETS`** (`value_fields` upgraded from bare names to specs
  `{name, type, values?, default?}`; a bare string stays a string field — backward-compatible).
- **Enum fields render as a drill-to-pick sub-level** in the Miller editor (no typing).
- **Add-flows route through the shared schema-driven editor** — `add doc`/`add site`/etc. become "open the
  schema-driven add for set X," same editor + data + backend as browsing. The directory/program flows keep
  their **foreground auto-detection** step, then hand off to the shared field editor for the rest.
- **Open→register-context bridge** (today's `add site → add context`) extends to docs/directories — opt-in,
  no-op when a context already covers it. No context-*scoping* of destinations now (leave a future
  `context` field as a hook; don't build it).

### 17.3 Shared value vocabularies

Defined once, referenced by `values` name on an enum field. Some static, some registry-derived:

| Vocab | Source | Used by |
|---|---|---|
| `snap_position` | `SnapByAlias` aliases (main/left/right/top/bot/full/twin) | every destination's `position` |
| `open_mode` | static (new_tab/focus/navigate) | url destinations |
| `function` | the function index (`fn_to_file_index.json`) | `open_fn` |
| `program` | program registry (Contexts with a `program` block) | directory/program-ref `program` |
| `monitor` | static (1/2/3…) — future | future `monitor` field |

### 17.4 The vertical-slice build plan (prove the pattern, then widen)

1. **Slice — google_docs `position`, end to end** (the proof): typed `value_fields` for `google_doc`
   (`url`, `title`, `position:enum=snap_position`); the registry editor renders `position` as an enum
   drill-pick; `OpenDestination`/the doc opener honors persistent `position` + spoken `doc journal main`;
   migrate `default_pos`→`position`. Validates schema → enum-pick → opener-honors before mass conversion.
2. **Widen the schema** to sites/notepad/etc.; add `open_mode` + `open_fn` rendering.
3. **One opener dispatcher** `OpenDestination(phrase, spokenPos)`; repoint `doc`/`site`/`open` grammars.
4. **Route the add-flows** through the shared editor; directory/program keep their detection step.
5. **Fold directories/programs** into the registry-editor surface (this is §16.7 Phase 5) — directories
   become a `type`, programs stay context-backed but edit through the shared field editor.

### 17.5b As-built increments (2026-06-23)

- **Typed field schema (Python).** `registries.py` gained `FIELD_TYPES` (field-name → widget/enum) +
  `_STATIC_VOCAB` + `_resolve_vocab` + `field_schema()`, carried per-registry in the cached dump. `position`/
  `snap`/`default_pos` → enum snap_position; `program` → enum auto-resolved from the program registry;
  `open_mode`/`open_fn` → enum; `url`/`path` typed text. No cross-language rename — the schema abstracts the
  stored field name. Verified in the cache for docs/sites/directories.
- **Field-level pick editor (Miller).** `RegistryEditorMenu.ahk`: an object entry now decomposes into one
  row PER FIELD (`name = value`); an enum field drills to a PICK LIST of allowed values (current marked,
  `(clear)` to unset), text fields edit inline. `_RegSetField` merges one field into the full entry +
  writes + reloads; `OpenRegistryEditor(arg, internalPath)` reopens DEEP at the entry so you stay put.
  Verified by deterministic screenshot (field rows + `default_pos` typed as a pick) + dry-run write.
- **Unified schema-driven add (Miller "+ Add entry").** `_RegAddEntry` rebuilt: key prompt → duplicate
  guard → `_RegAddSeed` (url fields pre-filled from the current Chrome tab) → per-field collection where
  ENUM fields are PICKED (`_RegPickEnum`, single-select) and others are text. Same `field_schema` + same
  write backend as editing — add and edit are one system. Verified: real add+remove round-trip on live
  google_docs (wrote `default_pos:"left"` via add_voice_choice with the reload touch, then cleaned up).
- **`open_fn` escape hatch — built for google_docs (2026-06-23).** An optional per-entry AHK function that
  opens the doc instead of the default Chrome-tab mechanic (sec 17.2). `google_doc` value_fields gained
  `open_fn` (optional); `add_voice_choice` now PADS missing optional trailing fields so the older 3-arg
  callers (AddGoogleDoc) keep working alongside the editor's 4-arg writes. `web_opener_commands.py` packs
  `url|||title|||default_pos|||open_fn`, and `_doc_open_action` dispatches `open_fn(url, default_pos)` when
  set (verified in isolation). The editor renders `open_fn` as an enum picked from the function index
  (resolved AHK-side via `_RegFunctionVocab`, not embedded in the cache — `field_schema` caps embedded enum
  values at 250). Verified: schema in cache, both-arg-count writes, dispatch logic, editor screenshot (the
  open_fn row).
- **`open_fn` extended to sites + directories (2026-06-23) — escape hatch now COMPLETE across all openers.**
  Sites: `_iter_context_links` + `load_sites` pack a 4th `open_fn` field (from both sites.json AND context
  links), `_unpack_site_value` returns it, `_site_open_or_go_to_action` dispatches `open_fn(url, default_pos)`
  before the wait/snap/focus logic. Directories: `DirRegLookup` reads `open_fn`, `OpenDirectoryByName`
  dispatches `open_fn(path, snap)` (try/catch tooltip on a bad fn) before `ProgRegOpenDir`. `site` +
  `directory` choice sets gained the optional field; the editor shows the `open_fn` pick on all three.
  Verified: 4-arg + back-compat 3-arg writes for both, site dispatch logic in isolation, AHK validate +
  closure. Signature is uniformly `open_fn(target, position)` (target = url or path).
- **NOT yet:** routing voice `add doc`/`add site`/`add directory` through this shared add (their bespoke
  flows carry dup-detection / alias-on-duplicate / context-bridge / fuzzy-title that must be PORTED into the
  shared core first, per 16.8 #7 — don't regress them).

### 17.5c Smart "add site" — CONTEXT-DRIVEN type detection + routing (2026-06-23)

`add site` is a universal, type-detecting entry point — and the type is the page's **CONTEXT**. Contexts are
the SINGLE URL classifier for BOTH foreground detection AND add-routing (no parallel matcher — Jamie caught
that a standalone classifier duplicated the context matchers). A new destination type = a **context** (created
via "add context") with a `destination` block; no code.

- **The `destination` block on a context** — exactly parallel to ProgramRegistry's `program` block. A context
  with one is ADD-ROUTABLE:
  ```json
  "destination": { "store": "links" | "registry:<id>", "open_fn": "" }
  ```
  `store: "links"` = the new destination becomes a **link on that context** (the sites-in-contexts model);
  `store: "registry:<id>"` = stored in a VoiceChoices registry. `open_fn` = the optional per-type opener.
- **`Helpers/ContextDestinations.ahk`** — the "destination" ROLE, mirroring ProgramRegistry:
  `ContextDestinationLookup(token)` (the block), `ContextDestinationDetectForeground()` (walk the foreground
  chain deepest→root for the first context with a block — like `ProgRegDetectForeground`),
  `ContextDestinationAllTokens()` (for the override picker).
- **`AddSmartDestination`** (voice `add site` / `add site <textnv>`): `ChromeCurrentUrl` →
  `ContextDestinationDetectForeground` → `_DestConfirmContext` (detected context is row 1 = Enter-accepts;
  pick another, or "Plain site"; nothing matched → plain site is row 1) → route by `store`: `registry:<id>`
  goes through the shared `_AddIntoRegistry` (seeded url + open_fn), `links` goes through `_AddLinkToContext`
  (phrase + url + title + position, written via `site_contexts.py add` — the Site Browser's writer).
- **Wired so far:** `youtube.json` gained `destination: {store: "links"}` (YouTube already stores its sites as
  links on that context); a new `google_docs.json` context (`match: docs.google.com/document`,
  `destination: {store: "registry:google_docs"}`) routes docs to their existing registry. Adding Instagram /
  Reddit / etc. = a `destination` block on those contexts (Instagram currently matches its Messages page by
  title, so it'd need a url match for generic pages).
- **`destinations.py` + the sites `type` field were RETIRED** — the context replaces both the classifier and
  the stored type marker. Verified: AHK validate + closure (ContextDestinations + the `site_contexts.py`
  writer resolve), context JSON valid, sites schema back to 4 fields. Mirrors the proven program-role
  machinery; the live detect→route→write rides Chrome + modals (not headless-testable here).
- **Regression note (acceptable per "breakage fine"):** `add site` no longer runs the old `AddHardcodedSite`,
  so its URL-dup-detection / alias-on-duplicate / `add site → add context` bridge are temporarily gone — those
  become generic capabilities folded into the shared add next.

### 17.5d Capability reroute — the dropped smarts, restored GENERICALLY (2026-06-23)

The smarts §17.5c temporarily dropped from `add site` are back — but as **DERIVED, schema-driven capabilities**
on the shared add, so EVERY url/path-identity registry (sites, docs, any future opener) gets them for free with
zero per-registry config. This is §18.3 step 1, done for the url-identity registries.

- **Derived capabilities (`registries.py` `registry_capabilities`)** — read straight off the typed field
  schema, carried per-registry in the cached dump as `capabilities`:
  - `identity_field` = the `url`/`path`-typed field (the thing that makes two entries "the same destination").
  - `title_field` = a `title` field, seeded from the live Chrome tab title.
  - `context_bridge` = `true` iff the identity is a url (so the macro-context offer only fires where it makes
    sense). Derived, not a hand-set flag.
  Verified: sites/docs → `{identity: url, title: title, bridge: true}`; directories → `{identity: path, bridge:
  false}`.
- **Generic duplicate detection (`registries.py identity-matches <id> <value> [exclude]`)** — replaces the
  site-only `_VoiceConfigReverseLookupSiteAll`. Two-tier match (exact, else url-normalized: drop
  query/fragment/trailing slash via `_normalize_url`), excludes the entry being written. Verified live: exact,
  `?usp=sharing`-stripped normalized, exclude-self, and miss all correct.
- **Shared add wiring (`RegistryEditorMenu.ahk`)** — both the voice path (`_AddIntoRegistry`) and the editor
  "+ Add entry" (`_RegAddEntry`) now, after collecting fields: read the FINAL identity value → `_RegDupCheck`
  (calls `identity-matches`; on a hit shows the existing `_VoiceConfigAliasPrompt` alias/replace modal) →
  thread `--replace-key` per dropped alias into the write → `_RegMaybeContextBridge` (calls the existing
  `_AddSiteContextBridge`) post-write. Title is seeded as a PREFILL (prompts, editable) vs the url `seed` which
  is AUTHORITATIVE (no prompt) — `_AddIntoRegistry(regId, seed, phrase, prefill)`. New helpers: `_RegCaps`,
  `_RegChromeTabTitle` (cross-desktop via `WinExistAcrossDesktops`), `_RegIdentityMatches`, `_RegDupCheck`,
  `_RegFieldValueInArgs`, `_RegMaybeContextBridge`. `_RegWriteAndReopen` now accepts an Array of replace keys.
- **`add doc` rerouted** → new `AddDocSmart` (forces the `google_docs` registry, seeds url+title, rides the
  shared capabilities) — generic replacement for the bespoke `AddGoogleDoc`. The old flow stays reachable under
  **`add doc legacy`** as a safety net until `AddDocSmart` is keyboard-verified, then both it and the other
  bespoke add flows can be deleted.
- **`add directory` is DEFERRED on the bespoke `AddDirectoryByVoice`** — deliberately. Its smarts are
  genuinely specialized (Explorer/terminal/selection PATH auto-detect, path validation, dup-check against BOTH
  the directory AND program key spaces, and a folder-CAPABLE program picker), none of which the generic
  url-identity path covers. Rerouting it is a separate task (would need path-seed + path dup-detection — the
  `path` identity already derives — plus a folder-capable program-vocab filter). Low reuse, higher risk; not
  worth bundling here.
- Verified: `registries.py` capabilities + `identity-matches` (isolated + live round-trip add→detect→alias-
  collapse→remove on `google_docs`), AHK validate (exit 0) + closure (clean) + the cross-file refs resolve
  (`WinExistAcrossDesktops`, `_VoiceConfigAliasPrompt`, `_AddSiteContextBridge`, `_LN_JsonParse`), root editor
  screenshot renders all 42 registries. ✗ live voice add→dup-prompt→bridge (Chrome + modals, not
  headless-testable) — the writer chain underneath it IS proven.

### 17.5e Quiet add by default + two editor bugfixes (2026-06-23)

Jamie: "for most of the time when adding a site it will just add it and then open the registry editor at
its location instead of asking me… it should just default to blank and then I can edit them manually… for
95% no prompts besides the voice command… but it will always open the registry right there so I can easily
edit them right in place."

**Quiet add (the new default for `add site` / `add doc`).** `AddSmartDestination` no longer runs the picker +
4-prompt wizard. It now: read URL → detect context → **write immediately** with blank/default fields → **open
the Registry Editor RIGHT ON the new entry** (lands on its field rows). ~95% of adds = one voice command, zero
modals; the imperfect bits (phrase, position) are edited in place.
- **Phrase:** the spoken word (`add site <name>`), else derived from the tab title (`_DestDerivePhrase`:
  leading page-name segment, punctuation-stripped, ≤5 words; domain fallback). `_DestUniqueKey` appends " 2"
  so a derived phrase never silently overwrites a different entry.
- **Fields:** url (identity) + title (tab title) + the context's `defaults`; everything else blank. Edit in
  the editor.
- **Duplicate URL → just opens the EXISTING entry** (no dupe, no modal) — `identity-matches` from 17.5d.
- **Per-context policy on the `destination` block:** `add_mode: "ask"` opts a context back into the full
  interactive wizard (`_DestAddInteractive`, the old flow); `defaults: {default_pos:"left", …}` pre-seeds
  fields on a quick add. (A global "always ask" toggle is a trivial follow-up — per-context covers the asked-for
  "specific contexts" case.)
- **Routing:** destination context with `registry:<id>` store → `_DestAddQuiet` into that registry (editable in
  the Miller, opens on the entry); `links` store → `_DestAddLinkQuiet` (link on the context, opens on the
  context entry); no destination context → the plain `sites` registry (the common case).
- **Deep-open mechanic:** `OpenRegistryEditor(regId, entryKey)` now lands deep on an entry from a FRESH MAINFUN
  process (routes through the GuiHost with both args; `_Mcp_NavigateToPath` resolves the 2-level path
  `[regId, "entry:"key]`). ✓ screenshot: `add`→lands on the entry's url/title/default_pos/open_fn rows.

**Bugfix 1 — sites vanished from the editor (THE reported bug).** Building the Sites entry list threw
`Map has no method "ToString"` (`RegistryEditorMenu.ahk:220`) on the FIRST site carrying a nested `open` block
(`[open] voice` = `{deck_profile, mode}`): `String(aMap)` throws in AHK v2, and the exception aborted the whole
entry-list build, so **no** sites rendered. Fixed with `_RegStr` (recursive safe stringify → compact JSON-ish
text for nested Map/Array), routed through every value-stringify spot in the editor. ✓ screenshot: all 32 sites
render, including `[open] voice`.

**Bugfix 2 — "Search capped (budget)".** The registry editor's search universe was built by a live tree-walk
capped at 3000 nodes (42 registries × entries × field-action rows), truncating before the later registries
(Sites is #37) — yet the opts-level `_RegSearchIndex` ALREADY dumps every entry into the universe, so the walk
was pure redundancy. Marked each registry branch `search_skip` (emit the node, don't descend). ✓ screenshot:
search for "Discover" is clean, no cap message.

### 17.5f Per-registry ADD POLICY — editable in the editor (2026-06-23)

Jamie: "edit the defaults for each registry type right there inside the registry… a row at the bottom… each
unique field shows up as a row and I can toggle them or set new defaults… defaults with ask or without ask."
Plus: "for adding site I do want to ASK about the phrase, I do not want that to be a default, but everything
else I want to default."

Every object registry now carries an **add policy** — per-field, how the quiet add fills it — editable in
the Registry Editor with **zero code per registry**.

- **Per-field policy `{source, value, ask}`** (`registries.py`, stored in `INIDATA/registry_add_policy.json`,
  carried per-registry in the dump as `add_policy`):
  - `source`: `auto` (url/path → the page, title → the tab title, `_phrase` → derived from the title) ·
    `fixed` (use `value`) · `none` (blank).
  - `ask`: prompt for the field on add (pre-filled with the resolved value) vs fill silently.
  - `_phrase` = the pseudo-field for the entry KEY / spoken phrase.
- **Built-in defaults** (`default_field_policy`): `_phrase` → **ask** (Jamie's request — the phrase is always
  confirmed, never silently guessed; the spoken word from `add site <name>` is the prefill); `url`/`path`/`title`
  → auto + silent; everything else → none + silent. So a typical `add site` is **one phrase confirm + nothing
  else**, and the new entry opens in the editor for any tweak.
- **The editor UI** — a **`⚙ Add defaults`** row under each registry's entries. Drill → one row per field (+
  `(phrase)` first), each showing `source=… · ASK/silent · value=…` and drilling to a small action list:
  **Ask when adding** (toggle), **Source** (auto/fixed/none pick), **Fixed value** (enum → pick list, else text;
  setting it implies `source=fixed`). Writes shell `registries.py set-add-policy` + reopen DEEP at the field.
  Code: `_RegAddPolicyNodes`/`_RegPolicyFieldNode`/`_RegPolicyActions`/`_RegPolicySourceNodes`/
  `_RegPolicySetValue`/`_RegPolicySet` in `RegistryEditorMenu.ahk`; CLI `add-policy` / `set-add-policy`.
- **Quiet add honors the policy** (`_DestAddQuiet`): phrase resolved+asked per `_phrase`'s policy (spoken word
  wins as the prefill); each field resolved by its `source`, then asked iff `ask` (enum → pick, else text). A
  context's `destination.defaults` still override a field's resolved value.
- **Deep-open** now also reaches special `__`-prefixed rows (`OpenRegistryEditor(regId, "__addpolicy")` →
  `[regId, "__addpolicy"]`), not just `entry:` keys.
- Verified: policy defaults + override + dump round-trip (isolated), AHK validate + closure, screenshots of the
  Add-defaults list (per-field rows, phrase=ASK) and a field's action list. ✗ live voice quiet-add honoring a
  custom policy (Chrome) — the resolution logic + writer are proven.
- Open follow-up (noted, not built): a GLOBAL "always ask" toggle (today it's per-field per-registry, which
  covers the asked-for cases); and the phrase interpretation chosen here is "spoken word prefills, ask=true
  still prompts" — flip `_phrase` ask off to make a named add fully silent.

### 17.5g Multi-instance TYPES get their own flat registry (2026-06-23)

Jamie added a YouTube video and it landed as a nested LINK on the youtube context — so "edit links" showed
EVERY video as one giant blob, and the add opened the general youtube context page, not the new video. Root
cause: the youtube context used `destination.store:"links"`, which appends to the context's `links` array
instead of creating a first-class entry.

**Decision:** a **multi-instance destination TYPE** (YouTube videos, and later Instagram pages, Reddit posts,
…) becomes its **OWN flat registry**, not a nested `links` blob. The `links`-on-context model is fine for a
**single-site** context (amazon = one site) but wrong for a type with many instances — each instance must be a
separately editable row with the type's own add-defaults.

- **`youtube_videos` registry** = a new `youtube_video` CHOICE_SET (`add_voice_choice.py`, site-shaped fields
  url/title/default_pos/open_fn) → auto-discovered as a writable registry with its OWN `⚙ Add defaults`.
- **Migrated** the youtube context's 7 links → `youtube_videos.json` entries (one-off; `.bak` kept), removed the
  `links` array, set `destination: {store:"registry:youtube_videos"}`.
- **`load_sites` now unions `SITE_REGISTRY_PATHS`** (sites.json + youtube_videos.json + …) via the shared
  `_load_site_registry`, so the videos stay sayable through the `<site>` grammar (`open mythical` etc.). Adding a
  new TYPE that feeds `<site>` = one path in that list.
- **Result:** on a YouTube page, `add site` → detects youtube → routes to `youtube_videos` → opens the editor ON
  the new video (its own row, own fields), exactly like docs. ✓ screenshot: 7 separate video rows + `+ Add
  entry` + `⚙ Add defaults`; per-video field rows in the preview.
- The `links` store + `_DestAddLinkQuiet`/`_AddLinkToContext` are now DORMANT (no context uses `store:links`
  after this) — kept for possible future grouping, not on any live path.
- **Still context-links (deliberately, for now):** the ~23 single-site contexts (amazon, gmail, …). They feed
  `<site>` and aren't blob-painful (one/few links each). Convert any of them to a flat registry the same way if
  it grows into a multi-instance type. **Open question for Jamie:** convert those too, or leave them?

### 17.5h Context links FLATTENED into the Sites registry (2026-06-23)

Decision (Jamie): "I want them to all be flat registries." The sites-in-contexts model (sites stored as nested
`links` arrays on 23 contexts) conflicted with first-class editing. All 28 links (1–3 per context, none a
multi-instance type) were **migrated into sites.json** as flat entries (`.bak` per file); the `links` arrays were
removed from the contexts (their `match`/`display`/`parent` stay for foreground detection). `load_sites` now
unions only `SITE_REGISTRY_PATHS` (sites.json + youtube_videos.json) via `_load_site_registry` — the
`_iter_context_links` union is dropped (the links were already duplicated in sites.json, so the grammar is
unchanged). 3 youtube phrases that were in BOTH were removed from sites.json (they live in youtube_videos now).
Result: sites.json = 29 entries, youtube_videos = 7, `<site>` grammar = 36, no dupes.
- **Downsides (raised + accepted):** loss of per-context grouping (compensated by favorites + search); the
  `_iter_context_links` / site_contexts.py / SiteBrowserMenu / `_DestAddLinkQuiet` link machinery is now DORMANT
  (no live path), kept not deleted. No grammar/behavior change.

### 17.5i Favorites / hidden ordering — registry + entry level (2026-06-23)

Jamie: "set favorites and hidden that … pop to the top or the bottom … with the built-in Miller breakpoint
things. … make sites and YouTube and documents pop to the top, and specific documents like journal and lyrical
pop to the top of that list locally."

A two-scope ordering layer in the Registry Editor:
- **Data (`registries.py`):** `INIDATA/registry_prefs.json`, in the dump as `prefs`. Scope `"_registries"`
  orders the root registry list; scope `"<regId>"` orders that registry's entries. Each maps key → `"fav"` |
  `"hidden"`. CLI `prefs` / `set-pref <scope> <key> <fav|hidden|normal>`.
- **Rendering (`RegistryEditorMenu.ahk`):** `_RegPartitionByPref` splits a level's nodes into
  **★ fav (top) ─divider─ normal ─divider─ · hidden (bottom)** (markers via `_RegMarkNode`, unique-key dividers
  via `_RegDivider`). Applied at root (`_RegistryEditorRootNodes`) and per registry (`_RegEntryNodes`, entries
  only — the Add / Add-defaults rows stay below). Unmarked levels render unchanged.
- **Setting it (`_RegPrefRowActions`):** N.2 = ★ Favorite→top, N.3 = ·Hide→bottom, N.4 = Normal, as the
  Miller's built-in **N.M row actions** on ANY registry or entry row. Writes via `set-pref` + reopen at the level.
- Seeded to Jamie's named config: Sites / Youtube Videos / Google Docs favorited; Journal + Lyrical favorited in
  Docs. ✓ screenshots: registry list (★ Sites/Youtube at top + divider) and Google Docs (★ Journal/Lyrical at top).
- **Bug found + fixed:** `_RegLoadData`'s `global` line omitted `RegPrefs`, so the assignment made a discarded
  LOCAL and prefs never loaded (AHK v2 assume-local). Added it to the declaration. (Gotcha logged in §18.4.)

### 17.5j Favorite / hide as VISIBLE ROWS (2026-06-23)

Jamie wanted to set favorites/hidden via rows, not only the discoverable N.M actions. Added:
- **Entry level:** a **`⭐ Favorite / hide`** branch in each entry's action list (`_RegEntryActions`) → drills to
  Favorite / Hide / Normal rows (current marked).
- **Registry level:** a **`⭐ Favorite / hide registry`** row at the bottom of each registry's entry list
  (`_RegEntryNodes`) → same three options for the whole registry (scope `_registries`).
- Both built on `_RegFavHideNodes`/`_RegFavHideRow` → `_RegPrefSet` (the same writer the N.M actions use, kept
  too). ✓ screenshots: the rows render in Journal's actions and youtube_videos' entry list.

**Note (resolved — NOT a bug):** youtube_videos.json appeared to "lose" 2 of 7 entries mid-session; Jamie
confirmed she had **deleted those videos herself**. No data-loss bug. (The recovery from `Contexts/youtube.json.bak`
re-added them; they can be removed again if unwanted.) The `.bak` files (per-context + sites.json) remain the
safety net for the flatten/migration.

### 17.5k URL/path AUTO-GROUPING within a registry (2026-06-23)

Jamie's design (discussed + locked): a destination's GROUP is derived from a **tunable prefix of its identity**
— url by domain/host/path, path (directories) by drive+ancestor folders ("E:\Media") — with a **manual override**
and easy **promotion** to a dedicated registry. Computed live (not stored) so tuning re-groups instantly.

- **Engine (`registries.py`, sec-17.5k block):** `group_key(value, type, level)` — url level 0 = registrable
  domain (small built-in multi-TLD list for `co.uk` etc.), 1 = host, 2+ = host+N path segs; path level N = first
  N folders. `resolve_entry_group` = manual `group` field **>** longest custom rule (a pinned prefix→name;
  domains match by SUFFIX, paths by PREFIX) **>** default-level key. Config `INIDATA/registry_groups.json`
  (`{regId: {level, rules}}`); absent = type default (url 0, path 2). Each entry carries its resolved `group` in
  the dump. CLI: `groups` / `set-group-level` / `add-group-rule` / `remove-group-rule` / `move`.
- **Manual override field:** `group` added (optional) to the site / youtube_video / google_doc / directory
  CHOICE_SETs. Blank = auto-resolve; set = pin. Quiet-add leaves it blank (add-policy default `none`) so
  auto-grouping applies; editable as a normal field row.
- **Display (`RegistryEditorMenu.ahk` `_RegOrganizeEntries`):** NORMAL entries cluster under group headers
  (shown when 2+ groups); ★ favorites still float to the very top, · hidden sink to the bottom.
- **Tuning UI:** a **`⚙ Grouping`** row per auto-grouping registry → pick the granularity (Domain / Host /
  +path… or Depth 1/2/3…) — re-groups every entry at once.
- **Transport:** a **`Move to registry…`** row in each entry's actions → pick a writable target → maps fields by
  name, writes (typed + reload), removes from source, lands on the moved entry (`move_entry`).
- ✓ Verified: engine unit tests (domain/host/path keys, suffix vs prefix matching, multi-TLD, override
  precedence), live `groups sites` (clustered by domain: google.com×8, amazon.com×2…), screenshots of the grouped
  Sites list + the ⚙ Grouping picker, move dry-run field-mapping, level-change re-grouping.
- **Guided-segment rule creator — BUILT (2026-06-23).** Each url/path entry's actions now has a **`✦ Group rule
  from this…`** branch → `registries.py group-candidates <reg> <value>` returns the granularity ladder for THAT
  entry (Domain → Host → Host+1/2 segments, or path depths), each row annotated with **how many existing entries
  the rule would capture** ("Domain → amazon.com  (groups 2)"). Picking writes `add-group-rule` and re-groups. The
  `⚙ Grouping` node also lists existing **custom rules** with a one-tap **✕ remove** (`remove-group-rule`).
  Closures capture the prefix by value (IIFE), per the AHK-v2 loop-capture trap. ✓ Verified: CLI counts on
  url+path registries, full add→group→remove round-trip, dump carries `group_config.rules`, screenshot of the
  per-entry action row.
- **One-click promote group → registry — BUILT (2026-06-23).** Each auto-grouping registry now has a **`⬆ Promote
  group → its own registry`** node listing every current group + its size; picking one runs `registries.py
  promote-group <src> <group>`. The promote is **fully data-driven** — it writes `INIDATA/promoted_registries.json`
  (`{id:{json,template:"site",source,name}}`), creates an empty `VoiceChoices/<id>.json`, then moves every group
  member in. Two readers pick the config up with **zero source edits**: `add_voice_choice._register_promoted_
  registries()` clones the `site` choice set (so the new registry gets typed writes + the web_opener reload touch),
  and `web_opener_commands._promoted_site_registry_paths()` unions it into `load_sites()` (so promoted sites stay
  openable in the `<site>` grammar — that union IS the "inherit the rule" Jamie asked for). In-process the promote
  busts the cached `add_voice_choice` module (`sys.modules.pop`) so the just-written choice set is live for the
  moves. CLI: `promote-group <src> <group> [new_id] [--dry-run]`. ✓ Verified end-to-end on a throwaway `plex.tv`
  group: new registry discovered (object/site fields/choice_set/writable), entries moved with intact field dicts,
  left the source, `load_sites` union included both phrases — then fully torn down (moved back, config+json
  deleted, 43/492 baseline restored, U+00B7 title byte-verified uncorrupted). Screenshot of the promote list.
- **Edge:** localhost/IP urls (`127.0.0.1:7459`) group oddly at level 0 (registrable-domain on an IP); harmless,
  manual-overridable, or use level 1 (host). github.io-style public-suffix hosts cut to the suffix (rare).

### 17.5 Interaction map (so the seams stay honest)

```
Contexts (program block)  =  LAUNCHER library   ("how to start/open app X" + foreground detection + bind scope)
Destinations              =  saved PLACES        ("go to this thing, at this position")
        directory/program-ref ──► names a launcher from Contexts (one ref direction)
        url-doc/url-site ───────► self-contained (Chrome), ignores Contexts
        open_fn ────────────────► names any AHK fn, no Context needed
        on add (opt-in) ────────► offer to register a Context for the opened window (site→context bridge)
```

Destinations never absorb Contexts and vice-versa; the only hard edge is `program-ref → program registry`
for launching, plus the optional add-time bridge.

---

## 18. HANDOFF SNAPSHOT (2026-06-23 — read this first after a compaction)

The unified opener / Destination work (sec 17) was built across one long session. This is the complete
current state so work can resume without re-deriving it.

### 18.1 What's BUILT + verified

1. **Typed field schema (Python).** `registries.py`: `FIELD_TYPES` (field-name → widget/enum), `_STATIC_VOCAB`
   (snap_position, open_mode), `_resolve_vocab` (also `program` from the program registry, `function` from the
   fn index), `field_schema()` (caps embedded enum values at `_EMBED_VALUES_MAX = 250`; the 2253-entry
   `function` vocab ships name-only, resolved GUI-side). Carried per-registry in the cached dump. ✓ cache.
2. **Field-level pick editor (Miller).** `RegistryEditorMenu.ahk`: an object entry → one row PER FIELD
   (`name = value`); enum fields drill to a PICK LIST; text fields edit inline. `_RegSetField` merges one field
   + writes + reloads; `OpenRegistryEditor(arg, internalPath)` reopens DEEP at the entry. ✓ screenshot + dry-run.
3. **Unified schema-driven add (`_RegAddEntry`).** Key prompt → duplicate guard → `_RegAddSeed` (url fields
   pre-filled from Chrome) → per-field collect (enum = `_RegPickEnum` single-select, else text). ✓ live
   add+remove round-trip.
4. **`open_fn` escape hatch — ALL three openers.** Per-entry AHK function that opens an entry instead of the
   default mechanic, signature `open_fn(target, position)`. Docs+sites: `web_opener_commands.py` packs
   `url|||title|||default_pos|||open_fn`, `_doc_open_action`/`_site_open_or_go_to_action` dispatch it.
   Directories: `DirRegLookup`+`OpenDirectoryByName` dispatch `open_fn(path, snap)`. `add_voice_choice` gained
   the optional field on all three + **optional-trailing-field PADDING** (adding an optional field never breaks
   shorter callers). Editor picks open_fn from the fn index (`_RegFunctionVocab`). ✓ writes (4-arg + 3-arg
   back-compat), dispatch logic isolated, editor screenshot.
5. **Smart `add site` — CONTEXT-DRIVEN (sec 17.5c).** The page's CONTEXT is the type. `ContextDestinations.ahk`
   (the `destination` role, mirrors `ProgramRegistry`). `AddSmartDestination` walks the foreground context
   chain → routes by the context's `destination` block: `store: "links"` → `_AddLinkToContext` (via
   `site_contexts.py add`), `store: "registry:<id>"` → `_AddIntoRegistry`. Wired: `youtube.json` (links),
   new `google_docs.json` context (registry:google_docs). `destinations.py` + the sites `type` field were
   RETIRED. ✓ validate+closure, JSON. ✗ live detect→route→write (needs Chrome+modals, not headless-testable).
6. **Capability reroute — generic dup-detect / alias / context-bridge / title-prefill (sec 17.5d).** Derived
   `capabilities` on every registry (`registries.py registry_capabilities`: identity_field, title_field,
   context_bridge — all read off the typed schema). `registries.py identity-matches` = the generic url-normalized
   duplicate finder. The shared add (`_AddIntoRegistry` + the editor's `_RegAddEntry`) now does dup-detect →
   alias/replace modal → `--replace-key` collapse → post-write context bridge → title prefill from the live tab.
   `add doc` rerouted to `AddDocSmart` (bespoke `AddGoogleDoc` kept as `add doc legacy` safety net). `add
   directory` deliberately LEFT bespoke (specialized path-detect + folder-capable program picker). ✓ caps +
   identity-matches isolated + live round-trip, AHK validate+closure, root-editor screenshot. ✗ live voice
   add→prompt→bridge (Chrome+modals) — but the writer chain under it is proven.
7. **Quiet add by default + 2 editor bugfixes (sec 17.5e).** `add site`/`add doc` now write immediately (blank
   defaults) and open the editor RIGHT ON the new entry — ~95% zero-prompt. Per-context `add_mode:"ask"` /
   `defaults` on the `destination` block. New deep-open `OpenRegistryEditor(regId, entryKey)` (via GuiHost, 2-level
   nav). FIXED: sites vanished from the editor (`String(nestedMap)` crash on the `open` block → `_RegStr`); and
   "Search capped" (redundant live-walk → `search_skip` on registry branches). ✓ screenshots: deep-open lands on
   field rows, all 32 sites render, clean search. ✗ live voice quick-add (Chrome) — components all verified.
8. **Per-registry ADD POLICY, editable in the editor (sec 17.5f).** Each object registry has a per-field
   `{source, value, ask}` policy (`registries.py` + `INIDATA/registry_add_policy.json`, in the dump as
   `add_policy`). Built-in defaults: `_phrase` ASKS, url/title auto+silent, rest none+silent — so `add site` is
   one phrase confirm. Editor: a `⚙ Add defaults` row → per-field rows → toggle Ask / set Source / set Fixed
   value. `_DestAddQuiet` honors it. Deep-open extended to `__`-prefixed special rows. ✓ policy round-trip +
   screenshots (per-field list, field action list).
9. **YouTube videos = own registry; ALL context links flattened into Sites (sec 17.5g–h).** Multi-instance TYPES
   get their own flat registry (`youtube_videos`, feeds `<site>` via `SITE_REGISTRY_PATHS`); the 23 single-site
   contexts' 28 links were flattened into sites.json (de-duplicated; `.bak` per file). `_iter_context_links`
   dropped from `load_sites`; the link machinery is DORMANT. ✓ screenshots, grammar count.
10. **Favorites / hidden — registry + entry level (sec 17.5i–j).** `INIDATA/registry_prefs.json` (dump `prefs`),
    scopes `_registries` + `<regId>`. `_RegOrganizeEntries`/`_RegPartitionByPref` float ★ fav to top, sink ·
    hidden to bottom with dividers. Set via N.M row actions OR visible `⭐ Favorite / hide` rows (entry + registry).
    ✓ screenshots both levels. (Fixed: `RegPrefs` missing from `_RegLoadData`'s `global` — assume-local gotcha.)
11. **URL/path AUTO-GROUPING (sec 17.5k) — COMPLETE incl. both UI pieces.** Entries cluster by a tunable identity
    prefix (url domain/host/path; path drive+folders), computed live. `registry_groups.json` (dump `group_config` +
    per-entry `group`). Manual `group` override field. `⚙ Grouping` row tunes granularity + lists/removes custom
    rules; `Move to registry…` transports entries. **Guided-segment rule creator** (per-entry `✦ Group rule from
    this…` → `group-candidates` ladder with match counts → `add-group-rule`) and **one-click promote** (registry
    `⬆ Promote group → its own registry` → `promote-group`, data-driven via `promoted_registries.json`,
    auto-registered in both `add_voice_choice` and `web_opener`) are now BUILT (2026-06-23). CLI `groups`/
    `set-group-level`/`add-group-rule`/`remove-group-rule`/`group-candidates`/`move`/`promote-group`. ✓ unit tests,
    full add→group→remove + throwaway-promote-then-teardown round-trips, screenshots of both new UI surfaces.

### 18.2 FILE MAP (everything touched)

| File | Role |
|---|---|
| `Scripts/codebase_tools/registries.py` | data layer: discover/schema/`FIELD_TYPES`/`field_schema`/vocab/`set_entry`/`build_dump`/`write_cache`. **+ capabilities + `identity_matches` (17.5d); add-policy (17.5f); `_load_prefs`/`set_pref` (17.5i); grouping engine `group_key`/`resolve_entry_group`/`set_group_config` + `move_entry` + `group_candidates` (guided rule ladder + match counts) + `promote_group` (data-driven group→registry) + CLI `prefs`/`set-pref`/`groups`/`set-group-level`/`add-group-rule`/`remove-group-rule`/`group-candidates`/`move`/`promote-group` (17.5i+k). Dump carries `prefs`, per-reg `group_config`, per-entry `group`.** |
| `INIDATA/registry_add_policy.json` (NEW, created on first edit) | per-registry per-field add policy overrides. Absent = built-in defaults. |
| `INIDATA/registry_prefs.json` (NEW) | favorites/hidden ordering — scope `_registries` + per-registry entry scopes (sec 17.5i). |
| `INIDATA/registry_groups.json` (NEW, created on first tune) | per-registry grouping config `{level, rules}` (sec 17.5k). Absent = type default. |
| `INIDATA/promoted_registries.json` (NEW, created on first promote) | data-driven promoted registries `{id:{json,template:"site",source,name}}` (sec 17.5k). Read by BOTH `add_voice_choice._register_promoted_registries()` (typed writes + reload) AND `web_opener._promoted_site_registry_paths()` (`<site>` grammar union). Absent = none. |
| `INIDATA/VoiceChoices/<promoted_id>.json` (NEW per promote) | a promoted group's standalone registry, site-shaped. Created empty by `promote_group`, then entries moved in. |
| `INIDATA/VoiceChoices/youtube_videos.json` (NEW) | the YouTube-videos TYPE registry (sec 17.5g); feeds `<site>` via `SITE_REGISTRY_PATHS`. |
| `Scripts/VoiceConfigManager/add_voice_choice.py` | the writer. `CHOICE_SETS` value_fields; google_doc/site/directory have optional `open_fn`; optional-trailing PADDING in `_handle_add`. **+ `youtube_video` set; `_register_promoted_registries()` clones `site` for each entry in `promoted_registries.json` so promoted registries get typed writes + the web_opener reload touch (sec 17.5k).** |
| `Helpers/RegistryEditorMenu.ahk` | the editor + smart add. Field-pick editor, `_RegAddEntry`, `AddSmartDestination`+`_DestConfirmContext`+`_AddLinkToContext`+`_AddIntoRegistry`, `_RegPickEnum`/`_RegResolveFieldValues`/`_RegFunctionVocab`. **+ capability layer (sec 17.5d): `_RegCaps`/`_RegChromeTabTitle`/`_RegIdentityMatches`/`_RegDupCheck`/`_RegFieldValueInArgs`/`_RegMaybeContextBridge`, `AddDocSmart`, prefill-vs-seed split, array `--replace-key` in `_RegWriteAndReopen`.** |
| `Helpers/ContextDestinations.ahk` (NEW) | the `destination` role: `ContextDestinationLookup`/`...DetectForeground`/`...AllTokens`. Included in MAINFUNCTIONS after ProgramRegistry. **+ `add_mode`/`defaults` on the block (sec 17.5e).** |
| `Helpers/DirectoryRegistry.ahk` | `DirRegLookup` reads `open_fn`; `OpenDirectoryByName` dispatches it. |
| `caster/rules/web_opener_commands.py` | doc/site pack+unpack+dispatch `open_fn` (4-field `|||`); `add site` → `AddSmartDestination`, `add doc` → `AddDocSmart` (`add doc legacy` REMOVED 2026-06-23, live-verified); **`SITE_REGISTRY_PATHS` (sites+youtube) + `_promoted_site_registry_paths()` unioned into `load_sites()` so promoted registries stay openable (sec 17.5g+k).** |
| `caster/rules/windows_commands.py` | `add directory` → `AddDirectorySmart` (quiet path-seed add, sec 17.5e); `add directory legacy` → `AddDirectoryByVoice` (safety net until keyboard-verified). |
| `INIDATA/Contexts/youtube.json` | `destination: {store:"registry:youtube_videos"}` (links migrated out, sec 17.5g). |
| `Helpers/RegistryEditorMenu.ahk` (grouping/prefs UI) | `_RegOrganizeEntries`/`_RegGroupHeader`/`_RegSortGroups`, `_RegPrefOf`/`_RegPartitionByPref`/`_RegMarkNode`/`_RegDivider`/`_RegPrefRowActions`/`_RegPrefSet`/`_RegFavHideNodes`, `_RegGroupingNodes` (now lists+removes custom rules)/`_RegSetGroupLevel`, `_RegMoveEntry`/`_RegPickTargetRegistry`. **+ guided rule creator `_RegGroupRuleCandidates`/`_RegSetGroupRule`/`_RegRemoveGroupRule` (per-entry `✦ Group rule from this…`); promote `_RegPromoteGroupNodes`/`_RegPromoteGroup` (registry `⬆ Promote group…`).** `RegPrefs` global (in `_RegLoadData`'s `global`!). `_RegStr` (safe Map stringify). |
| `INIDATA/Contexts/google_docs.json` (NEW) | `match: docs.google.com/document`, `destination: {store:"registry:google_docs"}`. |
| DELETED: `Scripts/codebase_tools/destinations.py` | superseded by context-driven detection. |

### 18.3 NEXT STEPS (priority order)

1. **Capability reroute — DONE for all three add types (sec 17.5d–e).** Dup-detection, alias/replace,
   context-bridge, and title-prefill are generic, schema-derived capabilities on the shared quiet add. `add site`
   → `AddSmartDestination`, `add doc` → `AddDocSmart`, and (2026-06-23) **`add directory` → `AddDirectorySmart`**:
   the directory analogue — path-seed via Explorer COM → terminal title → selected text (reuses the old
   `_VoiceConfigGet*Path` detectors), then `_DestAddQuiet("directories", path, …)`. Identity is `path` so the
   shared flow's path dup-detection + phrase prompt + typed write + edit-in-place all apply for free; program/snap
   stay BLANK on add and are edited in place (Jamie's quiet model). A soft program-key collision note covers the
   directory↔program `open <X>` ambiguity. **Decision:** the old "folder-CAPABLE program filter" was dropped — in
   the quiet model the program is edited in the editor, not picked at add time (filtering it is a future editor-enum
   nicety, not blocking). REMAINING under this head:
   - Jamie **live-verified `add site` + `add doc` (2026-06-23)**, so `AddGoogleDoc`'s voice wiring (`add doc legacy`)
     was REMOVED; `AddHardcodedSite` was already unwired. The AHK BODIES await a careful shared-helper trace before
     excision (the context-bridge helper is shared with the live add) → [[AHK_LEGACY_ADD_FLOW_REMOVAL]].
   - **Keyboard-verify `add directory`** (now `AddDirectorySmart`); `AddDirectoryByVoice` stays wired as
     `add directory legacy` until then, then its body joins the [[AHK_LEGACY_ADD_FLOW_REMOVAL]] pass.
2. **Grouping — all pieces DONE incl. auto-route (2026-06-23, sec 17.5k).** Guided-segment rule creator (`✦ Group
   rule from this…`), one-click promote (`⬆ Promote group → its own registry`), AND **promote auto-route** are
   built + verified. Auto-route: `promote_group` now records a membership `match` prefix in
   `promoted_registries.json`; `promoted_route(src, value)` returns the promoted child whose prefix the value falls
   under (longest wins); `_DestAddQuiet` calls `_RegPromotedRoute` at the top, so after "promote Amazon" a NEW
   amazon.com add files straight into the Amazon registry instead of Sites. CLI `promoted-route`. ✓ verified:
   promote plex.tv → a new plex URL routes to `plex_tv`, an amazon URL routes nowhere; throwaway torn down clean.
   REMAINING — keyboard-verify the three flows live (fire `✦ Group rule from this…`; promote a real group; confirm
   a subsequent matching add lands in the promoted registry).
3. **More destination contexts** — Instagram (needs a url match; today matches Messages by title), Reddit,
   etc. Each = a `destination` block on the context (`registry:<id>` or `links`).
4. **Sites-as-flat-registries — context-LINKS flow FULLY REMOVED (2026-06-23, Jamie approved).** Deleted:
   `_iter_context_links` (web_opener); the `_VoiceConfigAddLinkHere`/`AddSubContext`/`SiteCtxWrite`/`SiteCtxScript`
   cluster (VCM); `_AddLinkToContext` + `_DestAddLinkQuiet` (RegistryEditorMenu) with their `AddSmartDestination`
   / `_DestAddInteractive` branches rerouted to the Sites registry; the whole `SiteBrowserMenu.ahk` +
   `Scripts/VoiceConfigManager/site_contexts.py` + the `test_site_browser_menu.ahk` GUI test + its fixture; the
   "site browser / show sites / sites menu" voice command; and the `#Include`s (MAINFUNCTIONS + gui-test runner).
   Browse sites via "open registry" now. ✓ validate + 6-entry-point closure clean.
   **Also removed `AddDirectoryByVoice`** (live-verified `AddDirectorySmart`): the body + `add directory legacy`
   wiring; `OpenDirectoryConfigGui`'s "adddir" case rerouted through `MAINFUN.bat AddDirectorySmart` (it lives in
   the GUI subsystem, outside RemoteControl's curated closure — the closure checker caught this runtime-only break).
5. **Original five queued steps — TRIAGED 2026-06-23:**
   - **Per-context materializer — NOT BUILT (genuinely the next real subsystem; §15.7 phase 5).** CORRECTION
     2026-06-23: an earlier triage here wrongly called this "done" by conflating it with the GLOBAL materializer
     (`generic_materializer_commands.py`, done 2026-06-22 — that one materializes GLOBAL store rows live, no
     reboot). The PER-CONTEXT one was explicitly deferred (§15 "Context-kind rows skipped = later build"; §15.7
     "scope step blocked on per-context materializer"). It's why the Voice Command Editor has NO live global/context
     scope choice — a context-kind store row can't go live without it. Caster scopes per RULE, so this needs a
     materializer rule per distinct context (each reading its store slice with the right exe/title binding),
     pre-created so new contexts don't force a reboot. This is the most valuable unbuilt item, not obsolete.
   - **Dedup `ShowRecentFunctionsViewer` — a REFACTOR, not a deletion.** CORRECTION 2026-06-23: an earlier triage
     here wrongly called the Scripts file "un-wired / delete it." It is LIVE — `Helpers/ShowRecentFunctions.ahk`
     ("show fun" entry) spawns `Scripts/ShowRecentFunctionsViewer.ahk` as its own process via `Run(…scriptPath…)`
     (a VARIABLE, so a literal-string grep misses it). The actual task (§15 "Dedup debt"): the viewer still carries
     its OWN copies of the dispatch-read / paste-block logic that now lives in `Helpers/RecentFunctionsData.ahk` —
     refactor it to use that shared layer. Modest internal cleanup; the file stays.
   - **`OpenDirectoryConfigGui` → registry Miller — OPTIONAL consolidation.** Works today; would fold the
     directory/program config GUI into the Registry Editor. NOTE it already has a folder-capable program picker
     (`_DirCfg_PickFolderCapableProgram`) — the "folder-filter" deferred from `AddDirectorySmart` lives here.
   - **Retire `EXPLICIT_CHOICE_MAP` — LOW-VALUE maintenance reduction.** Still hand-maintained in
     `voice_commands_by_tag.py`; `derive_linkage` (registries.py) is the auto replacement but the two coexist fine.
   - **Cache mtime-check — SMALL resilience nicety.** No mtime guard today (cache regenerates on Stop hook + after
     writes); an mtime-vs-source check would auto-heal a stale cache. Low priority.

**NOT YET KEYBOARD-VERIFIED (built + headless-checked this session; need a live run):** quiet `add site`/`add doc`
with a real Chrome page (phrase prompt → write → open-on-entry); the add-policy/favorites/grouping row actions
firing live; `Move to registry…`. All components are unit/validate/screenshot-verified; only the live voice/modal
paths await Jamie at the keyboard.

### 18.4 GOTCHAS / verification notes (learned this session — don't relearn)

- **Headless AHK testing is BLOCKED for anything needing the full closure.** `#Include MAINFUNCTIONS.ahk`
  hangs (persistent always-on includes; the arg-gated dispatcher at MAINFUNCTIONS.ahk:384 doesn't help).
  Minimal-include also fails — cross-deps (e.g. `DetectContextChain`→`ChromeActiveAddress`) raise load-time
  "nonexistent function". So GUI/context AHK is verified via: `ahk.py validate` + `ahk_include_closure.py` +
  isolated Python logic tests + the mirror-of-proven-pattern argument + deterministic screenshots.
- **Synthetic-keystroke GUI driving is UNRELIABLE here** (Miller filter-vs-digit input handling; no-focus
  test-tagged windows resist `WinActivate`). For screenshots use the DETERMINISTIC right-pane preview:
  `py ~/.claude/helpers/ahk.py show OpenRegistryEditor --arg <regId> --title "Registry Editor" [--select N]`
  (an entry's preview shows its field rows). Don't burn time driving multi-step modal flows by keystroke.
- **The `|||` site/doc Choice value is now 4 fields** (`url|||title|||default_pos|||open_fn`). EVERY
  `_unpack_*_value` caller must unpack 4 — watch indentation in `replace_all` (a 4-space caller was missed
  once; `_doc_key_from_packed`).
- **`ChromeCurrentUrl()`** is the canonical URL reader (per the ahk skill). `_SiteBrowserWrite` (SiteBrowserMenu)
  shells `site_contexts.py add <tok> <phrase> <url> <title> <default_pos>` — the link writer.
- **The cache** (`~/.claude/context/registries_dump.json`) is regenerated on the Stop hook + after GUI writes;
  `registries.py cache` regenerates manually. The editor reads it at boot. **`capabilities` now rides in the
  dump — after changing `registry_capabilities`/`FIELD_TYPES`, run `registries.py cache` or the editor reads
  stale caps.**
- **`seed` vs `prefill` in `_AddIntoRegistry`** (sec 17.5d): `seed` values are AUTHORITATIVE (skip the field
  prompt — url/open_fn from detection); `prefill` values are DEFAULTS (still prompt — title from the tab). Don't
  conflate them or the title becomes uneditable / the url gets re-prompted.
- **Capability dup-detection runs on the FINAL field value**, read post-collection via `_RegFieldValueInArgs`
  (schema-position lookup) — so an edited url is what's checked, matching the bespoke "based on final URL".
- **AHK v2 is assume-LOCAL**: a function that ASSIGNS a global must name it in the `global` line, or the
  assignment silently makes a discarded local (the symptom: a feature reads an always-empty global). Bit the
  favorites work (`RegPrefs` missing from `_RegLoadData`'s `global`). When a new global won't "take", check the
  declaration first.
- **Registry-editor prefs/policy keys are CASE-SENSITIVE and must match the entry key exactly** (the row action
  passes the real node key, so live use is fine; hand-set `set-pref google_docs journal` ≠ entry `Journal`).
- **`String(aMap)` THROWS in AHK v2** ("Map has no method ToString") — and a throw inside a Miller node
  builder aborts the WHOLE level's build (caught as `Mcp/build_err`, the level renders empty). An object entry
  with a nested field (sites' `open` block) detonated the entire Sites list this way. Always stringify entry
  values through `_RegStr` (recursive), never raw `String()`. The log (`ahk_event.log` → `Mcp/build_err` with
  file:line) pinpoints these instantly — tail it FIRST.
- **Screenshot harness leaves windows open** without `--close`, and the next `show` re-captures the STALE
  window (looks like your change did nothing / landed at root). `ahk.py check` lists them; pass `--close`, or
  clear a stray window with **`ahk.py close "<title>"`** (graceful WM_CLOSE on that hwnd). **NEVER
  `Get-Process AutoHotkey64 | Stop-Process` — it kills Jamie's ~5 always-on scripts (her whole macro system)
  and she has to restart everything.** `ahk.py sweep` is only for #32770 error popups. (Full rule:
  `~/.claude/skills/ahk-functions/references/gui-testing.md`; deeper tooling: [[AHK_SCREENSHOT_CLOSE_TOOLING]].)
