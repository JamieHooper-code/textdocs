---
tags: [programming, poetry, design, printing, instagram, clipart, ahk, miller]
created: 2026-10-05
status: print-built-rest-designed
related: ["[[POETRY_SYSTEM]]", "[[SETTINGS_SYSTEM]]", "[[ART_SYSTEM]]"]
---

# Poem print & design

Three ways to turn one of her poems (pen name **Pollen**) into something physical
or shareable, all started from a poem in the `open poems` Miller:

| Output | What | State |
|---|---|---|
| **Print** + **print studio** | Letter paper, title + "by Pollen" + poem; a live page where she sets size / wrapping / pages / font and prints | **BUILT 2026-10-05** |
| **Instagram carousel** | 1080×1350 slides, minimal background, text across pages, optional image slide after | designed |
| **Art poster** | text-first layout + clipart she places by hand (drag editor) | designed |
| **Clipart search + picker** | find clipart that fits the poem, by search and by suggestion; place, drag, resize | **v1 BUILT 2026-10-05** (in the print studio) |

Replaces the old poem-poster render pipeline (`Scripts\poem_poster\`, see
[[POETRY_SYSTEM]] step 6), which she found too hard to use and never used: one
poem ever reached `completed/`.

---

## Why the old poem-poster failed (so we don't rebuild it)

1. **Image first, text second.** The paper template put clipart into the poem's
   body as CSS floats, then `autofit.js` shrank the font until no line wrapped.
   A float narrows the column beside it, lines may not wrap, so the longest line
   that happened to sit next to the picture set the size for the whole poem.
   **A bigger picture always meant smaller text.**
2. **Every fix compensated for the last.** Alpha-traced `shape-outside` polygons,
   a "diagonal cap" so two floats didn't evict each other, per-image transparent
   padding compensation, a bleed formula, two constants that had to move together
   (`PAPER_BODY_HEIGHT_FRACTION` + `.poster` padding). None is needed once the
   text is placed first. `shape-outside` didn't even engage under Playwright's
   `set_content`, so it fell back to a rectangle.
3. **Roulette instead of design.** 12 random variants × 9 token dimensions per
   round, steered with codes like `v0463`, then sweeps to fine-tune size and
   padding. Lots of choosing, no direct "put the bird there".
4. **The algorithm picked the art.** ~3,500 originals (27.8 GB with processed
   siblings) chosen by tag-taxonomy stratified sampling; per-image picking was
   shelved because there was too much to review.
5. **No long poems.** One poster, so a long poem became tiny text. And it ran in a
   separate terminal TUI, outside the Miller.

**Kept from it:** the HTML→Chromium (Playwright) render path, the image-processing
modes (`sources/_process.py`: alpha-light, cutout, otsu…), the ingest sources, the
"finished folder per poem" idea, `printer` integration.
**Retired:** the variant/token system, `menu.py`, the float/polygon layout,
`finalize.py` sweeps, tag-sampled auto-picking (may survive as *suggestions* only).

## The one rule: text first

Every output fits the TEXT first, at its largest size, in its own area. Images
only ever go in what's left. Nothing an image does can make the text smaller.

Measured against the store (464 poems, 2026-10-05): median 15 lines, p90 27,
max 57, 23 over 30 lines. Longest line per poem: median 44 chars, p90 71; 81
poems have a line over 60 chars, 29 over 80.

---

## Print (BUILT)

**The auto rule picks a STARTING point; the studio is where she decides.** Her
words: "I don't think we need to make the system smart enough to always do the
right thing every time ... I'm very particular." So the print opens in a page.

| Piece | Where |
|---|---|
| Layout engine (ONE copy) | `Scripts\poems\studio\poem_layout.js` — used by the studio preview AND the PDF |
| PDF + printer | `Scripts\poems\poem_print.py` (`render`/`print <id> [--opts-file] [--json]`) |
| Studio server, port **8296** | `Scripts\poems\studio\poem_studio_server.py` (`serve/state/stop/restart`) |
| Studio page | `Scripts\poems\studio\print_studio.html` → `http://127.0.0.1:8296/print/<slug>` |
| AHK | `OpenPoemPrintStudio(id)` / `PrintPoem(id)` in `Helpers\PoemsFunctions.ahk` |
| Miller rows (poems only) | **Print studio…** and **Print now** in `Helpers\PoemsMenu.ahk` |
| Per-poem saved settings | `E:\Media\Poetry\print\print_settings.json` (PDFs beside it) |
| Tests | `Scripts\codebase_tools	ests	est_poem_print.py` |

**The studio is self-contained** (her ask): every poem is reachable inside it
(Poem section: search, or Left/Right to step newest→oldest), and the default
settings themselves are editable there (Defaults section → `settings.py set`), not
only from `open settings`. `OpenPoemPrintStudio()` with no id opens the last poem.

**Sections** (collapsible; fold state + active section remembered per browser):
Poem · Text size (−/+ 0.5pt, Auto) · Pages (Auto/1–4) · Long lines (Auto / Turn ( / )
/ Never wrap / Wrap as prose) · Font (the `poems.print_font` options — one list) ·
Alignment · Margins · Line spacing · Space between stanzas · Title · Byline ·
Print (Print / Print on parchment / Open PDF / Save for this poem / Back to
defaults) · Defaults (smallest/largest auto size, default font & margins from the
current poem, parchment by default, byline name). Each header shows the current
value, so a folded section still says what it's set to. The panel is wide
(clamp 660px–48vw–1000px) and flows in two columns so nothing scrolls.

**Keyboard — arrow keys run the whole page** (her ask): ↑/↓ move between sections,
←/→ change the active section's value (fonts cycle and wrap around, sizes step,
options move, buttons in Print/Defaults take focus), Enter folds a section — or
presses the focused button in Print/Defaults, or jumps into the Poem search.
Clicking anything in a section makes it active, so: click once, then arrows.
Also `+ −` size, `/` auto size, `*` cycle long-line mode, `0–4` pages.

**Dark theme + the paper stays white:** the page declares `color-scheme: light dark`
(Chrome's auto-dark leaves pages that support dark alone) and `darkreader-lock`,
because either darkening the PAPER makes the preview lie about the print.

**Long-line modes** (engine `wrapMode`): `turn` (verse — `/ ` turned lines),
`none` (never — auto shrinks to fit, fixed size reports red), `prose` (wraps like
ordinary text, no slash/indent, width does NOT cap the size). Auto-detected:
prose when lines average over 12 words (her call: prose poems wrap normally).
Turned lines are highlighted yellow, lines running off the page outlined red.

**Layout bug worth knowing:** a CSS multi-column box with a constrained height
overflows into extra columns SIDEWAYS — the scroller and the `columns: 2` box must
be two elements.

**Autosave (her ask, 2026-10-05):** every change saves to the poem ~0.5s later and
before switching poems / printing / closing the tab — only the settings that DIFFER
from the every-poem settings, so an untouched poem keeps following them. "Reset this
poem to defaults" forgets them. The server echoes what it STORED and the page warns
if a key was dropped. **Two bugs found building it:** (1) the "saved" copy shared
the live `images` array, so every drag mutated both, compared equal, and was never
saved — `diffOpts` deep-copies now; (2) the long-running server didn't know a newly
added key and silently dropped it while the page believed it saved — hence the echo.
Saves are chained client-side and locked server-side (load-modify-write file).

**Floor is 10pt** (was 14 — "Assemblage" went to two pages that fit on one at 12pt).
"Settings for every poem" is open by default in the studio.

**Room for images** (`reserveIn`, Images section −/+ in 0.5in): keeps the bottom of
every page free; the text is still fitted first, into the shorter box. Without it a
text-first page is usually full and a new image has nowhere to go but a corner.

**Options in layers:** declared `poems.print_*` settings → the poem's saved studio
settings → the studio's live choices. **Printing saves** (what she printed is how
it prints next time, including "Print now" from the Miller).

**Auto sizing rule (her words, paraphrased):** a floor (`poems.print_min_pt`, 14).
Lay out at the floor and count pages = N; grow as large as still fits N pages (cap
`print_max_pt`, 36) and the size where the longest line still fits whole. Only a
line that can't fit even at the floor turns: rest on the next line, indented,
after `/ `. Choosing a page count may go BELOW the floor (to 5pt) — she asked. A
fixed size overrides both. "Never wrap" in auto mode drops to the no-wrap size.

**Pages** break between stanzas; a stanza splits only if taller than a page
(between logical lines, never inside a turned line). `1 / 2` folios on multi-page.
Untitled poems print byline + poem only.

**Bugs found building it (pinned by the tests):**
- The cap landed EXACTLY on the longest line's limit and float rounding turned
  that very line ("Roses are red", "Thumbprint"). The cap now sits at 0.998 of it.
- The font stack's quotes (`"Arial", ...`) closed the `style=""` attribute, so the
  page drew in the browser default (PDF came out Times-like) while the canvas had
  measured Arial. `esc()` now escapes `"`. The test checks the PDF's embedded font.

**Settings** (`open settings` → Poems → Print): `print_min_pt`, `print_max_pt`,
`print_font` (**Arial** — what her Google Doc uses), `print_margin_in`,
`print_byline` (empty = none), `print_out_dir`, `print_parchment`.

**Stale-code note:** the server holds `poems.py`/`poem_print.py` from when it
started (`restart` after editing them); the page and the engine are read from disk
per request, and rendering/printing run `poem_print.py` as a fresh subprocess.

**This is the studio's first page.** The Instagram carousel and the drag-to-place
poster editor are meant to be more pages on this same server, sharing the engine.

---

### Version mark (2026-10-10)

Every working print carries the poem's version number (`v1.2`) in 5pt light gray, 0.3in in
from the bottom-right corner of every page -- almost illegible on purpose, so she can tell
her stack of reprints apart. On by default (setting `poems.print_version_mark`); the
studio's **Version mark** row hides it for a real print. That choice is per PRINT, never
saved per poem (`UNSAVED_KEYS` in poem_print.py), so hiding it once can't leave that
poem's later working prints unnumbered. It is drawn outside the layout and never moves
or shrinks the text. Numbering rule: POETRY_SYSTEM.md § Version numbers.

## Instagram carousel (designed)

- 1080×1350 (4:5) slides, minimal background (flat colour / subtle paper texture).
- **One font size across the whole carousel**, chosen with the print rule's shape:
  floor → slide count → grow to fill those slides.
- Breaks only between stanzas; title on slide 1; byline on the last text slide;
  small `2/4` marker.
- Long lines: turned lines (same as print). A 71-char line unwrapped would force
  ~28px on a 1080 slide — too small on a phone.
- **Image slide is optional and comes AFTER the text**, its own slide, so it can
  never touch the text size.
- Output: `E:\Media\Poetry\instagram\<slug>\01.png…`, opened as a folder.
- Shares the print engine's measure/paginate JS (factor it out when building).

## Clipart search + picker (v1 BUILT 2026-10-05)

Her order (2026-10-05): images on the print FIRST, then Instagram — "the same
system we end up using for the Instagram image finder".

**Index** — `Scripts\poems\studio\clipart_index.py` (`build` / `search` / `suggest`):
one record per ORIGINAL in `E:\Media\Poetry\poster\clipart\_samples` (~3,170;
siblings `<source>.<mode>\<stem>.png` are listed as variants, never as results).
Tags come from poem-poster's own `sources/_tags.py` (folder tags + filename words +
the tags.yaml taxonomy, expanded upward) — one copy of the rules. Writes
`E:\Media\Poetry\poster\clipart_index.json` + 240px thumbnails in
`clipart_thumbs\` (the scans are big and on a spinning disk; a full thumbnail build
is ~45 min, incremental after). Icon sets + openclipart are NOT indexed (ruled out
long ago). Print default variant: cutout > alpha-light (ink on transparent) > rembg
> original. A record without a size (unreadable file) is skipped by search.

**Search:** every word must hit a tag / title word (or a 4+ letter prefix), ranked by
rarity. **Suggest:** the poem's own words + store tags that exist in the clipart
vocabulary, weighted by use in the poem × rarity in the pool; the matched words come
back as clickable chips ("why this picture").

**Picker** (in the studio, Images → "＋ Add image…"): opens on suggestions for the
poem; chips; search box (type, Enter); grid of thumbnails on white. Keyboard: ↓ from
the search box into the grid, arrows move, Enter adds, Shift+Enter / Shift+click adds
and stays open, typing jumps back to search, Esc closes.

**On the page:** a new image goes in the space the text left free on the last page
(centred under the text, as big as that space holds); no room → top corner, small,
with a note. Drag to move, pull the round corner handle to resize (aspect kept).
With an image selected: arrows nudge (Shift = far), + − resize, Del removes, Esc
lets go back to section navigation. Images list: variant (Ink / Cut out / Original
scan…), − +, page (multi-page), ✕. Images sit BEHIND the text and are never part
of the text layout — text first holds.

**Stored** in the poem's print settings as `images: [{id, title, v, vs, page, x, y,
w, ar}]` in PAGE px (816×1056), autosaved like everything else. The PDF embeds each
as a data: URI scaled to ≤2400px (`poem_print.image_map`) and waits for decode.

**Built 2026-10-10 (her "0" to all seven):**

- **Favorites + use counts** — `clipart_prefs.json` beside the index (kept OUT of the
  index so a rebuild can't lose them). ♡ on each picker tile or **F** on the focused
  tile; "★ Favorites" chip lists them. `boost()`: a favorite ×1.5, each use (max 5)
  +8% — what she reaches for rises without burying the rest. Adding an image to the
  page bumps its use count.
- **Image IDEAS from the poem** — local model task `poem_imagery` (tasks.json,
  qwen2.5:7b): 8–12 PHYSICAL things a small engraving could show, including images the
  metaphors suggest. Cached per poem + text hash in `clipart_ideas_cache.json` (one
  call per poem ever; a rewrite asks again). Shown as dashed "Ideas:" chips; merged
  into suggestions at `IDEA_WEIGHT` 0.7. The picker shows suggestions at once and
  refreshes when ideas land (~5–15 s the first time). **Prompt trap:** an example list
  in the system prompt ("wilted flower, empty chair, crow") came back VERBATIM for
  "Help" — the prompt has no examples now. "Help" (which names no objects) went from
  zero suggestions to clouds, rain, bed, blanket, spoon, hand.
- **Picture search (CLIP)** — `clip_search.py` (`build` ~35 s on the GPU for ~3,170;
  incremental). openai/clip-vit-base-patch32 via transformers (weights in `HF_HOME` =
  `C:\Tools\hf_cache`); vectors from the thumbnails flattened onto WHITE, stored in
  `clipart_clip.npz`. `clipart_index.search` = word score (0..1) + 0.9 × picture score
  (relative to the pool's 90th percentile); a picture-only result must reach 0.45 of
  the best. `suggest` also looks each idea up in the pictures (top 6). Fails soft —
  no vectors / no torch → words only. The studio server warms the model in a
  background thread at start (~25 s), after which a search is ~0.1 s.
  **transformers 5 trap:** `get_image_features` returns a ModelOutput (`.pooler_output`),
  not a tensor — `_vec()` handles both.
- **Thumbnails are written write-then-rename** — a build killed mid-save left a
  truncated PNG that "existed", so it was never remade and the CLIP build died on it.
  (One ORIGINAL scan is itself truncated and is skipped with a log line.)
- Suggestion vocabulary also drops weak common words (day, sweet, came, quickly,
  round…) and Dutch/German/French function words (die, en, op…) from Wikimedia names.

**Not yet:** text flowing around images, per-image opacity/rotation.

## Art poster (designed)

1. Fit the text first in a fixed text area.
2. A handful of layout patterns place images only in the leftover space (image
   above / below / beside in its own column / faint background).
3. Generate ~4 starting designs — not 12 rounds of roulette.
4. Open the chosen one in a **drag editor** (local web page): every image is a
   layer to drag/resize/flip/fade by touch or numpad nudges; the text block is
   locked to its fitted size (movable). Clipart comes from the picker.
5. The layout JSON is the source of truth → PNG/PDF render; plus a layered export
   (for Photopea/GIMP) as an escape hatch. Photoshop isn't installed.

## Turned lines: where to split (BUILT 2026-10-10)

`splitMode` (engine; studio section "Where long lines split"; every-poem setting
`poems.print_split`): **phrase** (default) or **late** (book style: as many words as
fit). Phrase = `bestSplit` over every legal k (first part fits the width):

- `grammar(ws, k)`: +3.5 after a sentence end, +3 after `, ; : — –`, +2.5 before a
  subordinator (because, that, when, which…), +2.2 before and/but/or/so/yet, +1.2
  before a preposition; −4 right after a CLINGY word (the, a, my, I, I'm, to, some,
  every…).
- `splitScore` adds balance (−4 × left/right length difference), −3 for one word
  turned / −1.5 for two, −3 for a one-word first half.
- **Parallel lines** (3+ lines sharing an opening of 3–8 words — anaphora): +6 at the
  best GRAMMATICAL spot inside the shared opening, so they all turn alike. Not the
  opening's last word: "Help" shares "…because I'm", whose end is "I'm / unable"; the
  best spot inside it is before "because".
- `bestSplit` takes −3 when the rest wouldn't fit on ONE turned line — less than a
  clingy break (−4), so a third line beats "I'm / unable".
- **Curly vs straight apostrophes are the same letter** (`bare`/`norm`): "Help" mixes
  don't / don’t, and treating them apart broke both the clingy rule and the grouping.

**Her own splits:** click a turned line on the page (highlighted, dashed on hover),
← → moves the split a word at a time, Del = automatic, Esc/Enter done. Stored per poem
as `splits[lineText] = k` (keyed by TEXT, survives edits elsewhere); the engine
ignores one that no longer fits and the page says so. `R.turned` = [{text, k, n,
manual}], segments carry `data-li`.

## Title size (BUILT 2026-10-10)

"Repercussions / Following a Soon to Be Dead Man Into a Dark Alley" wrapped by a hair.
`titleSize` (pt) = hers, fixed (may wrap if she makes it big). Auto = 1.45 × the body,
**shrunk to fit one line** when it would wrap, down to 0.7 × the body (a title may be
smaller than the poem); past that it wraps. Studio "Title" section: Show/Hide + − /
value / + / Auto, ← → steps it. `R.titlePt`, `R.titleRows`.

The panel is three columns at ≥1750px wide, so every section fits open at 2000×990.

## Order

1. Print — built.
2. Instagram carousel, text slides.
3. Clipart index + search + picker.
4. Carousel image slide + art poster editor.

Old `Scripts\poem_poster\` stays until the new system covers it, then archives.

## Open questions

- Chord-sheet songs: Courier New / Consolas are in the studio's font list now
  (chords line up in a fixed-width face); auto-detect chord lines?
- Prose poems: wrap paragraphs normally (no `/` turned lines) when detected as prose?
- Untitled poems: print no title (current) or the date written?
- Studio v2: "attempt to add an image" (after the clipart picker exists).
