---
tags: [programming, quotes, caster, ahk, tagging, media-system]
created: 2026-07-06
related: ["[[MEDIA_SYSTEM]]"]
---

# Quotes System

A tagged-text store for quotes (book highlights, gathas, collected quotes, your
own writing), built as a **sibling of the media catalog** — it reuses the same
tag vocabulary + umbrella rollup, atomic writes and locks, but keeps its own JSON
store and query layer so quotes never leak into `clog media-query`. This is the
first of several planned "parallel text systems" (journal articles next); the
engine is written so a second system is a thin reuse of the same core.

See also [[MEDIA_SYSTEM]] (the book/media catalog it shares infrastructure with).

## The mental model — TWO axes, not three

The one thing to internalize:

- **WHAT it's about → `tags`** (with the umbrella hierarchy). `theory→black`,
  `theory→queer` are tags that roll up to umbrellas. A quote can carry several
  (black AND queer AND disability). **All search + thematic browse is tags.**
  This carries ~all the weight.
- **WHERE it came from → `source`** (which book / gatha / your own / collected)
  + **author** (`source.creator`). One per quote, unambiguous — it answers the
  one thing tags shouldn't: provenance.

`group` is **demoted to a thin provenance "collection"** (`books/<title>`,
`gathas/<situation>`, `mine`, `collected`) — it is NOT a theme axis. It exists so
"browse by collection" works and folder-cascade-tags have something to hang on.
Gatha situations are the ONE place a deeper `group` hierarchy earns its keep
(functional retrieval — "give me a rushing gatha"); they stay nested and also
carry thematic tags for cross-collection search.

## Storage

| Thing | Path |
|---|---|
| Quote store | `E:\Media\catalog\quote.json` — `{"items":[...]}`, media-core item shape |
| Shared tag vocab | `INIDATA\media_tags.json` (quotes opt in via `applies_to: ["quote"]`) |
| Folder/cascade tags | `INIDATA\quote_group_tags.json` |
| Display pools | `INIDATA\display_pools.json` |
| New-tag review queue | `INIDATA\quote_tag_review.json` |
| Engine + CLI | `Scripts\quotes\quotes.py` |
| Importers | `Scripts\quotes\quote_import.py` |
| Viewer + add flow (AHK) | `Helpers\QuotesMenu.ahk` + `Scripts\QuotesViewer.ahk` |
| Voice rule | `caster\rules\quotes_commands.py` (`QuotesRule` in `rules.toml`) |
| LLM task | `Scripts\local_llm\tasks.json` → `quote_tag` |

Quotes are NOT in `media_catalog.MEDIA_TYPES` (so `media-query`/`_iter_media`
ignore them); `"quote"` lives in `media_catalog.EXTRA_APPLIES_TYPES` so the tag
taxonomy validator accepts `applies_to: ["quote"]` without warnings.

## Quote item schema

```json
{
  "id": "quote:<slug-of-text>",
  "type": "quote",
  "text": "full quote text",
  "title": "",                 // short label; set for long quotes, else blank
  "title_source": "",          // "" | "llm" | "manual"
  "source": {
    "kind": "book|gatha|collected|original|manual",
    "book_id": "isbn:...",     // book quotes -> links E:\Media\Books\<a>\library.json
    "title": "Circe",          // book title (searchable)
    "creator": "Madeline Miller", // author/attribution (searchable, browsable)
    "ref": "loc 278",          // kindle location / raw attribution / page
    "color": "pink"            // kindle highlight color (provenance only)
  },
  "group": "books/circe",      // provenance collection (NOT a theme)
  "tags": [{"tag":"mortality","src":"auto"}],  // src: auto|manual|import
  "length_class": "short",     // short<=240 | medium<=600 | long
  "display_ok": true,          // long passages -> false (never on the bell)
  "display_locked": false,     // true once you override display_ok by hand
  "status": "active",
  "added": "2026-07-06T..."
}
```

### Effective tags (derive, don't materialize)

A quote's **effective leaf tags = its own tags + its folder's cascade tags (all
ancestor group prefixes) + its linked library book's tags**, then expanded
through the umbrella DAG. Nothing is copied onto the quote — editing a folder's
tags or a book's library tags instantly reflows every member. This mirrors the
media system's "leaf tags only stored, umbrellas derived" rule.

- **Folder/cascade tags**: `quote-group-tags`. e.g. `gathas → mindfulness, health,
  calming`; `books/circe → fantasy, myth, mortality`. Tag the folder once.
- **Book cascade**: a book quote inherits its linked library book's genre tags
  automatically (Sand Talk quotes get `indigenous, nonfiction, storytelling` free).

## Length + display

`length_class` (short/medium/long) is derived from character count
(`SHORT_MAX=240`, `MEDIUM_MAX=600`). `display_ok = length_class != "long"` unless
manually locked. **Display pools** (`display_pools.json`) are named saved queries
a surface reads:

```
mindfulness_bell -> {groups:[gathas], length_class:[short], display_ok:true}
lockout          -> {tags:[calming,inspiring,hope,resilience], length_class:[short,medium]}
newtab           -> {tags:[theory,identity,philosophy], length_class:[short,medium]}
```

A surface calls `quotes pick --pool <name>` — nothing is tagged per-surface. The
existing `MindfulnessBellRule` is the natural first consumer.

## Local-LLM auto-tagging (favor existing; guard against explosion)

Task `quote_tag` (qwen2.5:7b-instruct, temp 0.15) via the local LLM gateway.
Three guards keep the vocabulary from exploding into one-off junk tags:

1. **Prompt**: tags must be broad reusable THEMES/TONES, never proper nouns /
   characters / places / objects; strongly prefer existing; "when unsure, add
   nothing." At most one new tag, prefixed `+`.
2. **Junk guard** (`_is_junk_proposal`): proposals with digits, >2 words, or >24
   chars never reach the queue.
3. **Frequency gate** (`MIN_PROPOSAL_SURFACE = 3`): a proposed NEW tag stays
   dormant until it recurs on 3+ quotes — one-offs never nag you. `review-prune`
   sweeps dormant proposals.

Flows: **manual add** = synchronous `suggest-tags` (pre-checks tags in the add
GUI); **bulk import** = async (`tag-async` detached) or `tag-batch` (warm model,
sequential). Only existing-vocab tags are applied to a quote; genuinely-new
proposals go to the review queue. Approving one (`review-approve <tag> --parent
<umbrella> [--book] [--apply]`) adds it to `media_tags.json` and optionally back-
applies it. The LLM is seeded with the quote's collection tags as "already
covered — add only quote-specific" so it stops repeating the book's genre.

## Importers (repeatable, idempotent — dedup by source+text)

- `import-kindle <file>` — Kindle "Notes and highlights" export (multi-book).
  Normalizes inconsistent title/author, dedups repeated books, records
  book/loc/color, groups `books/<title>`, resolves `book_id` against the library.
- `import-gathas <file>` — hierarchical `SITUATION → subcategory → line`. Splits
  blank-blocks into affirmation-lists (one quote/line) vs verses (one quote, first
  Title-Case line → title). Groups `gathas/<situation>[/<sub>]`; structural tags.
- `import-collected <file> [--kind original]` — `text - author #hashtags` blocks.
  Best-effort author parse (keeps raw attribution in `source.ref`), keeps
  hashtags. `--kind original` for your own writing (attributed "Jamie").

Run with `--commit` (dry-run without). `quotes relink --commit` re-resolves
`book_id` for unlinked book quotes after new books are added.

## CLI quick reference (`py Scripts\quotes\quotes.py …`)

```
add --text … [--source-kind --book-id --source-title --source-creator --ref --group --tags]
query [--group --tags --source-kind --book-id --creator --display-ok --length-class]
suggest-tags --text … [--group --book-id]        # sync LLM suggestion
tag-async <id> | tag-batch [--limit N]           # async / warm-batch tagging
group-tags <g> | group-set-tags <g> --tags … | group-add-tags | group-suggest-tags
set-tags | add-tags | set-group | set-display-ok | set-text | set-title | remove <id>
groups | tags | tag-report                       # tag-report = curation view
review-list [--min N] | review-approve <tag> [--parent --book --apply] | review-reject | review-prune
pools | pick --pool <name> [--all]
viewer --mode groups|tags|books|authors|pools|review|vocab|quotes|onetags|suggest …
parse-kindle-clip --file <clip> [--window-title …]   # Kindle grab: clip -> quote + provenance TSV
import-kindle | import-gathas | import-collected <file> [--commit]
```

## UI

- **`open quotes`** → `OpenQuotes` (Miller viewer, own process). Browse by
  collection / tag / book / author / display pool + a new-tag review queue. Each
  quote drills to copy / edit-tags (toggle) / move-group / toggle-displayable /
  AI-retag / delete; full text in the right preview pane.
  **Book-aware:** if Kindle for PC is the foreground window with a book OPEN when
  you say "open quotes", it drills straight to that book's page (Left backs out
  to the full tree). Resolution: `_DetectKindleRaw` → `quotes.py
  resolve-window-book --window-title` matches the reading title (ISBN, else
  normalized title) to the store's `source.title`; any miss just opens at root.
  The add-quote Miller lists **✓ Save quote FIRST**, so Enter-on-open commits the
  common case (grab already parsed text + author + book) without arrowing down.
- **`add quote`** → `AddQuoteByVoice`. Grabs the current selection (or type it),
  AI-suggests tags pre-checked in a toggle list, pick a group, save.
- New rule files need `reboot caster` **then** `enable <RuleDetails name>` (the
  plain-words name at the bottom of the rule file — for this one, "enable quote
  rules"). Edits to existing rule files hot-reload; brand-new files don't.

## Kindle grab — highlight on Kindle → quote in the store

The Kindle for PC app (Qt) renders the page as a custom-drawn surface with **no
accessible text** (UIA gives an empty `[document]`). So the grab flow OCRs the
page (`Lib\OCR.ahk` = Windows.Media.Ocr) to locate words, synthesizes a
click-drag between two word anchors to select, `Ctrl+C` to capture the exact
text **plus the citation Kindle appends on copy** (author / ISBN / Kindle
location), and invokes the highlight-color swatch via UIA.

**Flow:** say **"show grab"** (sentences) or **"show block"** (paragraphs) → a
click-through overlay outlines each unit (alternating white/gray boxes) with a
number badge at its start. Then **type a number** on the numpad (or `N.M` for a
range, e.g. `4.6` = 4→6) and **Enter** → the selection previews as **bright cyan
boxes** (nothing is highlighted on Kindle yet). **Nudge it with the arrows** —
**← →** move the end out/in a word, **↑ ↓** move the start — then **Enter** grabs:
it drag-selects the final span, **highlights it blue**, and drops it into the
add-quote form pre-filled with text + author + book link (from the ISBN) +
AI-suggested tags. **Esc** backs out. The preview-then-confirm design means you
correct any segmentation slip *before* anything is highlighted (OCR can't read
already-highlighted text, so this avoids a re-grab dead end).

In the adjust preview, **Enter** grabs, **Backspace or Esc** cancels, and the
`adjust: …` status tooltip is a tracked child process (`DebugTooltip.ahk`) that
is **killed the instant you commit/cancel** — `_KGStatus`/`_KGKillStatus` in
`KindleGrab.ahk` hold its PID (the tooltip otherwise runs for its full 2-minute
duration and only dies on a physical Esc, so it used to linger after a save).

**Preview is a MODE** (default ON, flag `INIDATA\kindle_grab_preview.txt`,
toggled by voice **"switch preview"**). When ON, EVERY grab — typed number,
voice **"grab N"/"grab N to M"**, voice **"grab <phrase>"** (say `X to Y` for a
range) — opens the cyan adjust preview. When OFF, grabs highlight immediately.

**"make preview"** takes the LAST committed grab, un-highlights it (toggles its
color off), and re-opens it as an adjustable preview — so you can keep preview
OFF for speed and only fix a grab after the fact (`INIDATA\kindle_grab_last.txt`
stores the last selection's word rects + color). Re-confirming a "make preview"
edit **overwrites the original quote instead of adding a near-duplicate** — but
only when the edited text is still *the same passage*: `_QAddSave` records the
last grab's quote id to `INIDATA\kindle_grab_last_quote.txt`, make-preview passes
it as `add --replace <id>`, and `quotes.py` overwrites only if word-token Jaccard
≥ `REPLACE_SIM_THRESHOLD` (0.5) — a genuinely different page adds fresh, never
clobbering an hour-old quote.

**"grab reset"** clears all highlights on the page (select-all → blue → blue
again toggles off) so OCR reads cleanly again — highlighted text is unreliable
for OCR.

**Bare numbers while reading** — `kindle_book_commands.py` adds a *book-reading*
context where the word "grab" is OPTIONAL: say just **"32"** or **"32 to 34"** and
it grabs. These bare forms are reckless globally (Dragon transcribes stray
numbers), so they live ONLY in this hyper-specific context; the reliable **"grab
N"** forms stay in `kindle_commands.py` (active everywhere in Kindle). Add future
reading-only hotkeys to `kindle_book_commands.py`.

The context is a **`function_context` predicate** (`_reading_a_book`), NOT a title
substring. The reading title is NOT reliably `… -- <author> -- isbn …` — a
sideloaded document reads as just `Jamie's Kindle for PC 4 - Critique Process Grad
TRANS CLASS` with no `--` at all. So the predicate keys on the **` - <name>` tail**
after the app prefix (any non-empty name = a book/doc open), excluding Kindle's
own view names (library/home/store/settings). It reads the live foreground title
via `window_context_helpers.foreground_title()` (one Win32 `GetWindowTextW`, cheap
per-utterance). Same signal powers book-aware "open quotes".

Segmentation breaks a unit on sentence punctuation, a paragraph/section gap
(> median line spacing), a column jump, OR a **short line** (right edge fills
< 55% of the column) — the short-line rule makes a dateline/header like "May 28,
1985" or "Cambridge, Massachusetts" its own unit. The 55% threshold is high on
purpose: Kindle wraps prose ragged-right (lines fill 70–100%), so a lower
threshold splits every line. Voice "grab N" collided with the global CCR
**"Grab [<n>]"** (select-N-chars); that global command was commented out (Caster
forbids context-scoping a `CCRType.GLOBAL` rule, and app rules don't win over
global CCR).

**Architecture** — an own-process host keeps the 100 KB OCR lib OFF the MAINFUN
voice hot path:

| Piece | Role |
|---|---|
| `caster\rules\kindle_commands.py` | voice grammar, all of Kindle (`executable="kindle"`) |
| `caster\rules\kindle_book_commands.py` | book-reading-only grammar (`function_context=_reading_a_book`, keyed on the live title's ` - <name>` tail) — bare-number grabs + future reading hotkeys |
| `Helpers\KindleGrabLaunch.ahk` | thin launchers (in MAINFUNCTIONS) that spawn the host |
| `Scripts\KindleGrabHost.ahk` | own-process host: OCR + UIA + Miller/Quotes stack + engine; `#SingleInstance Force` so a grab auto-dismisses the overlay |
| `Helpers\KindleGrab.ahk` | the engine (OCR, segmentation, overlay, drag, highlight, add-form wiring) — loaded ONLY by the host |
| `quotes.py parse-kindle-clip` | splits the copied clip into quote + creator/title/isbn/ref/book_id |
| `Lib\OCR.ahk` (Descolada, vendored) | excluded from the codebase scanners via the `"Lib"` dir name, like `UIA-v2-main` |

**Proven quirks (2026-07-06 spikes):** OCR coords are CLIENT-relative → convert
with `ClientToScreen`; a drag from the first word's center to the last word's
center selects the whole span in reading order, even across line wraps; `Ctrl+C`
CLEARS the selection, so highlighting = re-select then UIA-invoke the swatch;
the copy citation is the ONLY source of the Kindle location; **`Esc` exits
Kindle fullscreen — never send it**; the color swatches are checkboxes named
`"<color> highlight"` under `NotecardViewClass`, addressable by UIA name so the
popup's shifting position doesn't matter.

## Provenance note — the media_tags.json recovery (2026-07-06)

While building this, found `media_tags.json` had been silently loading **empty**
since commit `a5e3e14` (a corrupt head clobbered the file) — so umbrella rollup +
alias resolution were dead across the whole book/media system, not just quotes.
Recovered by regenerating from `book_tag_aliases.json` via
`convert_book_tag_aliases.py` + re-adding the post-conversion music tags. If
umbrella grouping ever looks broken again, check that `media_tags.json` parses
(`load_taxonomy` swallows a parse error into an empty taxonomy — silent).
