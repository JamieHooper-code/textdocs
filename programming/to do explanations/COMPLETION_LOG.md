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
- The **book tag picker** is now the generic `MillerTags` control (a NODE, not a
  separate GUI): drill `Tags ▸` under any book → one ✓-toggle row per tag, ordered
  **current → suggested → relevant → everything else**, with usage counts and
  captioned section dividers. Backend registered in `ReadingMenu.ahk`
  (`_ReadingTagsBackendInit`, system id `"book"`); ordering comes straight from
  `clog tag-rank`, not from the menu. Row 1 is `↑ Push these tags up to the author`
  (`clog book-promote-tags`, additive — skips tags the author already has).
  Toggling writes via `book-set-all-tags` (NOT `book-set-tags`, which preserves
  auto tags and would make toggling an auto tag OFF silently do nothing).
  - **`suggested`** = tags Anna's Archive proposed that anna_metadata.py refused to
    invent (the `unmapped_tags` field) — one keystroke away instead of buried.
  - **`relevant`** = co-occurrence with the book's current tags, rarity-damped
    (`score = co / sqrt(frequency)`), so a precise tag beats a merely-common one:
    `noir` (used once) outranks `queer` (used 33×) for a crime/thriller/mystery book.
    Engine is `media_catalog.rank_related_tags(current, items)` — generic across
    every media type, takes an item iterable so clog can compose books + catalog.
  - The old two-pane `_PickBookTags` still backs the **row-action** fast path
    (`N.M` → Edit tags) from the book LISTS. Drill-ins all use the new control.
- clog `tag-rank`: signal is CROSS-MEDIA (ranks over the whole library — a
  `feminism` album vouches for tags on a `feminism` book) but the OFFER is
  type-scoped (`--for-type book` never lists `album`/`artist`). `--type` narrows
  the corpus when you want within-type signal only.
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
  - **PARTLY RESOLVED (2026-07-16)** for BOOK tags: the drill-in picker is now the
    `MillerTags` checkbox control (above), so no dead right pane and no truncated
    Tags column there. Still open for `add log` / friend pickers and for the book
    row-action fast path, which all still use the two-pane `_PersistentLoopPickGui`.
    Fixing those properly means adding a `layout: "checklist"` opt to that shared
    template (1690 lines, 69 call sites across 20 files) rather than reworking it —
    follow the existing `secondary_catalog` precedent, which already swaps the right
    pane conditionally and defaults to current behaviour.
- Friend hangouts are tagged only `#friends` (friend name is in the entry text).
  A per-friend tag (`stats cedar`) is a possible later upgrade.
- `log <template>` and `log <friend>` share the `log <X>` shape with disjoint
  Choice sets — keep template/friend phrases from colliding.

## Ambient soundtracks — pairing a book with background music (added 2026-07-18)

A book can carry ONE ambient link (a YouTube video today). Opening the book
opens the soundtrack; the Reading Miller shows it next to the book.

**Storage** — on the book, in its `library.json`:

```json
"ambient": {"url": "...", "title": "...", "set": "2026-07-18T04:31:12-04:00"}
```

`clog book-set-ambient <id> --url U [--title T]` writes it (empty `--url` clears
the whole record, so an unlinked book has no half-populated block); `clog
book-ambient <id>` reads it back as `url<TAB>title` and prints **nothing** when
unlinked — so an AHK caller's "is it empty" test is the whole check, with no
sentinel string to keep in sync on two sides.

**`book-recent` gained an 8th column: the ambient TITLE.** The Miller renders
`♪ <title>` in a book's detail line, and shipping the label in the row is what
keeps that free — the alternative is a `book-ambient` subprocess per rendered
book, which is exactly the per-row-subprocess mistake `_ReadingRecentRowMap`
was built to fix.

### Why there is no ambient catalog

There is no unified data backend for ambient tracks yet, so this stores the one
fact we actually have (this book → this url + label) rather than inventing a
taxonomy to hang it off. Every link records url/title/date, so this field is the
migration source if a catalog arrives later. Picking from a library of known
tracks inside the Miller is deliberately deferred until that backend exists.

### Two commands, one write

The link can be established from either side, and the two commands differ only
in which half they already have — whichever window Jamie is in, the link is one
phrase away and she never has to go fetch the other half. Both land on the same
write (`_AmbientLink`).

| Say | Standing in | Function | What it does |
|---|---|---|---|
| `grab music` | Kindle, reading | `GrabAmbientForBook()` | jump to Chrome, take what's PLAYING, come back to the book |
| `grab book` | Chrome, watching | `GrabBookForAmbient()` | take this tab's URL, ask Kindle which book is open — never touches focus |

**Both phrases are literals sharing a prefix with a Dictation sibling**
(`grab <phrase> [<color>]` in `kindle_commands.py`, `grab <link_text_document>`
in `chrome_commands.py`), so each MUST be declared ABOVE its sibling in the same
mapping — Dragonfly prefers the rule it sees earlier (dict insertion order).
Declared below, "grab music" gets swallowed and highlights the *word* "music".
`voice_index check` reports these as COLLISION; that verdict measures prefix
overlap and does **not** account for ordering, so it is a prompt to place the
literal correctly, not a reason to pick a different phrase. `grab reset` and
`grab tab` are the existing precedents in these same two families. Do NOT move
the literal to a new rule class to dodge this — explicitly tested and rejected
(`caster-voice/references/troubleshooting.md` § "Sibling rules with a shared
prefix").

### The gotchas

**Chrome's UIA tree exposes a tab's Name but NOT its URL** — the address bar
only ever shows the *selected* tab. So "what is the URL of the tab playing
music" is unanswerable without first making that tab current. That's why
`ChromeSelectAudibleTab()` selects rather than merely reports: selecting IS the
read.

**The "Audio playing" annotation persists on BACKGROUND tabs**, which is what
makes that function possible where `ActiveChromeTabIsPlayingAudio` can't — the
latter deliberately reads only the selected tab so it doesn't false-positive on
background audio, and here background audio is the entire target. With several
audible tabs, an **ambience**-titled one wins (same `ambience_keywords.txt` as
`ActiveChromeTabTitleIsAmbience`): if a lecture and a rain video are both
playing, the rain video is the soundtrack.

**Tab-name annotations stack in any order** — `"… - Pinned - Memory usage -
246 MB"`, `"… - Audio playing - Pinned"` — and each pattern is anchored to the
END. A single fixed-order pass leaves the inner one behind (a first cut returned
`"Pinned Thing - YouTube - Audio playing"` because "Pinned" was still on the
tail when "Audio playing" was tested). `ChromeCleanTabName` peels until nothing
changes, which is order-independent by construction.

**Focus, don't reopen — and match on the TITLE.** `FocusChromeTab`'s url pass
reads the **address bar**, and the address bar only ever shows the SELECTED tab
(the same UIA limitation that forces `ChromeSelectAudibleTab` to select before
it can read). So a background music tab never matches by url, and passing an
empty `titlePattern` skips the one pass that CAN see it — the tab-strip scan.
Shipped that way it logged `no match -> opening new tab` on *every* open: a
duplicate tab each time, track restarting from 0:00, which is precisely what
focus-don't-reopen existed to prevent. `OpenAmbientForBook` passes the stored
title as `titlePattern`; because the scan is `InStr(tab.Name, pattern)` and the
stored title is the CLEANED name, it still matches once Chrome re-annotates the
live tab with " - Audio playing". The video id (`_AmbientUrlKey`, never the bare
host — that would match any YouTube tab at all) stays as the second signal for
when the track happens to be the selected tab.

**"Already reading" must still start the soundtrack.** `ReadBook`'s
do-not-disturb guard returns early when the requested book is already open, and
the ambient hook was first written inline *after* it — so a second "read lorde"
on an already-open book silently dropped the music, the one thing the feature
exists to do. "Do not disturb" is about the PAGE (where you are in the book);
opening the music disturbs nothing. `_ReadBookAmbient` is now the single
definition called from every ReadBook exit path so they cannot drift again.

**The soundtrack opens on BOTH `ReadBook` paths** (plain open and read-for-real).
The soundtrack belongs to the book, not to the intent — peeking at a book you
linked to rain should still give you the rain. Focus returns to Kindle, because
opening a book means you want to be in the book.

### The ambient library — GenreLinks.ini → catalog items (2026-07-20)

`GenreLinks.ini` was two stores in one coat, split almost perfectly by HOST:
its YouTube sections (WoW 276, Classic 83, Elden Ring 41, Skyrim 14, …) were the
ambient library; its Spotify sections (Spanish 96, IndieFolk 33, …) duplicate
albums `music.json` already models. **Only the YouTube half migrated.**

`Scripts/MediaCatalog/ambient_migrate.py` — `extract` / `titles` / `retag` /
`report` / `commit`. Result: **343 ambient items** in `music.json`
(`subtype:"ambient"`), catalog 9,931 → 10,274, all originals intact.

**Key on the video id, not the url.** `&list=…&index=23` is context, not
identity, so the same track appears under several urls: id-keying merged **85
duplicate appearances** that a url-keyed pass would have imported repeatedly.

**Section names are shorthand, not taxonomy — verify against real titles.**
`[Classic]` is *WoW Classic*, not classical music; all 83 read
"… (1 hour, 4K, World of Warcraft Classic)". Tagging on the section name would
have pushed 83 WoW tracks into the SHARED `media_tags.json` that books and
quotes also read. Caught only because titles were fetched before committing.

**Under-tagging is the quiet failure.** `[Ambient]` contained WoW tracks that,
tagged by section alone, never got `wow` — so "random ambient from wow" would
have silently returned an incomplete set. `TITLE_EVIDENCE` repairs this and may
only ever apply tags already declared in the section map, so it cannot fork the
vocabulary the way `thrillers`/`thriller` once did.

**Authors come free from oEmbed.** `author_name` arrives in the same response as
the title, and for this library the channel IS the artist — Meisio 277,
Ramsiene 41, Everness 16, Athena IV 4. Fetching titles without authors would
have meant a second full pass over 344 videos.

New vocabulary (`mythology`, `ancient greece`) was registered via
`quotes.py vocab-add`, never by hand-editing `media_tags.json`.

### Group links — `ambient: {kind, ref}`

A book's ambient link is now a REFERENCE, not a literal:

```json
"ambient": {"kind": "track", "url": ..., "title": ...}
"ambient": {"kind": "group", "ref": "wow", "title": "wow"}
```

A group resolves to a fresh random pick **at play time** — storing a resolved
url would freeze the choice forever, which is the difference between a
soundtrack and one song on repeat. `book-ambient` returns the same
`url<TAB>title` shape either way, so **the AHK caller never branches on kind**;
adding groups required no change there. Links written before groups existed have
no `kind` and are read as tracks.

Query layer: `clog ambient-pick --tag T | --name N` (alias first — aliases are
hand-curated, so higher confidence), `ambient-tags`, `ambient-list`. Tag lookups
run through `expand_tags`, so `game soundtrack` reaches wow/skyrim/elden ring
without naming them.

`SpotGenreGo` is the cutover point: **catalog first, INI fallback**. Migrated
ambient groups resolve against `music.json`; the un-migrated Spotify sections
keep working through the INI untouched. `StartLockoutWithGenre` inherits this
without knowing about it (packs still win first).

### Two bugs this design created, and their fixes

**Never offer per-field getters on a random pick.** A first cut had
`AmbientPickUrl()` + `AmbientPickTitle()`; for a `--tag` lookup that is a second
RANDOM query, so the tooltip would name a different track than the one that just
opened. `AmbientPick()` returns the whole chosen row.

**A group link stacks tabs.** Every play resolves to a different title, so
`OpenOrFocusChromeTab`'s title match can never hit the tab from last time — two
WoW tracks playing at once. Fixed by remembering the current pick in
`%TEMP%\ambient_now.txt` (a FILE, not a global: every MAINFUN call is a fresh
process) and treating "still open" as "already playing". `force := true`
re-rolls, which is what the menu's explicit *Play it now* does; `ReadBook`'s
implicit open does not.

**Match a title PREFIX, never the whole string.** Tab titles are unstable:
YouTube prefixes the playing tab with `(1) `, Chrome appends
` - YouTube - Audio playing - Memory usage - 209 MB`, and a file round-trip
leaves trailing bytes that `Trim()` does not strip — a full-string `InStr`
failed on a tab that was visibly correct. `_AmbientTitleKey` takes 40 printable
leading chars, which sits after YouTube's prefix and before every suffix.
`ChromeHasTabTitled` is the read-only tab-strip test (no activation, no desktop
switch) that makes the check safe on a hot path.

### Packs integrated — one track, url AND file (2026-07-20)

The scraped audio packs on disk (`E:\Media\Music\<pack>`) and the catalog knew
nothing about each other: 284 WoW mp3s, 279 `wow` catalog items, no link.
`ambient_migrate.py` gained two commands to close it, joining on TITLE:

- **`link-local`** — sets `local_file` on catalog items that have a matching
  download. **332 matched.**
- **`import-local`** — creates catalog items for pack files with NO catalog
  entry (OSRS's 454 game rips, Metroid's 9 ASMR tracks — never in GenreLinks).
  **463 added, no url**, which is correct: a game rip has no YouTube origin.

Ambient library is now **806 tracks**: 332 with both file+url, 463 local-only,
11 url-only. One track = one record that knows its tags, creator, url AND
whether it's on disk.

**The join is by normalised title (`_norm_title`), and the enabling insight is
that yt-dlp filenames ARE the video titles** — it only substitutes the FULLWIDTH
lookalikes for characters illegal in Windows filenames (`｜`→`|`, `：`→`:`,
`⧸`→`/`). Undo those + collapse to lowercase alphanumerics and the two sides
join exactly, no fuzzy score that could mis-pair similar names.

`ambient-pick` ships `local_file` as a 4th column; `AmbientPlay` prefers the
**local file** (instant, offline, no browser) and falls back to the url.

### FINAL root cause of the `spot <name>` crash — a dragonfly patch (2026-07-21)

**The two entries below (and every "overlapping Choices" / "length-2 rule table"
/ "spare IntegerRef" theory) are SUPERSEDED.** After a long, circular hunt
(days of reboot-testing grammar-shape guesses and instrumenting Natlink), the
real cause was found by *reading dragonfly's own source*:

- Dragon reports an **out-of-range Natlink rule id** for unusual `Choice`-value
  words it isn't confident about (`teldrassil`, `wow`, `songs`) — a known DNS
  quirk dragonfly documents (`GrammarWrapper._dictated_word_guesses_enabled`:
  "*DNS does not always report accurate rule IDs*").
- `dragonfly/grammar/state.py::State.rule()` **raises `GrammarError`** on that
  bad id, and the exception aborts the whole recognition — so nothing dispatches
  and no tooltip is possible.
- The `Choice` path (`ListRef.decode`) matches on the spoken **word** and never
  calls `rule()`; only `Dictation.decode` does. So the throw was killing the
  recognition *before* the Choice — which would have matched — got its turn.

**Fix:** one-line local patch to `state.py` — the final `else` of `State.rule()`
returns `None` (dragonfly's graceful "unknown rule" signal) instead of raising.
The dictation alternative then fails cleanly and the Choice matches. This fixes
**all** grammars and every future mis-tagged word — not just `spot`. Full patch,
backup, and re-apply steps: `WIN11_SETUP_GUIDE.md §2.1 Step 8, Patch B`; skill
copy in `caster-voice/references/troubleshooting.md`. Re-apply after any
dragonfly upgrade (upstream master still raises). The `spot <spot_name>` single
Choice + `spot <textnv>` dictation design below was correct all along — it just
couldn't work until `state.rule()` stopped throwing.

**Lesson (logged so it sticks):** when a `GrammarError` comes from a library,
read the library's throw site and its callers *first*. The instrumentation was
adequate; the wasted time was theorizing about DNS internals instead of reading
the ~15 lines of dragonfly that actually raise.

### `spot <anything>` was fully broken — the fix  *(SUPERSEDED — see 2026-07-21 entry above)*

Every Choice-backed `spot <…>` was silently dead: `spot <genre>`,
`spot <spot_slot>`, `spot <music_track>`, `spot <playlist_url>`. Dragonfly threw
`GrammarError: Malformed recognition data: word 'songs', rule id 2` while
DECODING the Choice — the action never ran, so no tooltip was ever possible
(Jamie's exact report: "no tooltip or anything to explain what is happening").
Confirmed across two Choice lists (`songs`/`weaving` in spot_slot, `teldrassil`
in music_track) and it survived a Dragon restart. `spot <textnv>` (Dictation)
was unaffected the whole time.

**Fix: collapse all four into the one working Dictation command**, and resolve
the spoken words in AHK (`SpotResolve`) instead of feeding Dragon four fragile
DictLists. Precedence, in one place: hand-named slot → ambient catalog (alias,
then random-from-tag) → music.json by voice_phrase → GenreLinks section. **Every
exit tooltips, including the miss**, which lists what it tried. The now-unused
`music_track`/`playlist_url` Choice extras were deleted (fewer DictLists = less
of the surface that was throwing).

### Two bugs the local-file column created

**Never put an optional field first in a delimited row read by `_ClogRun`.** A
local-only track has an empty url, so a url-first row began with a tab —
`_ClogRun` Trim()s its output, silently ate the leading tab, and shifted every
column left. `ambient-pick` now emits TITLE first (always non-empty).

**`_ClogRun` didn't strip trailing newlines** — AHK v2 `Trim()` defaults to
spaces+tabs only. The last tab-column (the file path) kept its `\r\n`, so
`FileExist("…Rat Hunt.ogg\r\n")` failed and a local-only track "couldn't find" a
file that was right there. Fixed at the source (`Trim(…, " \t\r\n")`), which
also drops the empty trailing element `StrSplit(out, "\n")` produced for every
multi-row caller — a latent papercut across the whole clog bridge.

### Hardcoded `spot <name>` recognition restored (2026-07-20)  *(root-cause claim SUPERSEDED — see 2026-07-21 entry above; the single-Choice design here is still correct)*

Dictation-only `spot <name>` recognised poorly ("really bad at picking them up
if it's not hardcoded"). The four old Choice commands gave reliable recognition
but were dead. Root cause re-examined: it was the FOUR OVERLAPPING Choices on
the `spot` prefix, NOT Choice-plus-Dictation as such — `windows_commands.py`
runs eight Choices + a Dictation on `open` and is fine. So the fix is ONE
unified Choice, mirroring `open <directory>`:
*(Correction 2026-07-21: overlapping Choices were never the real trigger — the
single unified Choice is a fine design, but what actually unblocked recognition
was the `state.py` `rule()` patch. See the FINAL root cause entry above.)*

- `Scripts/gen_spot_choices.py` writes `hardcoded_spot_names.json` as a MAP
  `{phrase: resolve_arg}` from the catalog (`clog ambient-names` = tags +
  aliases) + legacy slots + GenreLinks genres. Ambient names resolve to
  themselves; a genre resolves to its INI section (`folk punk` → `FolkPunk`).
  Genres are written FIRST so catalog tags override the overlap (`wow` → `wow`,
  not the `WoW` section — the catalog is the real home now).
- `load_spot_names()` (with the import-fallback the cached-helper trap requires)
  backs a single `Choice("spot_name", …)`.
- `spot <spot_name>` (hardcoded, reliable) is declared BEFORE `spot <textnv>`
  (dictation fallback); both call `SpotResolve`.
- The commit step of `ambient_migrate.py` regenerates the Choice, so "committed"
  means "and it's speakable".

`spot wow` correctly opens the YouTube URL again: `AmbientPlay` is URL-first
(packs.ini `spot_source = youtube` is the documented intent — the downloaded
mp3s are the lockout timer's offline source), file only as fallback for
local-only tracks (OSRS).

### The ~200 un-migrated GenreLinks Spotify sections — left as-is, on purpose

Spanish (96), IndieFolk (33), FolkPunk (28), Dance (25), SadBoi (10), … are
**regular-music genre pools**, not ambient: ~130 album URLs + ~65 playlists.
Only ~43 of the albums are already in the catalog; ~87 are not. Folding them in
as first-class genre-tagged albums is the **Spotify scraping pipeline's** job
(it exists — prefetch/media_ingest), needs network scraping for the missing 87,
and is a separate feature (`spot folk punk` = random album, not ambience). They
already WORK today via SpotResolve's GenreLinks fallback, so nothing is broken —
they are just not upgraded to catalog items. Deferred, not forgotten.

### The Spotify genre pools imported before scraping (2026-07-20)

The ~200 un-migrated GenreLinks Spotify sections (Spanish, IndieFolk, FolkPunk,
Dance, SadBoi, …) are now catalog items. Jamie's worry was right: the scraping
pipeline finds albums by walking an ARTIST's discography, so a saved album not
reachable from any scraped artist would never surface — waiting could lose the
curation. So `spotify_genre_import.py` preserves it now:

- **140 new stubs + 41 existing tagged** (183 staged, 14 multi-section dups
  merged). Titles via **Spotify oEmbed** (no key, like YouTube's).
- Stubs are `status: "queued"` (visible/usable), NOT `placeholder` (hidden) —
  these are confirmed items Jamie chose. `enrich_pending: "spotify"` + empty
  creator marks them for later fill; `media_ingest._find_existing` matches by
  URL, so a later scrape of the same album augments in place, never duplicates.
- Genre → tag (`FolkPunk` → `folk punk`), registered via `quotes.py vocab-add`.

`spot folk punk` now resolves against the **catalog**, not the raw INI:
`clog music-pick --tag` picks a random non-ambient item for a genre, wired into
`SpotResolve` as step 4 (before the GenreLinks fallback). The Choice generator
maps genres to their tag so the hardcoded path hits it too. GenreLinks stays as
a fallback, now space/case-tolerant (`folk punk` → `FolkPunk`).

Guard: the regular music catalog predates flat-string tags and holds some
dict-shaped ones; `music-pick` keeps only string tags before `expand_tags`
(which `.lower()`s and would crash on a dict).

### Changing a lockout's music mid-session — `spot` during a lockout

`spot <name>` now works INSIDE a running lockout without breaking it. The
overlay runs in its own process with GLOBAL hotkeys (l/Space/arrows/m) that fire
regardless of focus, so a naive nav would trip them and the pack would play over
the new music. The fence:

- `LockoutBeginExternalMusic()` (fires only when `timer_audio.pid` exists)
  raises `timer_extmusic.flag` and waits ~1.15s (one overlay tick + margin).
- The overlay's `_SyncExternalMusic()` (polled each Tick, next to the test
  suite's `_SyncSuiteSuspend`) sees the flag and **suspends its global hotkeys**
  + **mutes the pack once** (flipping to mirror display via `_RefreshDisplay`).
- `SpotResolve` navigates, then `LockoutEndExternalMusic()` drops the flag; the
  overlay hands hotkeys back — now in mirror mode, so arrows/Space drive the new
  music. The mute PERSISTS: the lockout keeps running, mirroring the new source.
  M brings the pack back. A 30s mtime backstop recovers hotkeys if the caller
  dies mid-nav.

This required separating resolution from action: `_SpotResolvePick` resolves
with no side effects, and `SpotResolve` fences a lockout ONLY once it has a real
target — muting the pack on a miss would leave silence. The fence is a no-op
(and adds no delay) when no lockout is running.

### CORRECTION: `spot` hardcoded recognition uses LITERALS, not a Choice (2026-07-20)

An earlier entry claimed a single unified `spot_name` Choice fixed the dead
Choice commands, and that the cause was "four overlapping Choices". **Both were
wrong.** The single Choice threw the SAME `GrammarError: Malformed recognition
data: word 'wow', rule id 2` and survived a Dragon reboot. And the very first
failure ("spot songs") had "songs" in only ONE list, so overlap was never the
cause.

The honest, empirical picture: **every Choice-backed `spot <...>` in this rule
throws that GrammarError** (four Choices, or one) for a Natlink reason not yet
diagnosed. But **dictation and LITERAL phrases both work** here -- the rule
already runs literal "spot add"/"spot random"/"spot check" fine.

So `_add_spot_literals(SpotifyGlobalRule)` now generates one LITERAL "spot <name>"
mapping entry per hardcoded name (from `hardcoded_spot_names.json`), each firing
`run_mainfun_args("SpotResolve", [arg])` -- list-based so a multi-word arg like
"folk punk" stays whole (mainfun_action's `args_text` would split on the space →
"Too many parameters"). Literals compile to fixed grammar, not a DictList, so
they sidestep whatever breaks the Choice. The generator skips any phrase whose
"spot X" is already a hand-written command, so "spot random"/"spot favorite"
keep their behaviour. `spot_slot`/`genre` Choices remain only for
"spot swallow <...>"; there is no `spot_name` Choice.

Lesson logged: do not claim a Dragon-grammar fix works without a live test --
this is the third iteration on the same bug, each prior "fix" verified only
structurally. The AHK dispatch side (SpotResolve) was always fine; the failure
was entirely in Dragon's Choice decode.

### `spot` GrammarError — ROOT CAUSE from the Dragonfly source (2026-07-20, 3rd pass)

Prior two entries theorised (Choice overlap; "one Choice fixes it") and were BOTH
wrong. Reading the actual Dragonfly source at the crash point settled it:

`dragonfly/grammar/state.py:93` `rule()` raises `Malformed recognition data:
word 'X', rule id N` when a recognised word's `rule_id` is `>= len(_rule_names)`
(and isn't the dictation/letters sentinel). The crash is inside
`elements_basic.py:1164`, the **Dictation element's decode**: saying "spot wow"
makes Dragon tag "wow" with the rule_id of the **genre Choice's list sub-rule**,
but this grammar's `_rule_names` is shorter than that id -> raise. Every failing
word (wow=genre, songs/weaving=spot_slot, teldrassil=music_urls) is a **Choice
list value**. Confirmed deterministic: `bin/reboot.bat` fully kills natspeak +
recompiles all grammars, and it still reproduced -- so it's the grammar
DEFINITION, not stale Dragon state.

**Every `Choice` compiles to a Natlink list sub-rule.** In THIS grammar the
compiled rule table and Dragonfly's `_rule_names` desync (exact Natlink reason
undiagnosed -- other rules with Choices are fine, so it's something about this
grammar's size/shape). The fix that addresses the mechanism directly: **remove
ALL Choices from the rule.** No list sub-rule -> no out-of-range list rule_id ->
the Dictation decode and the literal phrases both resolve cleanly.

So the spotify global rule now has **zero Choices**:
- Hardcoded names are LITERAL `spot wow` / `spot teldrassil` entries generated by
  `_add_spot_literals` (literals are part of the main rule, not a list sub-rule).
- `spot <textnv>` Dictation is the catch-all.
- `spot swallow <genre>` / `<spot_slot>` (the only Choice users) were dropped;
  `spot swallow <textnv>` (dictation) remains. Minor loss: no save-to-named-
  genre-by-voice; restore as a dictation form if wanted.

STILL UNVERIFIED against live Dragon (I can't drive recognition headless) -- but
unlike the first two passes, this is the first that removes the exact construct
the source shows is failing. If it STILL throws with zero Choices, the desync is
independent of the spot Choices and the next step is bisecting the rule's
command count.

### `spot` GrammarError — THE ACTUAL FIX: mapping ORDER (2026-07-21)

Three wrong theories before this (Choice overlap; "one Choice fixes it"; "any
Choice in this grammar is cursed"). The real cause was documented in this
codebase the whole time and Jamie remembered we'd solved it before.

`failure-modes.md § shared prefix` + the LockoutRule's own comment:
> "lockout <textnv>" before "lockout meditate ..." causes Malformed recognition
> data errors on the word "meditate".

**Dragonfly matches a rule's mapping in INSERTION ORDER.** A Dictation catch-all
declared BEFORE a more-specific sibling sharing its prefix throws "Malformed
recognition data: word 'X', rule id 2" on the specific word. The LockoutRule
avoids it by declaring every literal/Choice variant BEFORE "lockout <textnv>".

`_add_spot_literals` APPENDED, so every generated "spot wow" landed AFTER
"spot <textnv>". That's why the log showed pure-dictation "spot well" working
(no literal sibling) while "spot wow"/"spot teldrassil"/"spot folk" threw (each
HAS a literal sibling, sitting after the catch-all). The fix: pop the fallback,
add the literals, re-append the fallback LAST -- verified every literal now
precedes "spot <textnv>".

Both documented rules now hold for this grammar:
- ORDER: specific-before-Dictation (this fix).
- NO within-rule Choice dup: the rule keeps ZERO Choices, so no word is a list
  value in two extras (the BookRule reading_person dedup lesson).

Lesson: when a Dragonfly rule throws "Malformed recognition data", check mapping
ORDER first (specifics before the `<textnv>` catch-all) -- it is the documented,
precedented cause, not an exotic Natlink bug. Cost of not checking the docs
first: three failed iterations.
