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

## Small print + using more of the page (2026-10-13)

Her report after printing "Repercussions": the Doc prints the title higher and the
poem closer to it (so the text comes out bigger), Arial printed small looks "thin and
ragged", and the print should use more of the page.

- **Margins 0.6in** (was 1.0; `poems.print_margin_in`). Repercussions 10.5 → 11.6pt.
- **Space under the title** `titleGap` 0.6 lines (was a fixed 1.1). Studio: "Space
  under the title" −/+.
- **Small-text font** — when the AUTO size lands below `poems.print_small_below_pt`
  (12) the poem is laid out again in `poems.print_small_font`. **Default: none** — she
  tried Verdana on paper and prefers Arial (2026-10-13); the control stays in the studio
  and in Settings for every poem, off unless she picks a font. Never on a fixed size;
  choosing a font in the studio turns it off for that poem (her pick wins); "Keep this
  font" / "Sturdier font" toggle it (`smallSwap`). `R.fontSwapped` = {from, to,
  wouldBe}; the status line says it happened; the PDF result reports `font` +
  `smallFontFrom`.
- **Why x-height, not width:** a long poem is limited by its LINE COUNT, not line
  width, so every candidate lands at the same pt (Repercussions: 11.5–11.6 in all 13)
  — the font only changes how big that pt READS. Sample sheet of the candidates at the
  size Repercussions gets: `E:\Media\Poetry\print\_font_samples_small_print.pdf`
  (Arial, Verdana, Tahoma, Segoe UI, Segoe UI Semibold, Franklin Gothic Book, Lucida
  Sans, Georgia, Century Schoolbook, Sitka Text, Cambria, Bookman, Palatino). Those
  fonts were also added to the font list.
- Panel: the plain −/+ sections are INLINE (control on the header line), fonts are a
  3-column grid and the panel is 55vw on wide screens, sections reordered so the three
  columns balance — everything open fits 2000×990 exactly.

## Pure black text (2026-10-13)

Her prints looked lighter than the same poem from Google Docs. Cause: the text was
`#111` (near-black). A mono laser can't print gray, so it HALFTONES it — a near-black
becomes a dot screen and reads light and soft. Text is now `#000` (`INK` in
poem_layout.js); the PDF test asserts it. The version mark went from 5pt `#c4c4c4`
(printed invisible) to 7pt `#8c8c8c` — readable, still quiet.

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

### Page shapes in the engine (BUILT 2026-10-18 -- step 2 below, done first)

`poem_layout.js` takes the page as options, so print and the carousel are one engine:
`shape` = `letter` (default) | `instagram` (1080x1350) from `PoemLayout.SHAPES`, plus
overrides `pageW/pageH` (px), `ink` (default `#000`), `paper`, `folio` (`center` "1 / 3"
under the text | `corner` small "2/4" bottom-right | `none`), `folioPt/folioColor`,
`bylineAt` (`top` under the title | `end` after the last stanza -- its space is part of
the layout, and it gets its own slide rather than being squeezed). Results carry
`R.pageW/R.pageH`; the studio reads those, never the Letter constants.
**Checked:** all 481 poems with their saved settings render BYTE-IDENTICAL HTML before
and after the refactor; 4 new tests (21 total) pin the shapes.

### Carousel backgrounds (2026-10-18)

Her ask: a REALLY subtle paper texture; the old poster backgrounds weren't subtle enough.
Measured: their brightness spread (sd) is 14-36 where paper that reads as "just paper"
is ~2-5. So a background is never the photo itself: **paper colour + grain tile x
strength**. `Scripts\poems\studio\paper_textures.py` (store `E:\Media\Poetry\backgrounds\`
= sources/ + tiles/ + backgrounds.json) reduces each photo to its grain (lighting and
blotches blurred out, contrast normalised so every tile is equally strong), white at
the paper's bright end; the page draws it with `mix-blend-mode: multiply` at opacity =
strength. Overlay over a mid-gray tile was tried first and was invisible on near-white
paper even at 30%. 12 in the library: the 7 old poster ones + ambientCG CC0 Paper001/002/
003/006 + Cardboard002. Option sheet: 20% = quiet, 45% = clearly paper.

### Bottom gaps (2026-10-18)

"Assemblage" on two slides left ~1/3 of each slide empty -- it "almost looks like the
poem ends". The size is set by the LONGEST LINE, not the height, so the fixes are:
centre up-and-down (each slide, or the same top on every slide), narrower margins
(0.75in: 26.1 -> 27.5pt), let the longest line(s) turn so the text grows (28.1pt, 2
turned), looser line spacing, or all of them (30pt). Sample sheet rendered; her pick next.

### What other poets do (research, 2026-10-18)

Little poetry-specific guidance exists; what there is: white/near-white backgrounds and
black text dominate (Bilston, Drake, Kaur); Bilston = bold title, left-aligned, name in
cursive below, text size varying by poem length -- i.e. our layout; Times New Roman is
the common face, Gregson uses typewriter scans on tinted paper; long lines wrapping on
phones is THE complaint about posting poems as text (Substack), which is why images;
4:5 1080x1350 is the tallest Instagram allows; keep stanzas together, number slides;
carousel guides put body text >= ~24pt equivalent on a 1080 slide.

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

## Where we are + next steps (2026-10-18) — START HERE in a new chat

**Done:** the print studio (`OpenPoemPrintStudio` / Miller "Print studio…" / :8296) —
auto sizing, phrase splits + manual splits, title size, autosave per poem, pure-black
text, the clipart picker (suggest / ideas / favorites / CLIP search), images placed +
dragged + resized, room for images, every-poem settings. Jamie's verdict on paper:
"they look fantastic". Tests: `Scripts\codebase_tools	ests	est_poem_print.py`.

**Next, in order:**

1. **Instagram carousel** — a second page on the SAME studio server
   (`/carousel/<slug>`), reusing poem_layout.js with a 1080×1350 page size instead of
   Letter (make PAGE_W/H an option), the same sections/keyboard/autosave pattern, the
   same picker. Slides break between stanzas, one size across all slides, title on
   slide 1, byline last, `2/4` marker, optional image slide after the text. Export =
   PNGs via Playwright screenshots (not PDF) to `E:\Media\Poetry\instagram\<slug>\`.
   Decide: background (flat colour / paper texture?), Instagram margins, whether turned
   lines use the same `/` convention on a phone.
2. **Shared "page" engine refactor** — DONE 2026-10-18 (see § Page shapes). Before (1): pull page size, folio and ink
   colour into options so print, carousel and poster are ONE engine with three page
   shapes. Keep the PDF-matches-preview invariant (tests).
3. **Art poster** — the drag editor already exists (images on the print page); the
   poster is a page shape + background + bigger image freedom (bleed, opacity,
   rotation, layering order). Optional layered export for Photopea/GIMP.
4. **Picker improvements** — text flowing around an image (optional, never shrinks
   text); per-image opacity/rotation/flip; "bury" (hide a picture forever); browse by
   source/collection; more clipart (the old wishlist: hand-drawn line art, memento mori,
   Doré) via `Scripts\poem_poster\ingest.py` then `clipart_index.py build` +
   `clip_search.py build`.
5. **Retire poem-poster** — once the carousel + poster replace it: archive
   `Scripts\poem_poster\` render code (keep `sources/` ingest + `_process.py` modes,
   which the clipart pipeline still uses), update POETRY_SYSTEM.md, `printer poster`.
6. **Small polish** — voice phrases ("print this poem", "poem studio"), Stream Deck
   button, chord-line auto mono (low priority — she doesn't print songs), untitled
   poems: no title vs date written.

**Known facts to not re-learn:** stale-code — the studio server holds poems.py /
poem_print.py / clipart_index.py / clip_search.py in memory: `poem_studio_server.py
restart` after editing them (page + engine are read per request). CLIP warms ~25 s
after a restart. Saved per-poem settings: `E:\Media\Poetry\print\print_settings.json`;
picker prefs: `E:\Media\Poetry\poster\clipart_prefs.json`.

## Old order (superseded by the list above)

1. Print — built.
2. Instagram carousel, text slides.
3. Clipart index + search + picker — built.
4. Carousel image slide + art poster editor.
Old `Scripts\poem_poster\` stays until the new system covers it, then archives.

## Open questions

- Chord-sheet songs: Courier New / Consolas are in the studio's font list now
  (chords line up in a fixed-width face); auto-detect chord lines?
- Prose poems: wrap paragraphs normally (no `/` turned lines) when detected as prose?
- Untitled poems: print no title (current) or the date written?
- Studio v2: "attempt to add an image" (after the clipart picker exists).
