---
tags: [design, settings, miller, architecture]
related: ["[[UNIFIED_MILLERS]]", "[[MILLER_LAYOUT_SYSTEM]]", "[[VOICE_COMMAND_SYSTEM]]", "[[CONTEXT_MANAGER]]", "[[TIMER_OVERLAY]]", "[[QUOTES_SYSTEM]]"]
status: built (v1) — 2026-07-29
updated: 2026-07-29
---

# Settings System — declare a tunable once, get storage + UI for free

**Status: BUILT and live.** Voice: **"open settings"** (or `settings <system>` to
land on one page). Two systems converted so far (Kindle, Meditation).

## The problem (Jamie 2026-07-29)

Every system's tunable constants were hardcoded somewhere: the Kindle highlight
colour was `"blue"` in four places, the meditation timer's 22 minutes was a
parameter default in four launchers. Two costs — changing a value meant a code
dive, and **exposing a value in a UI meant hand-writing a Miller node for it**.
The second is the one that actually blocked: *"it becomes really tedious to add
individual settings manually by hand."*

## The shape

A system **declares** its settings once, as data. Everything else derives:

```
       schema file  (one per system, code-authored)
              │
    ┌─────────┼──────────┐
    ▼         ▼          ▼
  storage    UI        readers
  defaults   Miller    Setting("kindle.highlight_color")     ← AHK
  + overrides node     settings.get("kindle.highlight_color") ← Python
  + validation (generated)
```

**Adding a tunable = one command. There is no UI code, ever.**

```bash
py Scripts/settings/settings.py declare kindle.highlight_color \
   --type enum --default blue --options pink=Pink orange=Orange yellow=Yellow blue=Blue \
   --label "Highlight colour" --detail "Colour applied when a grab highlights the page."
```

This is the third sibling of a pattern the codebase already proved twice — see
[[UNIFIED_MILLERS]] § "The control — one button that is a node":

| Control | Edits | File |
|---|---|---|
| `MillerPlacement` | positional pin/sink | `Helpers/Gui/MillerPlacement.ahk` |
| `MillerTags` | membership tags | `Helpers/Gui/MillerTags.ahk` |
| **`MillerSettings`** | **typed values** | `Helpers/Gui/MillerSettings.ahk` |

## The type system

Deliberately tiny — a small type set keeps the UI generator small.

| Type | Generated editor |
|---|---|
| `bool` | On / Off rows, ● marks current |
| `int` / `float` | "Set the value…" input + `−step` / `+step` rows; `min`/`max`/`unit` |
| `string` | free text — **the only free-text type** |
| `enum` | one row per option, ● (single) or ✓ (multi) |

`enum` carries the whole "options come from somewhere" story:

- **static** — `"options": [{value,label,symbol}, …]`
- **dynamic** — `"source": {"kind": …}`, resolved at open time
- **`"multi": true`** — value becomes a list, editor becomes a toggle list
  (exactly `MillerTags`' UX), per the standing rule that multi-select is toggles,
  never a comma-separated text box

### The "hardest part" was already built

Jamie flagged dynamic option lists as *"by far the most complicated part"* and
wanted them last. They shipped in v1 instead, because a dynamic list is **not a
new type** — it is a different `source` on the same `enum`, so the UI generator,
the validator and the store need **zero** new branches. And the list
infrastructure already existed: `registries.py` discovers and enumerates **45
registries** — the dictation value-lists behind `open <site>`, `go <directory>`,
`text <name>`. Resolving one is ~20 lines over infrastructure in daily use.

```
{"kind": "registry", "id": "sites"}     any of the 45 dictation lists
{"kind": "context"}                     every token in INIDATA/Contexts
{"kind": "function", "pattern": "…"}    AHK function names, optionally filtered
{"kind": "command", "argv": [...]}      escape hatch, one option per line / JSON
```

So "my default application, picked from the possible-applications list" is
`type: enum` + `source: {kind: registry, id: <that list>}`. Done.

## Storage

```
INIDATA/Settings/schema/<system>.json   the DECLARATION. Owns the DEFAULT.
                                        Human-readable, git-tracked; field order
                                        preserved on write so it stays reviewable.
INIDATA/Settings/values.json            ONLY Jamie's overrides, {system:{key:value}}.
```

**No generated cache file** (decided 2026-07-29). A reader parses `values.json`
(tiny) plus the ONE schema it asked about — two small parses, and the AHK JSON
parser was rewritten in June to be 24× faster. A flat resolved-cache would buy a
couple of milliseconds and cost a whole staleness bug class.

### Two editable levels

Jamie asked for both, not just the override:

- the **DEFAULT** lives in the schema, moved with `set-default` (UI: "Change the
  default ▸")
- the **OVERRIDE** lives in `values.json`, set with `set`; absent means "use the
  default", so **reset is just deleting the key**

Setting an override equal to the current default **deletes it** rather than
storing it, so "untouched" and "explicitly set to today's default" stay distinct
— otherwise a later default change would silently not reach her. `set-default`
drops an override that equals the new default, for the same reason.

**Hard rule:** the schema is the ONLY home for a default. Consuming code must
never keep a parallel fallback constant, or the two drift and the schema becomes
decorative. An undeclared key logs loudly rather than returning a silent `""`.

## The UI

`MillerSettingsNode(system)` returns one `⚙ Settings ▸` branch whose children are
that system's rows, each showing its current value, `•`-marked when overridden,
grouped by the optional `group` field. Focusing a row shows a **self-documenting
right pane** — the detail text, the reference, type, value, default, whether it is
overridden, its bounds, and where its options come from. A setting documents
itself; there is no separate doc to keep in sync.

**Three ways the row appears, all automatic:**
1. **`MlSystem` menus** — injected at the system root whenever a schema file
   matches the `system_id`. No line at all. Opt out with
   `MlSystem(..., Map("settings", false))`.
2. **Scaffolded menus** — `new_miller.py` emits the `MillerSettingsNode("<key>")`
   splice, which returns `""` until the first setting exists.
3. **The hub** — "open settings" lists every system by globbing the schema folder.

The hub is **entirely derived**: it holds no list of systems and no list of
settings. That is what earned it its place next to per-Miller mounts rather than
duplicating them (Jamie: worth building *"if things can be dynamically added to
this instead of having to be manually added"*).

## Files

| Piece | Path |
|---|---|
| Engine (CRUD, validation, option sources, `declare`) | `Scripts/settings/settings.py` |
| AHK reader — `Setting("<system>.<key>")` | `Helpers/SettingsStore.ahk` |
| Miller control | `Helpers/Gui/MillerSettings.ahk` |
| Auto-inject | `_MlSystemInjectSettings` in `Helpers/Gui/MillerSystem.ahk` |
| Hub ("open settings") | `Helpers/SettingsMenu.ahk` + `Scripts/SettingsViewer.ahk` |
| Store | `INIDATA/Settings/` |
| Drift check (Stop hook) | `Scripts/codebase_tools/settings_drift_check.py` |
| Tests | `Helpers/Tests/test_settings_store.ahk` (26 pure), `Helpers/Tests/Gui/test_settings_menu.ahk` (2 GUI, deep-validated) |

**Reads never shell Python.** A settings read is on the hot path and every MAINFUN
call is a fresh process, so a `py settings.py get` per read would add the ~140 ms
python-spawn floor to ordinary work. `SettingsStore.ahk` parses the JSON natively
and caches per process; option resolution is native too (a registry IS a
VoiceChoices json, contexts are a folder glob), because a Miller previews the
highlighted row EAGERLY and a per-preview subprocess would stutter the UI — the
gotcha `MillerTags` logged the hard way. Only the `command` source kind shells
out, and only when that row is opened. **Writes** go through the engine, so
validation and the default/override bookkeeping live in exactly one place.

## The drift check (why it exists)

The "schema is the only home for a default" rule is invisible at runtime, so the
Stop hook checks both directions, and they fail differently on purpose:

- **A read with no declaration → FAIL.** `Setting("kindle.nope")` returns the
  fallback and logs, so the feature quietly runs on nothing.
- **A declaration nothing reads → WARN.** It shows an editable value in "open
  settings" that does nothing — worse than a missing feature, because it actively
  misleads. Only a warning, because a key can legitimately be built at runtime.

It earned its keep immediately: it caught `meditation.warmup_minutes`, declared
during this build but never wired, and that declaration was withdrawn rather than
shipped (see below).

## Converted so far

| Setting | Was | Now |
|---|---|---|
| `kindle.highlight_color` | `"blue"` literal in `KindleGrabLaunch.ahk` ×2, `KindleGrab.ahk` ×4, plus a dead `KG_DEFAULT_COLOR` global | one declaration; `_KGDefaultColor()` resolves it |
| `meditation.default_minutes` | `minutes := 22` parameter default in 4 launchers | one declaration; each launcher takes `""` and resolves |
| `meditation.pack_default_minutes` | `fallback := 10` in `GetPackDefaultMinutes` | one declaration |

**AHK parameter defaults must be load-time constants**, so a converted function
takes `""` and resolves in its body. That shape is the norm for this system.

## Decisions taken (2026-07-29)

- One schema file **per system**, not one big file.
- Defaults and overrides in **separate** files; both editable from the UI.
- Dynamic option sources + multi-select **in v1** (they were nearly free).
- Auto-inject by matching `system_id`, rather than an explicit mount per Miller.
- **No** resolved cache.
- `Config.ahk` stays out — paths and exes are machine facts, not preferences.
- Settings and registries stay separate: a registry is a *list of things*, a
  setting is *one value* that may pick from one.
- Always-on scripts **re-read at point of use** (a long-lived script sees a change
  on its next process). Jamie: *"that is fine unless it bites us."* No reload
  marker unless it does.

## Not built (deliberately)

- **Per-context settings** (a different value per app/context). Out of scope for
  now; the schema is shaped so it could be added without migration. Jamie: *"we
  will eventually want it for very specific settings — build with that in mind."*
- **Voice commands that set a value directly** ("set highlight colour yellow").
  The options are already enumerable, which is exactly what a Caster `Choice`
  needs, so this is a natural v2. Deferred for the grammar-reload cost.
- **`meditation.warmup_minutes`** (the 2-minute settle before the first bell).
  Declared during the build, then **withdrawn** — it has TWO readers that must
  agree (`WARMUP_MINUTES` in `TimerOverlay.ahk`, a lean standalone overlay with
  no include closure, and `meditate_warmup_s` in `LockoutTimer/config.ini` read
  by `TimerAudio.py`), and converting only one side would replace a documented
  drift hazard with a worse undocumented one. It is a good future candidate —
  the payoff is killing that "keep these in sync" comment — but it needs both
  sides converted together and a real meditation session to verify.
