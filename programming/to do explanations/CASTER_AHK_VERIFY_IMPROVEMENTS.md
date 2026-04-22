# Caster/AHK Verify System — Next Improvements

Planned continuations of the verify infrastructure built on 2026-04-22.
None of these block each other; pick any one independently.

## Umbrella goal

Keep investing in testing and documentation for this tooling as gaps
surface. The reactive layer (verify hook + bridge + pytest) is in good
shape. The big remaining gap is **proactive help** — giving Claude the
context it needs to avoid mistakes in the first place, rather than
catching them after the fact.

---

## Where we are now (reference)

**Built 2026-04-22:**
- `~/.claude/helpers/caster_ahk_verify.py` — Stop-hook dispatcher. Runs
  ruff + JSON-parse + validate_rule + bridge + AHK /validate + pytest
  whenever Claude stops. Reads the session transcript to discover
  changed files.
- `~/.claude/helpers/caster_ahk_bridge.py` — orphan detector. Scans for
  `mainfun_action(...)` / `named_mainfun_action_with_args(...)` refs in
  Caster Python and `Name(args) {` / `Name() => expr` definitions in
  AHK. Two passes: Caster→AHK and AHK→AHK. Knows ~370 AHK builtins,
  excludes vendored dirs (UIA-v2-main, Archive, Compiler, Diagnostics).
- `~/.claude/helpers/tests/` — pytest suite (36 tests) covering bridge
  regex logic + add_voice_choice arg parsing + CHOICE_SETS schema.
- Stop hook wired in `~/.claude/settings.json`, invocation log at
  `~/.claude/helpers/caster_ahk_verify.log`.

Typical hook run: ~1 second. PASS is silent; WARN prints one line;
FAIL re-prompts Claude with the exact error.

---

## 1. AHK LSP integration

### Why

`AutoHotkey64.exe /validate` catches syntax errors and broken
`#Include` chains, but misses runtime-only warnings. The common failure
mode: rename a function or variable elsewhere, and footpedals.ahk
silently breaks — you only see `#Warn VarUnset` / "this function was
not declared" pop-ups when the code path actually executes. That's
weeks after the change landed.

### What the LSP catches that /validate doesn't

- Undefined locals used inside function bodies (not just load-time)
- Type mismatches at known-type call sites
- Unused assignments / unreachable code
- Some reference-to-missing-member cases
- Anything the thqby LSP flags as a diagnostic at file open

### Approach

Use [thqby/vscode-autohotkey2-lsp](https://github.com/thqby/vscode-autohotkey2-lsp).
It's designed as an LSP server for VS Code, but under the hood it's a
standalone binary speaking JSON-RPC over stdio. We can drive it from a
Python wrapper.

### Concrete steps

1. Install the LSP binary from the thqby repo. Either the VS Code
   extension (bundled) or the standalone exe — the repo README has
   both install paths.
2. Write `~/.claude/helpers/ahk_lsp_check.py` — a wrapper that:
   - Spawns the LSP process
   - Sends `initialize` / `initialized` / `textDocument/didOpen` for
     the changed file
   - Waits briefly, collects `textDocument/publishDiagnostics`
   - Prints structured errors + exits 0 if clean, 1 if any
     "error"-severity diagnostic
3. Add `check_ahk_lsp` to `caster_ahk_verify.py` — runs when any .ahk
   file changed. Initially gate behind a `CHECK_AHK_LSP=1` env var so
   it can be toggled off if too noisy.
4. Tune the severity threshold (block only on `error`, surface
   `warning` as WARN) after running on the existing tree to see what
   comes up.

### Effort

1–2 hours including the LSP protocol plumbing. Main time sink is the
JSON-RPC handshake and figuring out which diagnostic codes matter.

### Tradeoffs

- Adds a third-party dependency (the LSP binary, ~10MB).
- First run on the existing tree will likely surface a pile of
  pre-existing warnings, same story as ruff. Treat those as WARN not
  FAIL to avoid turning the hook red from day one.
- LSP process startup is ~500ms each time. Could keep it hot via a
  long-running socket server if that matters, but probably fine as
  one-shot for now.

---

## 2. Auto-generated codebase inventory

### Why

Biggest proactive-help gap today. Right now Claude doesn't know what
AHK helpers exist, what voice commands are already defined, what
choice sets are JSON-backed. It writes `_snake_case` again when
`_VoiceConfigSnakeCase` is already there, reinvents helpers, picks
wrong file locations. An inventory file lets Claude check *what
exists* before writing new code.

This directly addresses the anxiety about "Claude writing random
functions that shouldn't exist."

### What it contains

Auto-generated Markdown file at `~/.claude/context/codebase_inventory.md`:

- **AHK functions** grouped by file, each with first-comment-line as
  description. E.g.:
  ```
  ## Helpers/SpotifyFunctions.ahk
  - SpotSpit(slot) — open URL stored under that SpotLinks.ini slot
  - SpotGenreSave(genre) — save current Chrome URL into genre's INI section
  ```
- **Voice commands** grouped by rule file, each with its rdescript:
  ```
  ## spotify_global_commands.py
  - "spot <genre>" → SpotGenreGo(genre) | open random URL from genre pool
  - "spot swallow <genre>" → SpotGenreSave(genre) | save current URL to pool
  ```
- **JSON choice sets** with current entry counts:
  ```
  ## Hardcoded choice sets (JSON-backed)
  - hardcoded_spot_genres.json (15 entries) — {phrase: PascalCase_section}
  - hardcoded_spot_slots.json (6 entries) — {phrase: snake_slot_key}
  ```

### Token model: on-disk, read-on-demand

Critical — don't auto-load the file into system prompt every turn.
That'd burn ~4000 tokens per turn whether needed or not.

Instead:
- File exists on disk
- CLAUDE.md has a **conditional instruction**: "Before creating a new
  AHK helper, voice command, or choice set, read
  `~/.claude/context/codebase_inventory.md`."
- Claude reads it only when about to write something that could
  duplicate existing code
- Net effect: zero token cost on turns that don't need it, ~3000
  tokens when Claude DOES need it — replacing what would be 5–8
  Grep/Glob/Read calls (~10–15k tokens) to reconstruct the same info

### Interaction with existing skills

Clean separation of concerns:

- **Skills (caster-voice, ahk-functions)** — procedural knowledge.
  "How to add a voice command." Stable, changes rarely.
- **Inventory file** — declarative knowledge. "What functions exist
  right now." Auto-updates every turn.

Skills reference the inventory. The caster-voice skill's SKILL.md can
include: "Before defining a new helper, check `codebase_inventory.md`
for existing patterns."

### Generator design

Extend `caster_ahk_bridge.py` — it already parses everything needed
(AHK defs, Caster refs, choice sets). Add a `--generate-inventory`
flag that writes the Markdown file instead of running orphan checks.

Wire into `caster_ahk_verify.py` so it regenerates at end of every
hook run (silent, no status message — it's just a cache refresh).
That way it's always current without Jamie having to think about it.

### Concrete steps

1. Extend `caster_ahk_bridge.py` with `--generate-inventory` mode.
   Parse the same data; emit Markdown.
2. Add a "first comment line" extractor — for each AHK function,
   grab the comment block immediately preceding the def. Use as
   description; fall back to "(no description)" if none.
3. For Caster voice commands, use the rdescript= keyword text as the
   description. Parse the mapping dict.
4. For choice sets, enumerate the JSON sidecars under VoiceChoices/
   and count entries.
5. Create `~/.claude/context/` directory (parallel to helpers/).
6. Write the inventory file there.
7. Add regeneration call to `caster_ahk_verify.py` (one line at end).
8. Update user CLAUDE.md with the conditional instruction.
9. Update caster-voice and ahk-functions skills' SKILL.md to
   reference the inventory path.

### Effort

~2 hours. Most of that is the descriptive-comment extractor and
getting the Markdown formatting right.

### Gotchas

- Inventory gets regenerated at end of every turn. If the generator
  is slow, it adds latency. Keep it under 500ms. File I/O dominates;
  caching doesn't help much since Claude is likely touching different
  files each turn.
- Don't include vendored code (UIA-v2-main etc.) — same exclusion
  list the bridge already uses.
- Keep inventory size bounded. If it ever exceeds ~20KB, start
  truncating per-section (show first 40 lines, link to full files).

---

## 3. Template skills

### Distinction from existing skills

Current skills (caster-voice, ahk-functions) are **context-loader
skills**: SKILL.md describes patterns and conventions. They answer
*"how does this codebase work?"* Always passive, always reference.

Template skills would be **workflow-executor skills**: SKILL.md
describes a specific multi-step procedure, usually with fill-in-the-
blank templates. They answer *"do this N-step task consistently."*
Active, invoked on a specific request pattern.

### Which workflows deserve templates

Rule: only templatize workflows done 3+ times. Once is exploration,
thrice is a pattern.

High-frequency candidates from history:

- **`add-voice-command`** — done ~12 times. Edits Caster rule file
  + AHK helper file + voice commands doc + touches reload marker.
  Has a reliable 5–8 step shape.
- **`migrate-choice-dict-to-json`** — done 13 times. Matches the
  5-edit template already documented in
  `reference_voice_add_pattern.md`. Would formalize it as a skill.
- **`add-streamdeck-button`** — done several times. Edits
  ProfilesV3 manifest.json + involves image selection + position
  decisions.
- **`add-hardcoded-choice-set`** — tightly coupled to the JSON
  migration; probably bundle them.

### Independent skills vs nested inside caster-voice

**Recommendation: independent skills, each one references the
convention skills (caster-voice, ahk-functions).**

Why:
- **Discovery.** Claude Code matches skills on request pattern. An
  independent `/add-voice-command` fires when the user says "add a
  voice command for X" even if caster-voice isn't already in
  context.
- **Granularity.** One template per workflow = easier to update,
  version, disable independently.
- **Composition.** Templates reference the convention skills, so
  patterns stay centralized even though workflows are separate.

### Skill structure

```
~/.claude/skills/add-voice-command/
├── SKILL.md                      # Triggers on "add a voice command"
├── templates/
│   ├── rule_entry.py.tmpl        # {{verb}} → mainfun_action({{ahk_fn}})
│   ├── ahk_function.ahk.tmpl     # {{ahk_fn}}() { ... body ... }
│   └── voice_doc_entry.txt.tmpl
└── scripts/
    └── scaffold.py               # Optional — actual file editing
```

SKILL.md body is a checklist:
1. Parse request for {{verb}}, {{domain}}, {{ahk_fn}}, {{description}}.
2. Read `codebase_inventory.md` (once #2 above lands).
3. Check for {{verb}} collisions in existing voice commands.
4. Add mapping entry to `{{domain}}_commands.py` (template: rule_entry).
5. Add AHK function to `Helpers/{{domain}}Functions.ahk` (template:
   ahk_function).
6. Add entry to `voice commands current.txt`.
7. Touch reload marker (the add_voice_choice.py helper already does
   this for data-only adds; for brand-new commands, write a
   `# _reload_marker:` line).
8. Run `caster_ahk_verify.py` to confirm no orphans.

### Rollout order

Don't preemptively build all four. Build them **after** the next time
you do the workflow, so patterns are fresh:

1. First: `add-voice-command` (most frequent, clearest pattern).
2. When next JSON migration comes up: `migrate-choice-dict-to-json`.
3. When next SD button comes up: `add-streamdeck-button`.
4. Skip `add-hardcoded-choice-set` — merge into #2.

### Effort per skill

15–30 minutes each for simple templates, 1 hour if the skill
includes a `scaffold.py` helper that does multi-file atomic edits.

### Gotchas

- Templates go stale if conventions shift. Run verify on the result
  each time — if the hook catches something, either the template or
  the conventions have drifted.
- Don't over-templatize. If a workflow has too many decision points,
  the skill devolves into a choose-your-own-adventure that's worse
  than explaining each time.

---

## 4. AI-generated commit messages via Claude CLI

### Why

Current git history is "updates" × N. Effectively timestamps, not
content. Bad rollback targets, no project narrative. Every project
is a glorified backup with pseudo-versioning.

### Approach: Claude CLI with `--model haiku`

The Claude Code CLI supports headless one-shot mode:

```bash
claude -p "prompt..." --model haiku
```

- **Model choice: Haiku 4.5.** Commit messages are short structured
  summarization — Haiku nails them at ~1/5 Sonnet's cost, 1/25
  Opus's cost. Quality gap is imperceptible for this task. Using
  Haiku preserves your Opus/Sonnet plan quota for interactive work.
- **Latency:** ~1–3 seconds per commit. Fine for `git everything`.
- **Quality:** Excellent. It reads the actual staged diff and
  summarizes faithfully.

### Integration: modify `GitPushAll.ps1`

The flow is:

```powershell
# Stage everything per git_push_config.json (existing behavior)
git add ...

# Get the staged diff
$diff = git diff --staged

# Generate message via Haiku
$prompt = @"
Write a concise 1-line git commit message for this diff.
Rules:
- Max 72 characters
- Imperative mood ("Add X", "Fix Y", not "Added X", "Fixes Y")
- No trailing period
- No emoji, no prefixes like "feat:" or "fix:"
- Focus on the WHY when non-obvious, else the WHAT

Diff:
$diff
"@

$msg = claude -p $prompt --model haiku

# Commit and push (existing behavior)
git commit -m $msg
git push
```

### Concrete steps

1. Identify the current `GitPushAll.ps1` — lives in
   AutoHotkey/Scripts/ per memory `reference_git_everything.md`.
2. Verify `claude` CLI is on PATH. If not, resolve the full path
   and hardcode (or use the full install path).
3. Add the diff→Haiku→message flow between stage and commit.
4. Handle empty diff gracefully (no-op, don't call Claude).
5. Handle Haiku failure / timeout gracefully (fall back to
   "updates" so the workflow doesn't break).
6. Optional: cap diff size before sending (truncate to e.g. 50KB
   with a "... (truncated)" marker) so a huge refactor doesn't
   burn tokens.

### Effort

30 minutes including testing.

### Tradeoffs / gotchas

- Each commit costs a trivial amount of plan quota. At your commit
  frequency (maybe 5–20/day), negligible.
- Diff-to-message is inherently lossy. Messages will miss context
  ("I renamed this because of that separate issue we discussed").
  Accept that — they're still vastly better than "updates".
- Git allows commit message amendment after the fact: `git commit
  --amend -m "better message"`. You can always correct a bad one.
- If you want offline capability OR fully zero-cost, the fallback
  is a local LLM via Ollama. Negligible CPU wear (modern hardware
  handles brief inference bursts with no measurable lifetime
  impact), ~5GB disk + RAM. But for your use case, Haiku via API
  wins on quality + latency + setup simplicity.

---

## 5. Meta: keep extending testing and documentation

Ongoing investment, not a one-shot task:

- **As patterns stabilize, convert to pytest tests.** The bridge
  regex tests are the model — each new heuristic (local assigns,
  comment stripping, etc.) should get a test that pins current
  behavior so we notice if it regresses.
- **As workflows repeat, convert to template skills** (section 3).
  Don't templatize speculatively — wait until a pattern has fired
  3+ times.
- **As drift surfaces in production, extend the bridge.** Each new
  "I thought this was safe but it broke" incident is a candidate
  for a new check type. Examples of future checks worth
  considering:
  - Voice-commands-doc staleness (every rule file mapping key
    should appear in `voice commands current.txt`).
  - Stream Deck button orphan check (every
    `MAINFUN.bat FunctionName` in a ProfilesV3 manifest should
    resolve to a defined AHK function — same logic as the bridge,
    applied to a different file set).
  - Choice-set schema validation (every JSON sidecar should match
    the shape its CHOICE_SETS entry declares).
- **Surface pre-existing WARN clusters.** The hook currently
  tolerates style warnings in chrome_commands.py and
  windows_commands.py. At some point, dedicate a session to bulk-
  cleaning those to rdescript convention — then tighten the hook
  to block future drift.

---

## Quick reference: files that make up the system

| File | Purpose |
|---|---|
| `~/.claude/helpers/caster_ahk_verify.py` | Stop-hook dispatcher |
| `~/.claude/helpers/caster_ahk_bridge.py` | Caster↔AHK orphan detector |
| `~/.claude/helpers/tests/` | pytest suite |
| `~/.claude/helpers/caster_ahk_verify.log` | Invocation log (tail to confirm hook fires) |
| `~/.claude/settings.json` | Stop hook wiring |
| `~/.claude/skills/caster-voice/` | Caster convention skill |
| `~/.claude/skills/ahk-functions/` | AHK convention skill |

### Known non-blocking state (2026-04-22)

- Pre-existing rdescript style warnings in `chrome_commands.py` +
  `windows_commands.py` (surfaced as WARN, not FAIL)
- `chrome debug` and `save reboot` voice commands intentionally
  commented out (AHK side missing or disabled per machine
  constraints — see in-file comments for re-enable conditions)
