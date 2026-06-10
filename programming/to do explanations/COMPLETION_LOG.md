# Completion Log

A record of everything Jamie does in a day — captured by voice (`log <thing>`),
from a control-center GUI (`add log`), from a daily checklist (`log daily`), and
automatically from other systems (lockout timers, the to-do list). Designed so
the data is **future-proof for analytics** ("every day I meditated last year",
"meditates per month", "how long since I watered the plants") while the readable
view stays a colored, tagged markdown file in the vault.

Related: [[NEW_TAB_PAGE]] (recurring-task reset-on-check panel consumes this),
[[TIMER_OVERLAY]] (lockout timers auto-log here), [[Personal]].

---

## Status: built on top of an existing system

This is **not** from scratch. `Helpers/ListManager.ahk` already runs a
voice-driven multi-list system, and "completion" is already one of its lists:

- `add completion <thing>` already writes a dated, tagged line into
  `ObsidianVault/Lists/CompletionLog.md` (Obsidian colors the tags). **Being
  deprecated** — folded into the `log` namespace (see below).
- To-do → completion linking already works: `update <list>` marks a to-do done
  and, if `link_completion_on_done`, prompts to log it to completion with its
  tags. Live for `todo atlantic` / `todo house` / `todo macros`.
- Tags exist (`INIDATA/VoiceChoices/list_tags.json`, flat phrase→`#tag`).
- All GUIs already use the template system (Picker / Form / SingleField+Skip).

What this project **adds**: pre-made templates with default fields, a structured
JSONL source of truth, a multi-parent tag taxonomy, the `log` voice namespace
(`log <thing>` / `add log` / `open log` / `log daily`), the friends sub-flow, a
one-line `LogCompletion()` callable from anywhere, and the analytics layer.

---

## Architecture — structured truth + readable view (Path B)

Two artifacts, one generated from the other so they can never drift:

1. **Source of truth — `INIDATA/CompletionLog/log.jsonl`** (append-only, one JSON
   object per line). Every query, the recurring-task timer, and all analytics
   read this. Never edited in place.
2. **Readable view — `ObsidianVault/Lists/CompletionLog.md`** (the existing dated
   markdown file). Colored by tag in Obsidian. Regenerable from the JSONL.

Every log action does **both**: append a structured record to the JSONL *and*
append the readable markdown line. Both writes live in `clog.py` (Python owns
all JSON because AHK has no JSON parser); the markdown writer matches the
`## YYYY-MM-DD` / `1. - [x] thing #tag` format `ListManager._ListAppendDated`
produces. AHK (`CompletionLogFunctions.ahk`) and voice are thin shells that call
`clog.py`. The ~300 ms Python start is negligible for a voice command, and it
keeps add / query / render / css in one place.

### The entry schema (one JSONL line)

```json
{"ts":"2026-06-04T14:32:00-04:00","date":"2026-06-04","template":"meditate",
 "name":"Meditate","tags":["mindfulness","health"],"fields":{"duration_min":22},
 "note":"","source":"voice"}
```

Future-proofing decisions baked in:

- **`ts`** — full ISO-8601 with timezone. Day/week/month/year all derivable; time
  of day never lost. **`date`** denormalized for cheap grouping.
- **`fields`** — typed, **units in the key** (`duration_min`, `hours`). Self-
  documenting forever.
- **`tags`** — **leaf tags only**. Parents are expanded at query time from the
  taxonomy, so re-parenting a tag next year applies retroactively to all history.
- **`template`** — stable id, separate from the display **`name`** (rename the
  display without breaking history).
- **`source`** — `voice` | `ui` | `daily` | `lockout` | `todo`. Distinguishes
  auto-logged from manual.

### Unlogged vs not-done (Jamie's distinction)

Three states must be distinguishable in analytics:

| Situation | Meaning | How it's represented |
|---|---|---|
| Day has **zero** records | **Unlogged** — unknown, don't count it | absence of any line with that `date` |
| Day has records but no stretch | stretch **not done** (ambiguous-leaning) | no stretch line that day |
| `log daily` explicitly answered "no" to stretch | stretch **definitively not done** | record `{"template":"stretches","daily_check":"not_done","source":"daily",...}` |

Adherence math ("stretched 18 of 30 days") uses **only logged days** as the
denominator — unlogged days are excluded, not counted as misses. The rendered
view marks zero-entry days explicitly as `unlogged` so a gap reads as "no data",
not "did nothing".

---

## Config files (all in `INIDATA/VoiceChoices/`)

### `completion_templates.json` — the pre-mades

Rich config (not a flat Choice map). The Caster loader derives the flat
`{phrase: key}` Choice from it.

```json
{
  "meditate":  {"name":"Meditate",  "tags":["mindfulness","health"],
                "fields":[{"key":"duration_min","type":"duration","default":22,"prompt":"Minutes"}],
                "daily":false},
  "swimming":  {"name":"Swimming",  "tags":["exercise","physical_therapy","outside","health"]},
  "atlantic":  {"name":"Worked on Atlantic Logistics", "tags":["atlantic_logistics"],
                "fields":[{"key":"hours","type":"number","prompt":"Hours"},
                          {"key":"what","type":"text","prompt":"What I did"}]},
  "stretches": {"name":"Stretches", "tags":["physical_therapy","health"], "daily":true},
  "spanish_flashcards": {"name":"Spanish flashcards","tags":["spanish","learning"],"daily":true}
}
```

### `completion_taxonomy.json` — multi-parent tag DAG + colors

Obsidian's nested tags (`#health/exercise`) only do single-parent trees, so they
can't express "Atlantic belongs to programming **and** work". Hence a real DAG:

```json
{
  "health":             {"color":"#3aa655"},
  "work":               {"color":"#c77d2a"},
  "programming":        {"parents":["work"], "color":"#5b8def"},
  "atlantic_logistics": {"parents":["programming","work"]},
  "exercise":           {"parents":["health"]},
  "physical_therapy":   {"parents":["health"]},
  "mindfulness":        {"parents":["health"]}
}
```

Colors drive a generated Obsidian CSS snippet (`.obsidian/snippets/`) so the
readable view is colored per the taxonomy — editable/extendable from `add log`.

### Daily checklist

Daily items are just templates with `"daily": true`. `log daily` walks them in
order. Seeded: Spanish flashcards, Spanish videos, stretches, reading,
housecleaning, outside.

---

## Voice commands — the `log` namespace

| Phrase | Action |
|---|---|
| `log <template>` | Quick-log a pre-made, **instantly with its defaults** (no prompt). `<template>` is a **Choice extra** (fixed grammar → best Dragon recognition), built from `completion_templates.json` by a loader called **inline in `extras`** (the established JSON-sidecar pattern). |
| `log form <template>` | Pop the fill-in **Form** (e.g. meditate minutes, Atlantic hours + what) prefilled with defaults, then log. Templates with no fields just log with defaults. |
| `add log [<textnv>]` | Control-center GUI: type a freeform thing, or manage templates/tags. |
| `open log` | Open `CompletionLog.md` in Obsidian. |
| `log daily` | Sequential binary checklist (below). |

**Deprecated:** `add completion` / `add completion <textnv>` (removed from
`list_commands.py`). The to-do→completion link continues to work and writes
through the new structured path.

**Trade-off of the fixed grammar:** adding a template from the `add log` GUI is
not sayable until Caster reloads (automatic via the content-delta watcher, or
`reboot caster`). Accepted: recognition quality wins over instant availability.

### `log daily` — sequential binary checklist

A chain of **Confirm** modals, one per daily template:

- `1` / PgUp = did it → logs a normal completion entry (`source:"daily"`).
- `0` / PgDn = didn't → records `daily_check:"not_done"`, advances.
- Esc / End = abort the remaining items.
- Items already logged today are **pre-skipped** (not re-asked).
- Ends with a one-line summary tooltip (e.g. "logged 4, skipped 2").

---

## GUIs — every screen maps to an existing template

No hand-rolled GUIs (the codebase has zero one-offs left).

| Screen | Template |
|---|---|
| 0–99 template/category list (type # or letters) | **Picker** `_PersistentLoopPickGui` + `numpad_dispatch` (`N.M` row.action) |
| Meditate-minutes / Atlantic hours+what | **Form** `_MultiFieldInputGui` |
| Friends → pick-or-add activity (audition + curate a growing list, prompt to save new) | **Picker, two-pane music-picker mode** (`right_pane_on_enter`, `filter_alpha_autofocus`) |
| `add log` freeform type-a-thing | **SingleField+Skip** |
| To-do "log this?" / `log daily` binary | **Confirm** |

Numpad convention is the documented standard: `0`=cancel, `1`=default, `N.M`=
row.action. Not reinvented.

---

## Friends sub-flow

A friend (e.g. `cedar`) is a template tagged `friends` whose field is a
`pick_or_add` list of common activities. `log cedar` (or picking Cedar in
`add log`) opens the two-pane Picker of common activities; pick one or type a new
one. A genuinely new activity prompts to persist it to that friend's list (the
sound-library curation pattern).

---

## The one-line API — `LogCompletion()`

A single AHK function (new `Helpers/CompletionLogFunctions.ahk`) that any
caller fires with one line:

```ahk
LogCompletion("meditate")                       ; defaults
LogCompletion("meditate", Map("duration_min", 30), "source", "lockout")
```

It resolves the template, expands tags, writes the JSONL line, and appends the
markdown view line. **Lockout hook:** the meditate lockout end
(`Helpers/LockoutTimerFunctions.ahk`) calls `LogCompletion("meditate", ...,
"source","lockout")` — one line.

---

## Data interface for other apps (Python, `AutoHotkey/Scripts/completion_log/`)

This is a **consumable data layer, not a UI** — the new-tab page and any other
app plug into it later ([[NEW_TAB_PAGE]]). The rawest interface is `log.jsonl`
itself (any language can read it line-by-line). On top, `clog.py` exposes
**JSON-emitting** query commands so consumers never parse markdown:

- `last <name>` → `{"name","last_ts","days_since"}` (name = template key OR tag).
  Feeds the new-tab recurring-task "time since" counter; logging that template
  resets `days_since` to 0.
- `stats <name>` → `{"total","days_done","first","last","by_month",
  "not_done_days","adherence_denom"}`. `--text` prints a human summary instead.
  Adherence uses logged days only (unlogged days excluded). Tag names match via
  parent-expansion (e.g. `stats work` includes Atlantic).
- `export [--template|--tag|--source|--since|--until|--include-not-done]` →
  JSON **array** of matching raw records (adds `tags_expanded` when filtering by
  tag). The general "grab the data from any app" query.
- `streak <name>` → `{"current","longest"}` consecutive-day streaks (a
  not-yet-logged today doesn't break `current`). Name = template key OR tag.
- `unlogged-days [--since|--until]` → JSON list of days you logged **literally
  nothing** (distinct from per-item `not_done`). Defaults to first-log..today.
- `dashboard [--days N]` → **the one-shot new-tab payload.** Hit it once, render
  everything: `{generated, today, totals, unlogged_recent (last N days), daily[]
  (per daily template: last/days_since/current_streak/longest_streak/total/
  not_done_days), tags[] (leaf+expanded parents, by volume)}`. The granular
  commands above stay for targeted widgets.
- `render` — regenerate `CompletionLog.md` from the JSONL. **Each day grouped
  into broad categories** (Reading · Programming · Work · Health · Learning ·
  Friends · Other — ordered, first-match via `_display_category`), prefixed by a
  **`Dailies X/N`** meter (X = daily templates done that day, N = total daily
  templates). Hidden marker tags drive the CSS: `#hl-<key>` (pop: `finished`,
  `shrooms`), `#log-muted` (routine noise), `#daily-meter`. `cssclasses:
  completion-log` frontmatter scopes the styling to this note. `add`/`addfree`
  full-re-render so grouping + the meter stay correct.
- `daily-status` / `set-dailies --keys k1,k2,…` — read/replace which templates
  are daily (the `edit dailies` voice GUI's backend; re-renders after).
- `gen-css` — regenerate the snippet, all scoped to `.completion-log`: tag-pill
  colors (global), hide markers, **kill the done-task strikethrough/fade**,
  **compact** spacing (tight headings/lists), the **Dailies meter** badge,
  **dim** muted lines, **POP** highlighted lines (`:has()`, Reading view). Colors
  + opacity: `INIDATA/CompletionLog/render_styles.json`. Enable once: Appearance →
  CSS snippets → `completion_tags` (set in `appearance.json`; reload with Ctrl+R).

Contract for consumers: read `log.jsonl` directly for streaming/raw access, or
shell `clog.py export …`/`stats …`/`last …` for filtered JSON. Field keys are
stable and units live in the key name (`duration_min`, `hours`). The frontend
itself is out of scope here — built later on [[NEW_TAB_PAGE]].

---

## Integrations

- **To-do list** (existing): `update <list>` → mark done → prompt → logs through
  the structured path. As the to-do system matures, the "prompt to add to
  completion log on check-off" extends naturally.
- **New-tab recurring tasks** (deferred — [[NEW_TAB_PAGE]] "reset-on-check"):
  the widget reads `last <template>` to show "time since last done"; logging that
  template resets the counter to zero. **Schema is ready now**; wiring waits for
  the widget to exist.
- **New-tab analytics dashboard** (data layer DONE, frontend deferred —
  [[NEW_TAB_PAGE]]): the page shells `clog dashboard` once and renders
  streaks / adherence / time-since / tag rollups / unlogged-day markers from the
  single JSON. The frontend is intentionally not built yet; this is the contract
  it plugs into.

---

## File placement

| Thing | Path |
|---|---|
| Structured truth | `AutoHotkey/INIDATA/CompletionLog/log.jsonl` |
| Templates config | `AutoHotkey/INIDATA/VoiceChoices/completion_templates.json` |
| Taxonomy config | `AutoHotkey/INIDATA/VoiceChoices/completion_taxonomy.json` |
| Readable view | `ObsidianVault/Lists/CompletionLog.md` |
| Tag-color snippet | `ObsidianVault/.obsidian/snippets/completion_tags.css` (generated) |
| AHK functions | `AutoHotkey/Helpers/CompletionLogFunctions.ahk` |
| Voice rule | `caster/rules/log_commands.py` (+ loader) |
| Analytics / render | `AutoHotkey/Scripts/completion_log/*.py` |

---

## Phased build plan

1. ✅ **Foundation** — configs seeded; `CompletionLogFunctions.ahk` with
   `LogCompletion()` (shells clog.py: JSONL + markdown view); `log_commands.py`
   with `log <template>` Choice + loader; `open log`. Enabled, verified.
2. ✅ **Fields** — `log form <template>` pops the Form (clog `formspec` +
   `LogCompletionForm`). Screenshot-verified.
3. ✅ **`log daily`** — Confirm chain (`daily-pending` + `LogDaily`), pre-skip
   today, not-done records (JSONL-only), summary. Screenshot-verified.
6. ✅ **Lockout hook** — `TimerOverlay.ahk` shells clog.py on meditate
   completion (`--source lockout`). Wired + validated; live-fires next real
   meditation.
7. ✅ **Data interface** — clog.py `last` / `stats` / `export` (JSON-first for
   the new-tab page & other apps) + `render` + `gen-css`. Verified.
8. ✅ **Deprecated** `add completion`; to-do→completion link now writes through
   `LogFreeform` (structured). Curated-closure include added.
4. ✅ **`add log` control center** — Picker-first (`AddLog`): all templates show
   numbered immediately; pick by number, type to filter, or type an unmatched
   thing → freeform entry + optional tag pick (PgDn skips). A "+ New template"
   row collects name/phrase/tags and writes via clog `add-template` (sayable
   after caster reload). Reopens after each add. Screenshot-verified.
5. ✅ **Friends** — dedicated `log <friend>` (`LogFriend`): activity Picker
   (shared `_shared` list + the friend's own additions); pick or type a new one;
   a brand-new activity prompts to save to that friend's list. Logged as
   "Name: activity" tagged `#friends`. Config: `completion_friends.json`.
   Screenshot-verified.
9. ⬜ **Deferred** — new-tab recurring-task "time since" + reset-on-check
   (consumes clog.py `last`).

## v2 behaviors (added)

**Smart prompting.** A bare `log <template>` prompts only for fields that have
**no default** (and for all `pick` fields); fields with defaults pre-fill; all-
defaulted or field-less templates log instantly. So `log meditate` is instant
(default 22), `log shrooms` always asks grams (no default), `log code` prompts
for "what", `log reading` pops its type picker. `log form <template>` force-
prompts every field (e.g. to change meditate's minutes).

**Inline fill.** `log <template> <spoken text>` fills the template's `primary`
field from speech and logs immediately — `log code completion log` → programming
with what="completion log". If the primary field is a pick+as_tag (reading),
the spoken word becomes the tag (`log reading scifi` → `#scifi`).

**Aliases.** A template can list `aliases` (extra spoken phrases) — `code` →
programming. The Caster loader emits each alias into the `log <template>` Choice.

**Pick fields (the "sub add-log").** A field of `type: "pick"` with an
`options` list pops a mini-picker (theory/spanish/scifi/…); pick a number or
type a new one (which prompts to save to that template's list, like friends).
`as_tag: true` makes the picked value a tag on the entry; otherwise it's a field
value. Cancelling a pick aborts the whole log (no stray entry).

**`edit log`.** One smooth picker of recent entries (last 14 days, newest first),
numpad-driven and **no confirmations**: **Enter / N = edit** (text + tags form),
**N.2 = delete instantly**, **0 / Esc = done**. The picker **stays open and
refreshes** (`catalog_refresh_fn`) so several can be fixed fast — no action
sub-menu, no delete prompt. Both rewrite the JSONL and re-render the view.
Records carry a stable `id`. Also reachable from `add log` via the "= Edit
recent…" row. clog: `recent` / `delete <id>` / `edit <id>`.

> clog forces UTF-8 on stdout/stderr so em-dashes + accented titles round-trip
> cleanly to AHK (fixed the `Reading � Lorde` mojibake in the editor list).

### Template schema v2 keys
- `aliases`: [str] — extra spoken phrases.
- field `primary`: bool — the one filled by inline `log <template> <text>`.
- field with **no `default`** → always prompts (e.g. shrooms grams).
- field `type: "pick"` + `options: [...]` + `as_tag`: bool — option picker.
- `add-template-option <tmpl> --field <k> --option <v>` grows a pick list.

## Book / reading integration (added)

`log reading` is book-aware. The reading template's field is `type: "book"`,
which opens a **two-pane, music-picker-style hub** (`LogReading`):

- **LEFT pane** — **current reads on top**, then the genre types, then
  **"+ Add a book"** and **"= Merge book tags"**.
- **RIGHT pane** — the options for whatever LEFT row is selected, re-rendered as
  you arrow through the list (`right_pane_on_select`). For a **book**: Log it /
  Mark finished / Edit tags / Remove. For a **type**: Log this type. For the
  add/merge rows: that single action. Press **→** to focus the right pane, arrow
  to an option, Enter to run it; or pick it by its cross-pane number; or use the
  left numpad directly (below). Wired exactly like `ModesDashboard`.

**Source of truth = the existing book catalog** (`E:\Media\Books\<Author>\
library.json`, managed by `kindle_import.py`). "Currently reading" is a
`status: "reading"` field written onto the book; finishing adds `status:
"finished"`, `rating`, `review`, `finished_date`. All survive `kindle_import`
re-scans (it only refreshes scan fields + keeps `src:"manual"` tags) — verified.

The left numpad still drives every action in one keystroke via the **`N.M`
row.action** syntax (`numpad_dispatch`):

- **Enter / N** → log it. A book logs `Reading — <title>` with the book's catalog
  tags (+ `learning`); a type logs `reading #type`.
- **N.2** → mark finished (score /10 + review, saved to the library; logs a
  `Finished: <title>` entry).
- **N.3** → edit tags (hierarchical tag picker, below; saved to the library).
- **N.4** → remove from current reads.
- **N.5** → set the book's **spoken name** for `log reading <name>` (pre-fills the
  current; blank reverts to the author surname). Also a right-pane option.
- **+ Add a book** → *Search my library* (substring over every `library.json`)
  or *Add from scratch* (creates a catalog entry, with an optional spoken-name
  field); marks it reading.

The **hub stays open** after every action (log several in a row); **0 / Esc /
End** finishes. `catalog_refresh_fn` rebuilds the current-reads list after each
book mutation, so finishing/removing a book updates the list live without a
reopen. (Typing a one-off title that isn't in the library + Enter logs it and
closes.)

#### `log read` — auto-detect the book open in Kindle and log it

Say **"log read"** with no argument while Kindle for PC is the active window and it
figures out the book, logs a reading session, and (if needed) catalogs it first.
`LogRead()` grabs the Kindle window title and runs **`clog book-resolve-kindle --raw`**,
which strips the `Jamie's Kindle for PC N - ` chrome and matches against the catalog
in a cascade (slug-normalized via `_slug`, so messy filenames and clean titles compare
equal):

1. **ISBN** embedded in the title (Anna's Archive rich filenames).
2. a previously-stored **`kindle_title`** slug (the *learned link* — see below).
3. **de-slugged title** vs catalog titles (`mindfulness-in-plain-english` → matches
   *Mindfulness in Plain English*).

- **MATCH** → flips the book to `reading`, **stores the raw slug as `kindle_title`**
  (so it's an instant exact match next time, however garbled the filename), and logs
  via `LogReadingBook(id)`.
- **MISS** → `book-resolve-kindle` runs the normal online lookup (OpenLibrary→Google,
  by the de-slugged title/author) and returns best-effort title/author/isbn/tags;
  `LogRead` shows a **prefilled form to fill what's missing**, then `book-add --status
  reading --kindle-title <slug>` + logs. The de-slugged title (recognizable) is kept as
  the form default rather than a fuzzy online title, since author-mashed slugs can match
  the wrong book online — the form is the safety net.
- **`add read`** uses the same Kindle path as a fallback when no browser book is detected
  (so you can stand in Kindle and say "add read" too); stores `kindle_title` on add.
- Why store the raw string: all of Jamie's books are sideloaded with wildly different
  title formats (often a bare slug), so the only reliable long-term key is the exact
  window-title string — confirm once, match forever. clog: `book-resolve-kindle --raw`,
  `book-set-kindle-title <id> --raw`, `book-add --kindle-title`.

#### `log read <name>` / `log <name>` — log a current book by spoken name

Every **currently-reading** book has a **spoken name** (a book's custom
`log_name`, else the **author's surname**), so you can say **"log read lorde"**
and it logs that book *with its catalog tags* — no picker. (Renamed 2026-06 from
"log reading" → "log read"; "log reading" now cleanly means the daily *reading*
template.) Pipeline:

- `clog._sync_reading_choices()` regenerates `INIDATA/VoiceChoices/reading_books.json`
  (`{spoken_name: {id, title}}`, surname default, deduped on collision) **and bumps
  the `log_commands.py` reload marker** every time a book's reading status changes
  (`book-set-status` / `book-add`). Caster's content-delta watcher reloads the rule
  → the new book is sayable within seconds, no manual reboot.
- `log_commands.py` `load_reading_books()` reads that sidecar into the
  `Choice("reading_book", …)` (a hardcoded grammar — best recognition). Empty file
  → one harmless `"nothing"` placeholder so the rule still compiles.
- `"log read <reading_book>"` → `LogReadingBook(id)`, which pulls fresh
  title+tags via `clog book-get <id>` and logs a `Reading — <title>` entry.
- The bare `log <reading_book>` shorthand (no "read") also works.
- Set/clear a custom spoken name with `clog book-set-log-name <id> --name <name>`
  (the add-from-scratch form has a "Spoken name" field; blank = surname default).

#### To-read list (`add read` / `open read`) — Phase 1

To-read books live in the **same `library.json`** with **`status: "to_read"`**, a
**`recommended_by`** field, and genre/mood as ordinary **book tags** — so querying,
tagging, and merging all reuse the existing machinery.

- **`add read`** (`AddRead`) — if the active browser tab is a known book site, the
  title (+author) auto-detect; else you type them. Either way it does an **online
  lookup** (OpenLibrary primary, Google Books fallback) by title/author → ISBN +
  genre subjects, normalized through `canon_book_tag`, pre-filled for you to
  edit + add a recommender. Saved with `status: to_read`.
  - Sites config: `INIDATA/book_sites.json` (Libby · Goodreads · StoryGraph). Each
    entry is a `title_regex` matched against the browser tab title by `clog
    detect-book`; add a site by appending an entry. (JSON parsing stays in Python.)
- **`open read`** (`ToRead`) — two-pane hub (like the reading hub): left = matching
  books **grouped under umbrella headers** (`book-query --group` emits `__hdr:<umbrella>`
  section-label rows, which the picker renders non-selectable), right = per-book
  options (Start reading · Edit tags · Set recommender · Remove). The filter box
  narrows by any column; stays open + refreshes after each action. A `= Open reading
  list…` row cross-navigates to the reading hub (and vice-versa). (`to read from
  <friend>` was removed — filter by recommender in the hub instead.)
- clog backends: `book-add --status to_read --recommender --isbn` · `book-query
  --status --tags(AND, EXPANDED) --recommender [--group]` · `book-lookup --title
  --author` · `detect-book --title` · `recommenders` · `book-set-recommender` ·
  `book-remove`.

**Phase 2:** consolidate the ~500+ scattered to-read titles in
`TEXTDOCS/BOOKS AND MEDIA/TOCONSUMEBOOKS*.txt` etc. into this system — file-by-file,
parsed into a reviewable staging TSV → `import-staged` (dry-run default, `--commit`
to write; dedupes vs the library + within the file). **Done 2026-06:** 111 books
from `TOCONSUMEBOOKS.txt` (`_STAGED_to_read.tsv`) + 21 from
`TOCONSUMEBOOKSGENDER.txt`/`TOCONSUMEBOOKSRACE.txt` (`_STAGED_gender_race.tsv`).
Library 16 → 148. Remaining: `COSMERE NOTES.txt`, `TOLKIENREADINGORDER.txt`,
`REDREADINGCIRCLE.txt`, the `MEDIACONTENT…` book section.

> **This whole book system is being generalized into a unified media catalog**
> (movies, music, podcasts, etc. — books become "just another type"). Design:
> [MEDIA_SYSTEM.md](MEDIA_SYSTEM.md).

### Book tags — unified vocabulary + a derived umbrella hierarchy

Book tags are a **separate system from the activity taxonomy** (so tagging a book
never shows activity tags like `atlantic_logistics`). Two layers, both in the
single shared file `INIDATA/book_tag_aliases.json`:

1. **Normalization** (`aliases` + `stopwords`) — **shared by `kindle_import.py`
   (auto-import) and `clog.py` (manual tagging)** so synonyms collapse identically
   everywhere — e.g. `queerness`/`lgbt` → `queer`, `lesbian stories` → `lesbian`,
   `scifi` → `science fiction`, `memoir`/`biography` → `memoir`. `kindle_import`
   loads it (hardcoded defaults as fallback so import never breaks).
2. **Hierarchy** (`parents` + `umbrellas`) — a **DERIVED, multi-parent** rollup.
   Books store **only their leaf tag** (`anarchism`); `clog.expand_book_tags()`
   computes umbrellas (`theory`) transitively at query/render time. So editing the
   taxonomy **reflows every book with no re-import**, and a tag can sit under
   several umbrellas (`imperialism` → theory + history; `race` → identity + history).
   12 umbrellas: theory · identity · history · philosophy · science · health ·
   psychology · spirituality · fiction · memoir · climate · art.

Effects:
- **Query rollup:** `book-query --tags theory` matches every anarchism/marxism/
  imperialism/… book even though none carry a literal `theory` tag (AND-join is
  over the *expanded* tag set).
- **Grouped browse:** `--group` files each book under its **primary umbrella**
  (first in `umbrellas` order among its expanded tags; `Other` if none) and emits
  a header row per group — drives the `open read` hub's section view.
- **Orphan check:** `book-tags-orphans` lists leaf tags with no umbrella (run after
  each import batch). Format/geographic facets (`nonfiction`, `spanish`, `korea`)
  are intentionally umbrella-less.
- The **book tag picker** (`_PickBookTags`, reading `N.3`) lists the canonical leaf
  vocabulary with counts; new tags normalized on save.
- **Merge** (`= Merge book tags` in `log reading`): fold one tag into another
  library-wide — adds an alias *and* rewrites existing tags.
- clog: `book-tags` (vocab+counts) · `book-tag-alias <from> --to <to>` ·
  `book-tags-normalize` (canon pass) · `book-tags-orphans` · `expand_book_tags()`.

### Activity-tag hierarchical picker (the definitive activity tag list)

`completion_taxonomy.json` is the definitive tag list with parent links
(feminism → theory, fantasy/scifi → fiction, theory/fiction/nonfiction →
learning). Editing a book's tags (reading `N.3`) — and the freeform tag pick —
opens a **multi-select picker showing each tag's parent** ("Under" column),
pre-checks the current tags, and lets you type a brand-new tag (which then
prompts for its parent and is added to the taxonomy). clog: `tag-list` /
`add-tag <tag> --parent <p>`.

clog: `book-current` / `book-search <q>` / `book-set-status <id> --status
reading|finished|none [--rating N --review T]` / `book-set-tags <id> --tags
"a, b"` / `book-add --title --author --tags`. Catalog at `E:\Media\Books`;
degrades to empty if the drive is unplugged (one-time spin-up lag on first open).

**Tag delimiter note:** book tags can contain spaces ("love letters"), so tags
flow AHK→clog tab-delimited (not space), and the markdown view slugifies spaces
to hyphens (`#love-letters`) since Obsidian tags can't contain spaces. JSONL
keeps the raw tag for analytics.

### Known v1 polish items
- `add log` / friend pickers use the standard two-pane picker, so the right
  "Added so far" pane is dead weight in single-pick mode, and the Tags column
  truncates the longest tag lists. Cosmetic; revisit if it bugs.
- Friend hangouts are tagged only `#friends` (friend name is in the entry text).
  A per-friend tag (`stats cedar`) is a possible later upgrade.
- `log <template>` and `log <friend>` share the `log <X>` shape with disjoint
  Choice sets — keep template/friend phrases from colliding.
