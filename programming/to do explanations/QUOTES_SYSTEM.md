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

## `entry_types` — a quote's FORM (the third axis)

Not everything grabbed from a book is a snappy quote — exercises, meditations,
prompts. `entry_types` is a **multi-valued** list on each item classifying the
*form* (`quote` / `exercise` / `meditation` / …), **orthogonal** to thematic
`tags` and to `source`. An exercise still gets its normal tags + book provenance;
this axis just lets you browse "all the exercises from a book." Kept a dedicated
field (not a tag) so it's deterministic and never mixes into the LLM tag vocab.

- **Default bucket:** empty / missing == a plain `quote`. `query --type quote`
  matches empty items too; `query --type exercise` matches items whose list
  contains `exercise`.
- **Vocabulary:** seeded `[quote, poem, exercise, meditation]` in
  `data["entry_types_vocab"]`, grows as new types are created;
  `entry_type_vocab()` unions seeds + stored + used.
- **Auto-classify:** `default_entry_types(text)` sets `exercise`/`meditation` by a
  leading word ("Exercise …"), then falls back to SHAPE — `looks_like_verse(text)`
  (≥4 lines, most short and not closing a sentence) classifies a `poem`. AHK
  mirrors both in `_QDetectTypes` / `_QLooksLikeVerse`. Keep the two in sync.
  Shape detection only works *after* the OCR reflow has restored the line breaks
  (see "Line breaks" below) — on a flat Kindle clip everything looks like prose.
- **Per-book / per-author DEFAULT type** (`quote_default_types` in library.json,
  book wins over author): most books are entirely prose or entirely poetry, so the
  FORM is a property of the book rather than something to re-pick per grab. A
  poetry collection defaults to `poem`, a workbook to `exercise`. Empty = fall back
  to per-grab auto-detection, which is what a MIXED book (Audre Lorde's *Selected
  Works* — essays and poems in one volume) wants. Read via `quotes.py
  book-entry-types --book-id X` (also in `book-settings`); written via clog
  `book-set-quote-types` / `book-author-set-quote-types`, or the viewer's book
  ⚙ Settings → **Default type (new grabs)** picker.
- **Commands:** `add --types a,b` · `query --type X` · `entry-types` (list) ·
  `entry-type-add --name X` · `set-types <id> --types a,b` ·
  `book-entry-types --book-id X`.
- **UI:** the add-form Miller has a **Type** node — a multi-select toggle list
  (same control as tags) with a `＋ New type…` inline-create leaf. Kindle grabs
  land pre-checked with the book's default type, else the detected one. **Auto-save
  books pass `--types` explicitly** — auto-save skips the form, so without that a
  poem from a poetry book would silently land as a plain quote.
- **Titles are EXTRACTED, never invented.** `derive_title(it)` is the one place
  that picks the extractor, so the save path and `derive-titles` can't disagree.
  A poem uses `extract_verse_title` — the signal is the **blank line under line
  one**, not its length, which is what separates a real title from the first line
  of an excerpt grabbed mid-poem (no gap → no title, rather than losing the
  opening line). An exercise uses `extract_heading` (labelled heading). Applied
  automatically in `cmd_add`, so a poem is titled the moment it lands. The title
  line stays IN the text: that's how the book prints it, and removing it would
  move the text out from under make-preview's similarity check.
- **Front-end:** `open exercises` (see below) browses by type → category → book,
  and the quotes viewer has a **By type** branch. Non-default types are badged in
  the list (`poem · [long] tags…`) — the viewer emits `entry_types` as TSV field
  14 for it. Without the badge a poem is indistinguishable from a quote, which is
  the one thing the type axis exists to tell you.

## Ordering — newest first, grouped by capture recency

Quote lists (`viewer --mode quotes`, non-pool) sort by `added` **descending** and
emit `__DIVIDER__` section headers by capture recency: **Captured today** /
**Captured this week** (last 7 days) / **Earlier**. `_QQuoteNodesFor` renders the
sentinels as non-selectable divider rows. Display pools (`--pool`) keep their
curated order. Every quote stores `added` (UTC ISO timestamp) and, for book grabs,
`source.ref` (Kindle location, e.g. "loc 1951-1954") — both captured at save time.

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
| Exercises browser (AHK) | `Helpers\ExercisesMenu.ahk` + `Scripts\ExercisesViewer.ahk` |
| Line-break reflow | `Helpers\GrabText\GrabReflow.ahk` (+ `Helpers\Tests\test_grab_reflow.ahk`) |
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
ancestor group prefixes) + its linked library book's tags + its author's tags +
its facet bundles** (minus any per-book suppressed tags), then expanded through
the umbrella DAG. Nothing is copied onto the quote —
editing a folder's tags, a book's library tags, or an author's tags instantly
reflows every member. This mirrors the media system's "leaf tags only stored,
umbrellas derived" rule.

- **Folder/cascade tags**: `quote-group-tags`. e.g. `gathas → mindfulness, health,
  calming`; `books/circe → fantasy, myth, mortality`. Tag the folder once.
- **Book cascade**: a book quote inherits its linked library book's genre tags
  automatically (Sand Talk quotes get `indigenous, nonfiction, storytelling` free).
- **Author cascade**: a quote inherits its author's tags too. These live in the
  SAME media library the books do — `E:\Media\Books\<Author>\library.json`
  top-level `author_tags` — NOT a quotes-only store. Edit with clog:
  `book-author-tags <author>` (read), `book-author-set-tags <author> --tags a,b`
  (replace), `book-author-add-tags <author> --tags c` (append). `quotes.py
  author_library_tags(creator)` reads it; one edit reflows every quote by that
  author. Author = identity/theme tags that apply to ALL their work (feminism,
  Black liberation); book tags = collection/medium-specific (poetry vs prose).
- **Per-book suppress**: a book can DROP a tag it would otherwise inherit from
  its author (e.g. a prose collection suppressing an author-level `poetry`).
  Stored on the book entry as `suppress_tags`; `book_suppress_tags(book_id)` is
  subtracted in `item_effective_leaf_tags` (never suppresses a tag put directly
  on the quote). Set via clog `book-set-suppress-tags <id> --tags a,b`.
- **Compound tags, not intersections**: `black feminism` is its own vocab tag
  with **parents `[black, feminism]`**, so tagging it once rolls up to black +
  feminism + identity via `expand_tags` — you keep the specific concept AND
  satisfy every broader query, with no redundant triple-tagging. Same for
  `queer theory` (parents queer, theory). Add compounds with `quotes.py vocab-add
  <tag> --parents a,b --display "..."` (writes the shared `media_tags.json`).
  Apply broad identity tags at the AUTHOR level (true of all their work) and the
  specific compounds at the sub-group/quote level where they actually fit.

### Per-book / per-author auto-save

A book (or a whole author) can be set to **auto-save** new grabs: `quote_autosave`
on the library.json book entry (book wins) or top-level (author-wide fallback).
When on, a Kindle grab from that book **skips the add form entirely** — saves
instantly with book+author tags (derived) and fires the background LLM tagger.
`quotes.py book-autosave --book-id X` (read, used by the grab flow), clog
`book-set-quote-autosave <id> --value on|off` / `book-author-set-autosave
<author> --value on|off` (write). The grab flow's `_KGAutoSaveQuote` shells
`quotes.py add` (no `--no-tag`, so it auto-tags) and honors make-preview
`--replace`.

### Settings + AI sub-grouping in the viewer

Drilling a book or author in `open quotes` shows a **⚙ Settings** row on top
(`_QBookPage` / `_QAuthorPage`): toggle auto-save, edit book tags, edit author
tags, and (book) suppress inherited author tags — all via toggle-pickers that
persist to the media library through clog (`_QClog`). One-call reader:
`quotes.py book-settings --book-id X` (TSV: autosave / author / book_tags /
author_tags / suppress).

### Author facets — reusable tag-bundles ("modes") new quotes plug into

An author writes in a few recurring **modes** — Audre's political/liberation
register vs. her intimate/love register. A **facet** is a named tag-bundle for
one mode, stored per author in `library.json` top-level `tag_facets: {name:
[tags]}`. e.g.
`{"liberation": ["black","theory","liberation","power"], "love":
["black","queer","sapphic","love"]}`.

- A quote plugs into facet(s) via a top-level `facets: [names]` field; it inherits
  the **whole bundle** — derived, so editing a facet reflows every quote in it
  (`quote_facet_tags(it)` is folded into `item_effective_leaf_tags`). It's a
  plug-in, not a cage: a quote can be in multiple facets AND still get individual
  tags on top.
- **Coarse-to-fine tagging:** the background tagger (`apply_auto_tags`) first
  **routes** a new quote into the author's facet(s) — the LLM picks from ~3-5
  curated facets (task `quote_facet`), far more reliable than 300 free tags — then
  the fine individual tagger adds only what the facets don't cover (facet tags are
  passed as `already`-applied so they're not repeated). `route_facets(it)` returns
  `None` when the author has no facets (routing skipped).
- **Manage** them in the viewer: author page → ⚙ Settings → 🧩 Facets — new facet,
  edit a facet's bundle (vocab toggle-picker), **rename** (row action `2`, updates
  member quotes' `facets` too), **delete** (row action `3`), **🔄 Re-route my
  quotes** (re-files every quote by content — run after editing facets), or
  **🤖 Propose facets** (`suggest-facets`, local, seeds a starter set to prune).
  Writers: clog `book-author-facets` / `book-author-set-facet <author> <name>
  --tags …` / `book-author-rename-facet <author> <old> <new>` (+ quotes.py
  `rename-facet-refs`) / `book-author-remove-facet`; quotes.py `author-facets` /
  `set-facets <id> --facets …` / `route-facets <id>` / `route-facets-all
  --creator X`.
- **Model backend — LOCAL only.** Facet proposal + routing use `Scripts\local_llm`
  (free). The Claude gateway (`Scripts\llm_gateway.py`) is **DEPRECATED**: Anthropic
  changed the plan so gateway `claude -p` calls now cost real money — never wire it
  into an automated flow. For a sharper facet set, ask the **interactive** Claude
  Code assistant to do it in-session (that's on the Max plan, $0): it reads the
  quotes, reasons, and writes the facets/routing directly. (That's how the current
  Audre / Miller / Yunkaporta facets were built — grounded in the actual quotes.)
- **Back-fill existing quotes**: `quotes.py route-facets-all --creator X` routes
  every quote by an author into its facets (local model, one at a time, warm) — so
  defining or editing facets applies to the whole existing collection, not just new
  grabs.
- **Facets require the author in the library** (that's where `tag_facets` lives).
  An author referenced only by quotes (no `library.json`) needs cataloging first —
  e.g. Madeline Miller's Circe / Song of Achilles were added via clog `book-add`
  then `quotes.py relink --commit`, which also gave her 117 quotes book-tag
  inheritance they'd been missing.

(This replaced an earlier per-book quote-clustering experiment — facets are the
tagging-assist model Jamie actually wanted: define the modes once, route new
quotes into them, instead of re-picking tags every time.)

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

## Line breaks — Kindle destroys them, OCR geometry puts them back

**Kindle's clipboard hands over ONE flat line.** Verified 2026-08-06 against a
real 8422-character grab: the raw clip contained *zero* newlines in the body.
Every bit of structure a stored quote has was reconstructed afterwards. (There is
also a `re.sub(r"\s*\n\s*", " ", quote)` in `parse-kindle-clip`, but it is not the
culprit — there is nothing left for it to flatten.)

Reconstruction lives in `Helpers\GrabText\GrabReflow.ahk`, shared by the Kindle
grab and the generic Grab Text engine. It re-OCRs the grabbed page region, reads
the WORD GEOMETRY, and re-inserts breaks into the exact clipboard text.

- **Safety invariant:** reflow may only ever change whitespace. The output is
  compared to the input with all whitespace collapsed; if they differ, the
  ORIGINAL text is returned. So it can only add structure or no-op — a wrong OCR
  read can never drop or garble a word. Worst case is today's flat text.
- **Two modes.** `prose` = paragraph breaks only. `verse` = **one break per
  line**, with a wider gap becoming a blank line (stanza break) — because in
  poetry every line break is the poet's, and paragraph-only reflow ran whole
  poems together into a single block.

### The paragraph rule (and the bug that shaped it)

One question decides prose reflow: **which rendered line-ends are also paragraph
ends?** The governing fact is geometric — *a wrap line runs to the right margin;
that is why it wrapped.* Only a paragraph's LAST line is free to stop short.

This was originally four signals OR'd together (sentence-end, gap, indent,
bullet) — any one firing broke the line. So a few pixels of OCR jitter in a
line's start-x, or one slightly tall gap, split a sentence in half. On the 8.4k
Hunter College address that produced **17 mid-sentence breaks against 6 real
ones** (2026-08-06). The fix was not new thresholds but the SHAPE of the rule:
shortness is now a **precondition**, and gap/indent/bullet only corroborate it.

Rules in order (first match wins; each records the reason it fired):

| Rule | Condition | Trusted alone? |
|---|---|---|
| `section gap` | gap after the line is > 1.9× median | yes — a blank line is unambiguous, and the only case a full-width line may break |
| `sentence-end` | line stops short (<90%) AND closes with `. ? !` | yes |
| `short + cue` | line stops CLEARLY short (<80%) AND next is indented / bulleted / gap > 1.3× | corroboration only |

`_RflDecide` returns one record per line, and the SAME records drive both the
reflow and the report — so the explanation can never drift from the behaviour.

### Reading the report — `MAINFUN.bat ShowLastReflow`

Every grab overwrites `%TEMP%\grab_reflow_report.txt` with a per-line table:
reach (how far the line ran, as % of the column), the gap after it, whether it
closed a sentence, the verdict, and the rule that decided it. When a capture
comes out wrong, look for **a BREAK on a line whose reach is ~100%** (a wrap that
should not have broken) or **a wrap on a short line that ends a sentence** (a
break that was missed). Written *before* the reflow acts, so it exists even when
the reflow then no-ops — which is exactly when "why did nothing happen?" is asked.

### The capture log — permanent, and REPLAYABLE

The report above is overwritten by the next grab, which is the wrong shape for
improving the rule: by the time a bad capture is noticed the evidence is gone.
So every grab also appends a durable record to
**`E:\Media\catalog\capture_log.jsonl`** (append-only JSONL — a truncated tail
can never corrupt earlier records, and analysis is a line scan).

A record holds the thresholds in force, the per-line decisions, and **the raw
OCR word geometry** as compact `[x,y,w,h,text]` arrays. The geometry is the part
that matters: it cannot be reconstructed afterwards (the page has scrolled, the
OCR can never be repeated), and it is what makes a capture replayable.

Two record kinds:

| kind | written by | means |
|---|---|---|
| `capture` | the save path, once the quote has an id | what the reflow saw and decided |
| `correction` | the check editor, when the text is edited | **the reflow got it wrong** — before/after says how |

Corrections are the ground truth. Editing a poem in the check editor is exactly
the signal that the capture was wrong, so those records are what the rule should
be tuned against — invented test cases can't tell you what actually fails.

**The improvement loop:**

1. `quotes.py capture-report` — totals, break reasons by frequency, and the
   quotes you corrected by hand (listed first: study those).
2. Change a threshold in `GrabReflow.ahk` (`_RflShortEnd` … `_RflIndentPx`).
3. `MAINFUN.bat ReplayCaptures` — re-runs **the real rule** over every logged
   capture's stored geometry and reports each line whose verdict would change,
   flagging captures you had corrected. A change on a corrected capture is
   evidence of an improvement; a change on one you accepted is a regression
   until proven otherwise.

Replay calls `_RflDecide` itself rather than a reimplementation — that is why
`GrabReflow.ahk` is included in `MAINFUNCTIONS.ahk` (the include-closure check
enforces it). A copy of the rule would drift from the one that actually runs,
which is the single thing replay must never do.

**Gotcha worth remembering:** AHK v2's `Round(v, n)` returns a **String**, so
serialising `Round(reach, 4)` wrote `"1.0000"` into the JSON and every numeric
comparison downstream would have been silently wrong. The capture stores raw
floats.

### Repairing text captured before the fix — `repair-breaks`

`quotes.py repair-breaks [--id X] [--book-id Y] [--commit]` rejoins mid-sentence
breaks left by the old rule. Dry-run by default, printing every join.

Two conditions must BOTH hold: the line before does not end a sentence, AND the
line after starts **lowercase**. Verse lines, list items and headings start with
a capital or digit, so they survive. On top of that it is scoped by
**provenance** — only `source.kind == "book"` items with a Kindle `ref`, never a
poem or verse-shaped text. That scoping is load-bearing: gathas come from an
importer that never ran the reflow and their verse legitimately runs on
lowercase, so the text test **alone would have rewritten 19 gathas**.

Applied 2026-08-06 to 14 book quotes / 80 joins. Verified after: item count
unchanged, only `kind == book` items touched, and **zero words altered** — the
repair is whitespace-only, same invariant as the reflow itself.
- **`auto` (the default)** picks between them via `_RflIsVerse`: ≥4 lines and ≥60%
  of them finishing under 75% of the column's right edge. Kindle wraps prose
  ragged-right filling ~70–100% of the column (the same reason the segmenter's
  short-line rule sits at 55%), so "most lines finish short" is a shape prose does
  not produce. Deliberately conservative — a miss just yields paragraph reflow.
- **A book whose default type is `poem` forces `verse`**, overriding detection.
- Tests: `Helpers\Tests\test_grab_reflow.ahk` (pure tier, synthetic OCR geometry —
  no Kindle or screen needed). Locks down verse-vs-prose detection, one-break-per-
  line, stanza breaks, and the whitespace-only invariant.

## Capture mode — what a grab DOES once it's captured

Per book (or author) in library.json, `quote_capture_mode`:

| Mode | What happens |
|---|---|
| `form` | open the add-quote form and fill it in (the original behaviour) |
| `check` | **save it, then open the review editor** to eyeball / fix it |
| `save` | save silently, no window |

**Poems default to `check`.** Verse is the one case where a capture can be wrong
in a way only a human catches: the line breaks are rebuilt from OCR geometry, and
in a poem the line breaks *are* the content. Once a book's poems come through
clean, flip it to `save`. Everything else, with nothing set, falls back to the
legacy `quote_autosave` flag, so already-configured books are unaffected.

Resolved by `book_capture_mode(book_id, is_poem)`; both resolutions ship in the
single `book-settings` call the grab already makes (`capture_mode` /
`capture_mode_poem`), so this costs no extra Python spawn on the grab hot path.
Write with clog `book-set-quote-mode` / `book-author-set-quote-mode`, or the
viewer's book ⚙ Settings → **On grab**.

### The review editor (`_QReviewQuote`)

Deliberately **the house prose editor** (`_SingleFieldSkipableInputGui`,
`multiline`) rather than a bespoke form — the same surface `add journal` writes
into, per `gui-conventions.md`. What it buys:

- **`line_numbers`** — the Notepad++-style gutter. This is the whole point for
  verse: numbered lines make a wrong break obvious at a glance.
- **`draft_key`** — crash-proof drafting, so a mistake never costs the text.
- The header shows **title · author · book · location**, so provenance is visible
  without opening the viewer.
- **Ctrl+Enter** sends it through; **Esc keeps the quote exactly as captured** —
  it was already saved before the editor opened, so cancelling is never a loss.
- Title is edited as a leading **`title: Some Name`** line, consumed on save —
  the journal's inline-markup convention, so the editor stays ONE writing surface
  instead of growing a field stack. Parsed by `SplitLeadingMarkup` in
  `CommonFunctions.ahk` (only the first line counts; a `title:` mid-body is
  prose), tested in `Helpers\Tests\test_common_markup.ahk`.
- Writes back only what actually changed — a no-op `set-text` would re-derive
  `display_ok` and quietly undo a manual display lock. `set-text` gained
  `--text-file` so a multi-line edit survives (a CLI arg can't carry newlines).

## `open exercises` — browsing practices, not quotes

A quote is something you re-read; a practice is something you DO. So exercises
get their own Miller (`Helpers\ExercisesMenu.ahk` + `Scripts\ExercisesViewer.ahk`,
voice **"open exercises"**) shaped as a shelf you return to rather than a recency
feed: **category → book → the items in READING ORDER**.

- **Reading order is the point.** Exercises are grabbed out of sequence and
  re-grabbed, so `added` says nothing. `viewer --order ref` sorts by the Kindle
  location parsed out of `source.ref` (`ref_position()`), which is a real position
  in the book. Items with no parsable location sort last instead of jumbling.
- **Categories are DERIVED from tags in use** (`viewer --mode typecats --type X`),
  not a separate registry — tag an exercise `ifs` and IFS becomes a category. It
  inherits the usual cascade, so tagging the BOOK `ifs` categorises all of its
  exercises at once. Adding a category needs no schema change.
- **Voice `exercise <category>`** ("exercise IFS", "exercise poetry") deep-links
  straight into one category via the Miller's `initial_path`. `_ExResolveCategory`
  matches case-insensitively, treats underscores as spaces, and compares a
  squashed form too, because Dragon spells acronyms out as "I F S".
- **Items reuse `_QQuoteNodesFor`**, so every leaf drills to the same actions a
  quote does (copy / edit tags / AI retag / delete) with no parallel copy.
- **Names come from the book.** `quotes.py derive-titles --type exercise --commit`
  fills `title` by EXTRACTING the heading from the text (`extract_heading` — line
  one, once reflow has separated it), falling back to the local model (task
  `quote_title`) only for older grabs whose heading is glued to the body. Never
  overwrites a manual title.
- Tunables are settings, not constants: `exercises.entry_type` (the menu works for
  `poem`/`meditation` too) and `exercises.min_category_count`.

## Provenance note — the media_tags.json recovery (2026-07-06)

While building this, found `media_tags.json` had been silently loading **empty**
since commit `a5e3e14` (a corrupt head clobbered the file) — so umbrella rollup +
alias resolution were dead across the whole book/media system, not just quotes.
Recovered by regenerating from `book_tag_aliases.json` via
`convert_book_tag_aliases.py` + re-adding the post-conversion music tags. If
umbrella grouping ever looks broken again, check that `media_tags.json` parses
(`load_taxonomy` swallows a parse error into an empty taxonomy — silent).
