---
tags: [programming, poetry, design, printing, instagram, clipart, ahk, miller]
created: 2026-10-05
status: print-and-carousel-built-poster-designed
related: ["[[POETRY_SYSTEM]]", "[[SETTINGS_SYSTEM]]", "[[ART_SYSTEM]]"]
---

# Poem print & design

Three ways to turn one of her poems (pen name **Pollen**) into something physical
or shareable, all started from a poem in the `open poems` Miller:

| Output | What | State |
|---|---|---|
| **The studio** (one page, Shape switch) | Print (Letter) and Instagram (1080×1350) on one page, one store, shared choices linked — § "One studio, every shape" | **BUILT 2026-11-03** |
| **Print** + **print studio** | Letter paper, title + "by Pollen" + poem; a live page where she sets size / wrapping / pages / font and prints | **BUILT 2026-10-05** (page merged into the studio 2026-11-03) |
| **Instagram carousel** + **pick-an-option** | 1080×1350 slides on paper grain; Layout / Background / Images picked from numbered options; PNG export | **BUILT 2026-10-18** (page merged into the studio 2026-11-03) |
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

## Instagram carousel (BUILT 2026-10-18)

`http://127.0.0.1:8296/carousel/<slug>` -- AHK `OpenPoemCarousel(id)` (PoemsFunctions.ahk),
Miller row **Instagram carousel…** under Print studio, and the ◇ pages header ("Poem studio").

| Piece | Where |
|---|---|
| Page | `Scripts\poems\studio\carousel_studio.html` (read per request) |
| Store + export | `Scripts\poems\carousel.py` (`export <id> [--opts-file] [--json]`) -- per-poem picks in `E:\Media\Poetry\instagram\carousel_settings.json`, PNGs to `E:\Media\Poetry\instagram\<slug>\01.png…` (old numbered PNGs cleared first) |
| Server routes | `/carousel/*`, `/api/carousel/{poem,save,reset,export}`, `/paper/<name>` in `poem_studio_server.py` |
| Settings | Settings → Poems → Carousel: `carousel_min_pt` 20, `carousel_max_pt` 60, `carousel_margin_in` 1.0, `carousel_paper` (ambientcg-paper001), `carousel_paper_strength` 0.2, `carousel_out_dir` |

- Engine `shape: 'instagram'`: 1080×1350, small gray `2/4` bottom-right, byline AFTER the last
  stanza (its own slide rather than squeezed), title on slide 1, one size across all slides,
  breaks between stanzas (or a stanza continues -- `splitStanzas`).
- Long lines turn with the same `/ ` convention as print. Her rule: avoid turns unless they
  make the text substantially bigger, and then only a couple.
- The page = slides on the left, a panel of rows on the right: the three ASPECTS (Layout 1,
  Background 2, Images 3 -- pinned voice numbers) then −/+ rows (size, side margins, top +
  bottom, line spacing, stanza space, paper grain, font, alignment) and Export PNGs.
  ↑ ↓ rows, ← → change, Enter opens / presses. Autosave like print (only what differs).
- Every button has a kit.js voice number; title ends " - Pages", so the shared bare-number
  rule ("3") works with no Caster change.
- Images are placed over the finished slides (her model), anywhere they fit -- top right /
  bottom right beside the shortest lines, as big as possible, the same spot on every slide
  (`images[i].every`) or a different spot on each. "Shown as a last slide" was my wrong
  reading -- there is no image slide.

## Pick-an-option studio (BUILT 2026-10-18)

Her idea: single out ONE thing (spacing, background, images…), see a handful of genuinely
DIFFERENT fixes side by side, big, numbered; say or type the number; on to the next aspect.
"There are multiple different optimal solutions depending on what you are optimizing for" --
each option is the best answer to one goal. Least clicking for the best options.

- `Scripts\poems\studio\poem_variants.js` -- the generators, pure JS over the engine:
  - `layout()` -- spacing, size and margins as ONE aspect (she sees them as one solution
    space; margins are NOT their own aspect). Early-slide gap fixes (smaller text ≤ 8%,
    tighter line spacing, less stanza space, smaller top + bottom margins, let a stanza
    continue, a little of both), plus bigger-text options (narrower side margins; a line or
    two turned for ≥ 8% bigger, never past maxPt; one page/slide fewer; one MORE page/slide
    when small text gets ≥ 15% bigger). An option is offered only if it leaves no early-page
    gap, and near-copies are dropped. Lots of space on the LAST slide is fine (her words).
  - `background()` -- plain white, plain cream, every paper grain at the current strength,
    and "its colour" only where the paper's colour is visibly not cream.
  - `images()` -- one picture: every-slide corners, biggest spot on each slide, last-slide
    corners / biggest free spot, first-slide corners, and TEXT FLOWS AROUND IT (right / left /
    middle-with-text-above-and-below, each on the first and last slide, plus "few turns" = the
    biggest that turns at most 2 more lines). Free space comes from `PoemLayout.boxes(R)`.
- `Scripts\poems\studio\option_grid.js` -- the full-window numbered grid, shared by BOTH
  studios: sizes itself so every option is on screen at once (tries every column count),
  pinned voice numbers 1..N, the page's own numbers parked while it is open, digits pick
  (a second digit waits 0.9 s, or Enter), arrows + Enter, Esc = back unchanged.
- **Two views** (her ask, 2026-10-18 evening): **grid** (all at once) and **one at a time**
  (full screen, as big as it fits -- "I can't hardly see them at that size"). Tab switches
  and the view she switches to is remembered per aspect. One-at-a-time: ← → / PgUp PgDn /
  scroll wheel (one step per burst, so the Chrome pedals' 11-notch scroll is one step) /
  poem-studio pedals swap, wrapping around; **Space** = auto-advance on/off for now;
  **Enter** picks the one showing; numbered chips along the bottom jump straight to one
  (voice). Settings → Poems → Pick-an-option: `options_view_layout` (grid),
  `options_view_background` (one), `options_view_images` (grid) -- each grid / one /
  one_auto -- and `options_auto_seconds` (6). Read via `/api/options`, written through
  `/api/setting` one at a time (two quick switches raced once and stored the first).
- **Ranking = her affinity system** (2026-10-18): one view ↑ / ↓ = +1 / -1 (← → still
  swap), grid Shift+↑ / ↓, ▲ Like / ▼ Less buttons for voice. Scored per aspect + option KEY
  (a paper, a kind of layout fix, a kind of placement) -- what recurs across poems -- on the
  SHARED scale via `Scripts\poems\studio\option_affinity.py` -> `MediaCatalogffinity.py`
  (bands favorite/loved/essential, buried below zero, a dated log). Store
  `E:\Media\Poetry\option_affinity.json`, routes GET/POST `/api/option-affinity`. Order is
  set when a screen OPENS: as-it-is first, then liked, buried last -- never re-sorted while
  she ranks.
- Grid fit: card chrome is MEASURED from a real card, then re-fitted (a guessed label
  height pushed the bottom row off the screen); labels are one line, ellipsised.
- **Foot pedals**: context `INIDATA\Contexts\poem_studio.json` (title ends " — Carousel /
  Print - Pages"), bindings left-set pedals (left:left / left:middle / left:right) = Left / Space / Right in pedal mode
  default. Set2 IS her right set (confirmed 2026-10-18). Set1 keeps its Chrome scrolling,
  which the one view also treats as next / previous.
- Print studio: "Pick from options" section at the top -- **Layout options** (1) and
  **Place an image…** (2; after the picker, the placements open; Esc keeps the default spot);
  each image row has **Move…**. A one-option result says "nothing better to offer" instead of
  opening. `poem_print.py` OPT_KEYS gained `marginYIn`, `splitStanzas`.
- Shared clipart picker module `clip_picker.js` (the carousel uses it; the print studio still
  has its inline copy -- move it over when next touching the print picker).

### Text around images (engine, BUILT 2026-10-18)

`images[i].wrap = true`: rows beside the picture get narrower (`seg.dx/w`), rows with under
30% of the width are skipped (`seg.pre`) so the lines go below it. TEXT FIRST holds: the size
is chosen with no wrap-images, then the text flows around them at that size (more turns or a
page, never smaller text). `splitStanzas` is honoured there too (a bug caught in testing: the
first version dropped it and the early-slide gap came back). Tests pin: same pt with and
without the picture, no line box overlaps it, nothing lost, boxes match the DOM.

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

### Bottom gaps on slides that aren't last (2026-10-18)

Her correction: lots of space on the LAST slide is fine; lots of space on an EARLIER
slide is the problem (it reads as the poem ending). Cause: a stanza that doesn't fit
starts the next slide. Engine options added: `marginYIn` (top/bottom apart from the
sides) and `splitStanzas` (a stanza may continue on the next slide at a line break,
2+ lines each side). Survey of all 481 poems as slides (Arial, 1in, floor 20pt): 53
need 2+ slides, 29 of those have an early-slide gap over 1.6 lines, and the gap
generator closes it in 27 of the 29.

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

## The library: the Poetry Studio as the whole poetry home (phase 1 BUILT 2026-11-03)

Jamie (2026-11-03): convert the poems Miller into the basis for the Poetry Studio, so either
one can do ALL the poetry handling -- search, browse, read, edit, mark, print -- not just printing.

**One definition, two surfaces.** `Scripts/poems/poem_library.py` defines the shelves (the
Miller's root, Browse by rank/status/person/tag/year, Scraps, Google Doc, More), one poem's
actions + marks, and the pick-lists (rank, status, tags, scrap type). Each row carries a `do`
(a `poems.py` verb, a MAINFUN function, a picker, a link). The studio page draws it from
JSON; the Miller (`Helpers/PoemsMenu.ahk`) draws it from `--feed` TSV through the generic
`Helpers/Gui/MillerFeed.ahk` (format: `Scripts/viewers/miller_feed.py`) -- phase 2, BUILT
2026-11-03. PoemsMenu.ahk went from ~700 lines to ~170 (openers + the Doc check). A new
shelf or action is added ONCE, in poem_library.py.

**Page:** `studio/library.html` at `http://127.0.0.1:8296/` (and `/poem/<slug>`); the print
studio moved to `/print/`. Three panes: shelves + list | the poem | Do + Marks. Endpoints
`/api/lib/*` in `poem_studio_server.py`; writes go through `poems.py` (the one writer) or
`MAINFUN.bat`, both checked against `SET_VERBS` / `MAINFUN_FNS`. The page polls
`/api/lib/stamp` and redraws in place when the store or Doc state changes (writing-box saves
show up by themselves). Opening it starts a background Doc check, like the Miller.

**Voice + keys:** every control has an inline kit number, EVERY TEXT BOX TOO (she asked:
"numbered like any other link, so I can jump into them"). Pinned (2026-11-05: pins live in 11-29, never the 80s/90s): 11 back, 12 shelves,
13 search, 14 open in the Miller; 15-29 the most-used panel rows (read 15, edit 16, mark 17,
favorite 18 / less 19, rank 20, status 21, tags 22, title 23, print studio 24, print 25,
carousel 26, journal 27, special 28, in progress 29); a poem shows more rows than fit, so the
rare ones run on just past: kind 30, send/copy doc 31/32, versions 33, open doc 34, trash 35,
Doc warnings 36-38. `PINS` in `Scripts/poems/poem_library.py`. List rows read 1-10, then
carry on past the pins. A pick-list covers the page and numbers only its own choices
(11 = its filter box, 12 = Done).
Keys follow the Miller: Up/Down, Right/Enter in, Left/Escape/Backspace out, a letter starts a
search, digits + Enter = say that number. Trash needs two presses. Context `poem_library`
(pedals: left Up, middle Enter, right Down). Voice `open poetry studio`; setting
`poems.open_with` (miller | studio) decides what "open poetry" opens -- default miller.

**The shared kit (BUILT 2026-11-03, same day):** the page code moved out of the studio into
`Scripts/viewers/kit/feed_page.js` + `.css` and `Scripts/viewers/feed_routes.py`; the spec
contract into `Scripts/viewers/feed_spec.py`. Stories is the second system on it
(`story_library.py`, page on the shared `feed_server.py` :8302, voice `open stories page`,
setting `stories.open_with`, context `stories_page` + the same pedals). New systems start
from `new_feed_system.py <Name>` (tested with a throwaway system, then `--undo`).
`poems.open_with` is now `studio`: "open poetry" opens this page. Reading is the third
(`Scripts/reading/reading_library.py`, page :8302/reading/, setting `reading.open_with`,
context `reading_page`; see [[READING_HUB]]). Ideas for next systems:
practices, reminders' parts, the journal.
**A system whose page is fully fleshed out opens the PAGE by default** (Jamie,
2026-11-03): `<x>.open_with` defaults to the page, and "open <x> miller" opens the Miller.
Poems ("open poetry miller"), Stories ("open stories miller") and Reading ("open read
miller") all do this. A freshly generated system starts on the Miller until its page is
real, then flips.
Every kit Miller links to its page ("Open the page", under Settings) and every kit page to
its Miller (quiet button at the foot of the list, voice 14) -- built into the kit, not the
specs (Jamie, 2026-11-03).

## Her first real use (2026-11-03) — START HERE in a new chat

**Papers she likes** (▲ on the Background screen, `option_affinity.json`): 9 of 12 —
Cotton paper (ambientcg-paper001), Fibre paper (paper006), Smooth card (cardboard002),
Speckled paper (paper002), Aged parchment 1/2/3, Antique parchment, Kraft. NOT liked:
linen-neutral-01, parchment-aged-dark-01, ambientcg-paper003. "Get more like those."
- Those 3 are now `"hidden": true` in `backgrounds.json` (kept on disk, not offered;
  `carousel.papers()` skips them). A paper's display name can live in its registry row
  (`label`), so a new paper needs no code.
- **8 more papers** (2026-11-03): ambientCG Paper004 "Kraft lines", Paper005 "Smooth tan"
  (CC0; Cardboard001/003/004 skipped — torn corrugated) + 6 public-domain Wikimedia
  Commons photos of old paper (`commons-old-paper-*`: spotted, mellow, foxed, pale,
  folded, burnt-edge). ambientCG has no more papers; Poly Haven has none. 17 offered now.
- **A new carousel starts on a RANDOM favourite paper** (her ask: not the same one every
  time). `carousel.starting_paper(poem_id)` = her liked papers (affinity > 0) picked by a
  hash of the poem id — random across poems, stable for one poem. No favourites = the
  `carousel_paper` setting. Liking more papers widens the pool.

**Plain turned lines (BUILT 2026-11-03).** Engine option `turnStyle`: `slash` (default,
"/ " + indent) | `plain` (a turn is just a new line — same split point, no slash, no
indent). Her words: some turns read as a real line break ("Even my Rosemary Bush, / who I
know is far too established"). Button in both studios ("Turned lines": With / | New line),
saved per poem; the default is the setting `poems.turn_style` (Settings → Poems → Print),
now **plain** (her call, 2026-11-03) — so a poem with no saved choice uses new lines.
Width checks, `boxes()` and the PDF all honour it. Checked: 482 poems × print + carousel
render byte-identical with the default; test `test_plain_turns_are_a_new_line_...` (30 tests).

**Clipart search, "tree stump".** Nothing came up because the pool had NO stumps (0 of 3,168
records mentioned one) — not a search fault. Hence:

### Find more online (BUILT 2026-11-03)

In the clipart picker (`clip_picker.js`, so every studio shape has it): **🌐 Online (14)**
looks for the search words on **Wikimedia Commons** (searched as "<q> engraving",
"<q> illustration", "<q>", line art sorted first) and **Openverse** (Flickr Commons,
museums), **public domain / CC0 only**, and lists what is not in the library yet under a
blue line below the library's own pictures. It runs **by itself when the library has fewer
matches than `poems.clip_online_below` (12)**. Then:
- **pick one** (its number / Enter / click) → downloaded, cut out, added, placed (~10–15 s);
- **Download all N (15)** → every online result into the library (the tiles say "getting…");
- **Get 30 more like this (16)** → `poems.clip_online_more` more in the background. It is
  STRICT: it only takes results whose title names every searched word, because nobody is
  picking them — a loose full-text hit ("Sanctuary Wood" for "tree stump") is fine to SHOW,
  never to download unasked. The first test run without that rule pulled 3 WWI paintings.
15/16 sit in the top bar (always on screen), not down in the grid.

Engine `Scripts\poems\studio\clip_online.py` (CLI `search` / `get`): files go to
`E:\Media\Poetry\poster\clipart\_samples\online\<stem>.jpg` + `<stem>.json` sidecar
(title, tags = searched words + title words, source, page url, licence, creator, query);
variants `online.cutout\` + `online.alpha-light\` via poem-poster's `_process.py`;
`clipart_index.add_originals()` adds just those records (no 3,500-file walk; a full
`build` honours the sidecars too); `clip_search.build()` embeds only the new ones. Every
download is a line in `poster\online_provenance.jsonl` — its key is what hides a result
already in the library. Jobs: one worker thread in the studio server; routes
`/api/clip/online/{search,get,more,job}`. Tests `test_clip_online.py`.

**Traps found building it:** Commons now serves only its STANDARD thumbnail widths
(…330, 500, 960, 1280, 1920) — a 320px or 1600px thumb URL silently returns nothing, so
the tiles drew blank; Openverse caps anonymous `page_size` at 20 (40 = 401) and went
unresponsive for minutes on the day (curl too), so after one timeout it is skipped for 10
minutes and Commons answers alone in ~4 s; a fast burst of full-size Commons originals
earns a 429 (hence the 1920px thumbs); `np.load` without `with` keeps the CLIP `.npz`
locked on Windows, so the next build's replace failed "Access denied".

**One studio, not two — BUILT 2026-11-03, see § "One studio, every shape" below.**

## One studio, every shape (BUILT 2026-11-03)

Her words: two separate studios drift apart and need double maintenance, and she'll often
print a poem and make its carousel at once, with different settings per shape. So the print
studio and the carousel studio are now ONE page.

| Piece | Where |
|---|---|
| Page | `Scripts\poems\studio\studio.html` → `http://127.0.0.1:8296/studio/<slug>?shape=print\|instagram` |
| Old addresses | `/print/<slug>` and `/carousel/<slug>` serve the same page on that shape; it moves itself to `/studio/` |
| Store | `Scripts\poems\studio_store.py` → `E:\Media\Poetry\studio_settings.json` (poem → `shared` + one bucket per shape) |
| Routes | `/api/studio/{poem,save,reset,export}` in `poem_studio_server.py` (the old `/api/save`, `/api/carousel/*` still work, same store) |
| AHK | `OpenPoemStudio(id, shape)`; `OpenPoemPrintStudio(id)` / `OpenPoemCarousel(id)` are one-line wrappers. Miller + library rows "Print studio…" / "Instagram carousel…" open it on that shape (`poem_library.py`) |
| Context | `poem_studio`: title ends " — Studio - Pages" (regex still accepts the old " — Print / Carousel") |
| Retired | `print_studio.html`, `carousel_studio.html` (copies in `E:\Media\Poetry\_backups\old_studio_pages\`); `print_settings.json` + `carousel_settings.json` migrated (backups `_backups\*.premerge-20261103-*.json`, originals left in place, no longer read) |

- **Shape switch** at the top (section "Shape", ← → or voice **4** Print / **5** Instagram).
  A shape = an entry in `studio_store.SHAPES` + an engine shape (`poem_layout.js` SHAPES) +
  an export — the 3:4 slide and the art poster are meant to be added that way.
- **Linked vs per shape.** Linked (`SHARED_KEYS`): font, alignment, turned-line style + per-line
  styles, line splits + split mode, showing the title, showing the byline, the small-text
  swap. Per shape: size, margins, top + bottom, line/stanza spacing, pages/slides, paper +
  grain, images, room for images, **title SIZE** (a pt that suits Letter is a third of what a
  1080px slide needs, so only show/hide is linked). Linked sections say "⛓ all shapes".
- **Linking rule (`studio_store.save`)**: a shared choice she CHANGES becomes the poem's linked
  value and any shape's own copy of it goes; one she didn't touch is left exactly as stored —
  so saving the slides never quietly rewrites the print. The migration put a shared key in
  `shared` only when both old stores agreed; otherwise it stayed in its shape's bucket (no
  output changed) and links the first time she changes it. "Back to defaults" clears this
  shape's own settings AND the linked choices; the other shape keeps its own.
- **Everything both pages did is on it**: print's hand line splits, pages, small-text font,
  title size, version mark (print only), printer/PDF, Settings for every poem (print only);
  the carousel's Layout **1** / Background **2** (slides only — print paper stays white) /
  Images **3** option screens (`option_grid.js` + `poem_variants.js`, affinity ranking),
  paper grain, PNGs. Print got the option grid too. Pictures use `clip_picker.js` on both
  shapes (the print studio's inline picker is gone); a new picture is placed under the text,
  then the placement options open (Esc keeps it). After a pick the next aspect's button
  takes the keyboard focus.
- **Export row**: Print · PNGs · Both, then Print on parchment · Open PDF · Back to defaults.
  The shape on screen goes out as it looks now; the other shape as it is saved
  (`/api/studio/export`, what = print | pdf | pngs | both).
- Fits her 2000×990 window on both shapes with nothing scrolling (3 columns, checked in the
  built-in browser).
- **A new carousel still starts on a random favourite paper**, stable per poem
  (`carousel.starting_paper`) — that is the Instagram shape's base.

### Turn, flip and fine-move a picture (BUILT 2026-11-03)

Her ask: flip a picture, full rotation, and arrows / + − in very small steps "so that I
could theoretically do it all without the touchscreen". Image fields `rot` (degrees
clockwise about its centre), `flipX`, `flipY`; x/y/w stay the UNROTATED box.
Engine: `PoemLayout.imgTransform(im)` (CSS; '' when untouched, so every existing page
renders byte-identical, checked over 482 poems x 2 shapes) and `PoemLayout.imgBox(im)` =
the box it really covers, which text-around-images wraps and the resize handle sits on.
Studio, a selected picture: a toolbar just ABOVE it on the page preview (never over the
panel), voice-numbered: rotate 15 / 1 degree both ways, Straight, 90 degrees, Flip left/right,
Flip upside down, size -5 / -1 / +1 / +5 %, remove. Keys: arrows 1 px (Shift 10), + / -
1% (Shift 5%), [ / ] 1 degree (Shift = { } 15 degrees), H / V flip, Del, Esc. PNG, print
PDF and slides PDF all draw it (same render). Test `test_a_picture_can_turn_and_flip...`.

### Byline: under the title or at the end (BUILT 2026-11-03)

Her ask: "I should be able to move the byline to the beginning or the end" (she could
in the old poster, not in the new studio). Engine option `bylineAt` ('top' | 'end') was
already there as a SHAPE default (Letter = top, slide = end); now it is an option she
sets: Byline section → **Where: Under the title · At the end**. Per shape (the two
shapes start in different places); show / hide stays linked. `bylineAt` is in
`poem_print.OPT_KEYS`, so both shapes store it.

### One folder per poem (BUILT 2026-11-03)

Her ask: a picture she placed and moved must be "permanently linked with this poem", so she
can grab the image on its own, with the PNGs "saved alongside the poem and everything".
`Scripts\poems\poem_assets.py` → `E:\Media\Poetry\poems\<slug>\` (setting
`poems.poem_folder_root`):
- `poem.json`: title, version, text, the studio settings as stored (shared + each shape),
  every picture (id, title, variant, the copied file, its original, where it sits per
  shape: page / x / y / w / every / wrap, and for online pictures the source, licence,
  page url, creator), and the exports present.
- `pictures\`: a COPY of each placed picture: `<id>.<variant>.png` (as on the page) and
  `<id>.original.<ext>`. Copied once (same size = left alone).
- `instagram\01.png…` (carousel export now writes here), `<slug>.pdf` (the print PDF now
  writes here), `<slug> slides.pdf`.
Refreshed after every studio save (a thread off the request) and every export. The
store (`studio_settings.json`) is still the truth; the folder is the poem's durable,
self-contained copy. The old `carousel_out_dir` / `print_out_dir` settings are retired
(undeclared); Assemblage's slides moved from `instagram\assemblage\` to its folder.

### Print the slides on paper, background off (BUILT 2026-11-03)

Her ask: "I just made this Instagram slide, which I think is perfect, but I also want to
print it as a normal poem … without the background." Instagram shape, Export: **Print slides
on paper · PDF of slides**, and **Background on paper: Off · On** (remembered in the page,
off by default). `carousel.py slides-pdf <id> [--background] [--print] [--parchment]`: the
slides exactly as designed, one per Letter page, 7.5in wide and centred, as VECTORS (sharp
text, not the PNG blown up); background off = white paper, no grain. Route:
`/api/studio/export` what = `slides` | `slidesPrint`. (The Print shape is still its own
design; this prints the slide design.)

### Finished images: Mark finished, Mark posted, the gallery (BUILT 2026-11-03)

Her ask: "Post to Instagram" marks the poem in her catalog as posted; a place in the
studio to browse every posted flyer and every one that is done editing but not posted;
"Mark finished" / "Mark posted" buttons, "not in terms of the poem itself ... in terms
of the image flyer."
- `Scripts\poems\flyers.py`: marks on the poem's catalog record, `rec["flyers"][shape]`
  = finished / posted / posted_to / posts[] / dir. NOT the poem's status; never touches
  its text, `modified` or the Google Doc. CLI `mark | show | list`.
- Every mark COPIES the images as they are: `<poem folder>\marked\instagram-finished\`,
  `instagram-posted-<YYYYMMDD-HHMM>\` (one per post), `print-finished\` (the PDF + a PNG
  per page). Later layout edits never change what the gallery shows. Unmarking drops the
  mark, keeps the copies. Posting also sets finished; unfinishing also unposts.
- Studio section **Finished post / Finished print**: Mark finished · Mark posted
  (Instagram only) · All finished images…; each turns into Unmark. Marking ON exports
  that shape first with the live options. "Post to Instagram" marks posted (account =
  `poems.instagram_account`) when pressed, as she asked; it still stops before Share.
- `/flyers` (studio/flyers.html): tabs Posted · Finished, not posted; a paged grid
  (no scroll at 2000x990); Enter = every slide large, ← → slides then the next image,
  S studio, F folder, Esc back; Tab switches tabs. Routes `/api/flyers`,
  `/flyer/img/<slug>/<sub>/<file>`, `/api/flyers/open`, `/api/studio/mark`.
- Catalog: the row detail says "posted to Instagram" / "Instagram image finished", the
  panel meta "Instagram image: posted Nov 3, 2026 (@pollens_poems)", and the library
  root has "Finished images…" (page only). poem.json carries `flyers`.

### In progress, and three tabs (BUILT 2026-11-05)

Her ask: a "Mark in progress" button -- for the image, not the poem -- and sections for
the in-progress images, the finished-but-unposted ones, and every posted one.
- `flyers.py`: a third mark, `in_progress` (ISO time). Marking finished clears it;
  marking in progress clears finished. A posted image stays posted, so one being
  reworked shows under In progress AND Posted. In progress is NOT copied: the gallery
  shows the CURRENT exports (`instagram\*.png`; for print, page PNGs of `<slug>.pdf` in
  `marked\print-progress\`, remade when the PDF is newer). `entries()[].tab` =
  progress | finished | posted.
- Studio: the section is now "Instagram image" / "Print image", ONE row of toggles:
  In progress · Finished · Posted (lit ✓ = marked, press again to unmark; spoken
  names "Mark in progress" etc. via aria-label) · All images…. One row because the
  three columns are full at 2000x990.
- Same day, to stay scroll-free: Byline's two rows became one (Show · Hide · Under
  title · At the end) and the 3-column gap between sections went 7px -> 5px (Print's
  middle column was 2px over).
- `/flyers` is now "Poem images": tabs In progress · Finished, not posted · Posted
  (voice 1 2 3; Tab / Shift+Tab). Catalog rows say "Instagram image in progress".

### Every version kept; PNGs open as just the image (BUILT 2026-11-05)

Her ask: "every time I mark a poem finished or save the PNG, I want to save the
versioning so that if I do it five times, I can look through the folder and see every
version", and a PNG export should open "the PNG as a tab in my Chrome, showing just the
image as large as possible", not Explorer.
- One folder per poem: `<poem folder>\versions\<YYYY-MM-DD HH-MM-SS> <shape> <reason>\`,
  reason = export | finished | posted. Never overwritten (two in one second get " 2").
  `flyers.save_version()`; every PNG export (studio Export → PNGs / Both) and every
  Mark finished / post makes one. The old `marked\` folder is gone (nothing was marked
  yet); print's in-progress page PNGs moved to `<poem folder>\print-pages\`.
- `/view/<slug>/<sub>` (studio/png_view.html): black page, the slide as large as fits,
  ← → / click halves step slides, "1 / 2" corner count. The studio opens the tab
  ITSELF at the click (window.open before the export runs; a tab opened after the
  await would be blocked as a pop-up) and points it at the new version. If that tab
  can't open, the server opens it (`_open_in_front`: os.startfile + macro bridge
  ext.focus, since a tab opened from outside Chrome lands behind the current one).

### Turned lines, one line at a time (BUILT 2026-11-03)

Her ask: a poem can mix "with /" and "new line". Engine option `turnStyles: {lineText:
'slash'|'plain'}` (keyed by TEXT like `splits`) overrides `turnStyle` for that line;
`R.turned[i].style` / `.ownStyle` say what each got, segments carry `plain`. In the studio:
click a turned line (yellow) → **/** (or T) flips that line, ← → still moves its split, Del =
automatic for both, Esc done; the "Turned lines" section shows **This line: With / · New line ·
Like the rest**. Linked across shapes.

**Checked:** all 482 poems × print + carousel render BYTE-IDENTICAL from the old engine + old
stores vs the new engine + merged store (964 renders; old engine copy `%TEMP%\poem_layout_premerge.js`); 41 tests in
`test_poem_print.py` (per-line styles, render-unchanged default, boxes, the store's linking /
reset / migration, both modules on one store).

## Instagram caption + real hashtags (BUILT 2026-11-03)

Studio, Instagram shape → **Caption** section. Top to bottom: `CW: …` (picked from a list,
her earlier ones first) · her note · title (Show/Hide) + poem · thanks (ONLY what she writes —
she asked for no default) · hashtags. Live count against Instagram's 2,200, red when over.
Every change writes `<poem folder>/instagram/caption.txt` (UTF-8, LF), as does every PNG
export — the file `InstagramPostPoem` pastes; **Post to Instagram** runs it (stops before
Share). Note + thanks are written in the kit writing box (`/kit/writing-box.js`, Ctrl+Enter
keeps, Esc puts away, the draft survives). Engine `Scripts/poems/caption.py`; fields in
`studio_settings.json` → poem → `caption`.

**Hashtags are never invented** (her rule: "a real list … I don't want to just make them up").
A first version asked the local model for hashtags; it was dropped the same day.
- `Scripts/poems/hashtags.py` — the LIBRARY. Styles in `INIDATA/PoemHashtags/styles.json`
  (Poem, Queer › Sapphic / Lesbian / Trans, Love, Heartbreak, Grief, Sad, Mental health,
  Healing, Body, Disability, Nature, Political, Memory, Short, Free verse, Prose, Funny), each
  with seed hashtags + `match_tags` (her poem tags that pre-tick it). Scrapes
  best-hashtags.com (robots allow `/hashtag/`; 1.6 s apart; cached on
  `E:\Media\Poetry\hashtags\cache`, refreshed after `poems.hashtag_rescrape_days`): each seed's
  "top 10 used on the same posts" with share %, its best sets, and size tables; then each
  candidate's own page (its size, and whether ITS posts carry the seed). General poetry tags
  are discounted inside other styles (the Poem style carries them). Output
  `INIDATA/PoemHashtags/library.json`: style → ranked tags with post count, size band and
  the evidence for each. Junk goes in `block` (`hashtags.py block <tag>`).
- `Scripts/poems/ig_sample.py` — the CHECK: each style's tag page opened briefly in her
  Chrome through the AHK Macro Bridge (`ext.readtab` active, `list` the post links, close),
  then ~18 top posts read in the hidden worker window (`ext.fetchpage` → og:description =
  caption); hashtags counted once per poet. `→ INIDATA/PoemHashtags/ig_samples.json`,
  merged by `hashtags.py build`. Bridge gotcha: `/cmd` waits 4 s unless the body has
  `timeout_ms`.
- **Studio:** the first time a poem's caption opens (and on Post with nothing chosen) it asks
  **"What are the main styles of this poem?"** — numbered chips, pre-ticked with Poem + styles
  from her tags + the local model's guess (task `poem_styles`, cached per text; it only picks
  STYLE ids). Enter fills the closest N (`poems.caption_hashtag_count`, 20): each tag's score
  summed over the chosen styles, taken big / medium / small in a 25 / 40 / 35 mix (≥1M /
  50K–1M / <50K posts), round-robin so every style is in it. She trims; "More for these
  styles" and "Your usual" (only tags she has used) sit under it, chips show post counts.
- **Adding a style:** the picker's "＋ Style" (name + starting hashtags, scraped in the
  background, ~1 min), or `py Scripts/poems/hashtags.py add-style "Witchy" witchypoetry`.
- **Cleanup pass (2026-11-03, after simulating fills on 16 of her poems):** all 20 styles
  sampled on Instagram. ~60 tags blocked: global junk (`2000s`, meme / quote / support-group /
  campaign tags, too-broad `nature` `sad` `selflove` `memories`), and per-style strays (free-verse
  tags under Trans, breakup tags under Grief, haiku tags under Nature, yoga / HAES under Body).
  Memory and Body re-seeded from POETRY tags (`#memorypoetry`, `#bodypoetry`) so a remembered
  lover doesn't get `#childhoodmemories`. A sampled Instagram tag page counts as a seed even
  when best-hashtags.com has no page for it. `ig_sample.py` falls through to the next seed when
  a tag page won't load (`#lesbianpoetry` always drops the connection; `#lesbianpoet` works).
  Each style has an `about` line, which the model sees; it now picks Poem + at most 3 styles and
  keeps Heartbreak = a romance ending, Grief = a death or permanent loss. Before this it put
  Heartbreak + Grief + Sad on nearly everything.
  To re-check: pick styles, fill, read the chips; junk → `hashtags.py block <tag>`, or for a
  single style add it to that style's `block` in `styles.json`, then `hashtags.py build`.
- **5 hashtags (2026-11-03): Instagram's limit since 2025-12-18 (was 30).**
  - The fill is now **1 big + 2 medium + 2 small**. The big one is a general poetry tag from
    the Poem style (`#poetry`); the other four go to her styles, round-robin.
  - A style with nothing in a size fills its slot with its own next-best tag, never with
    another style's. Near-twins (`#griefpoetry` / `#griefpoems`) never share the 5.
  - The picker pre-ticks Poem + at most 2 styles (her tags' styles first, then the model's
    guess, which is now told 1–2). It warns past 3 styles.
  - The count reads `n / 2,200 · n / 5 #`. Past either limit it turns red, Post greys out,
    and Copy refuses. Hashtags typed into her note count too. The server refuses an
    over-limit post as well.
  - When all 5 are in, the spare list is "Swap in": tap a spare, then the tag it replaces.
  - Setting `poems.caption_hashtag_count` is 1–5, default 5. The model's cached guesses
    carry `GUESS_VERSION` (caption.py); bump it when the `poem_styles` prompt changes.
- **Spending the 5 well (2026-11-03, measured on Instagram):** ~24 top posts read per tag.
  Median likes of a tag's top posts: `#poetry` 12,000, `#poetrycommunity` 6,451,
  `#transpoetry` 2,897, `#queerpoetry` 605, `#queerpoet` 119, `#nonbinarypoetry` 99,
  `#nonbinarypoet` 48, `#queerpoetics` 41, `#sapphicpoetry` 41, `#queerpoetsofinstagram` 31.
  A small account is seen only in the small ones. So:
  - **No big slot.** The mix is 2 medium + 3 small. Big general tags are never offered as
    spares either.
  - **Standing tag `#queerpoet`** on every post (setting `poems.standing_hashtags`, counts
    toward the 5). About half its top posts are queer poets' poems that aren't about being
    queer: tagging the poet, not the poem, is normal. It doesn't block `#queerpoetry`
    (different grid); the near-twin rule only applies between filled tags.
  - **Polluted, blocked:** `#transpoet` and `#transpoets` (mostly trucks / "transport"),
    `#transpoems` (general poetry).
  - **Checked by hand and added** (a style's `checked` list in `styles.json`):
    `#nonbinarypoetry`, `#nonbinarypoet`, `#queerpoetics` under Trans;
    `#queerpoetsofinstagram`, `#queerpoetics` under Queer.
  - **Search words** (caption field `keywords`, up to 6): a plain line such as
    `grief · a dog · winter` just above the hashtags. Instagram searches caption text, so it
    finds readers without spending a hashtag. Picked from the poem's tags, its styles and
    words used before (`caption.word_choices`).
  - Mosseri (2025): hashtags help search and labelling, not reach. They decide where the
    post can be found, not how far it goes.
- **Standing tags now `#queerpoet #nonbinarypoet`** (Jamie, 2026-11-03). Poems filled earlier
  keep their old tags until refilled.
- **When to post (researched 2026-11-03; she schedules, Gainesville = Eastern):**
  - Default slot: **Tue / Wed / Thu at 8 pm ET**. Second choice: **Sunday 8–9 pm ET**.
    Avoid Friday and Saturday.
  - Basis: Buffer (9.6M posts) has Wednesday best, Wed 6 pm in its top three, and Fri/Sat
    weakest. Sprout's windows run Tue 1–7 pm and Wed 12–9 pm. Sociality (335K posts) finds
    8 pm – midnight best, Sunday night the most consistent. No study is about poetry;
    evening suits slow, reflective reading.
  - Timing matters less than consistency (2–3 posts a week, same slots) and the first hour:
    Mosseri's top signals are likes per reach and sends per reach, and early responses decide
    whether a post reaches non-followers. Schedule when she'll be around to answer comments.
  - Replace all of this with her own data once pollens_poems is a professional account with
    100+ followers: Insights → Total followers → Most active times. Until then, alternate
    Wed 12 pm and Wed 8 pm for ~6 weeks and compare reach.
- **Poetry-only (2026-11-03, Jamie: every hashtag "very much poetry-related"):** setting
  `poems.hashtags_poetry_only` (on by default). The fill and the swap spares only take tags
  with a poetry / writing word (`hashtags.POETRY_WORDS`), e.g. `#transpoetry` not `#trans`,
  `#bodypoetry` not `#bodyacceptance`. Tags she adds herself are never filtered. Body, Memory
  and Disability have just one poetry tag each, so their fills lean on general poetry tags.
- **"It asks when a poem is decided queer":** hand-tagging a poem queer / sapphic / lesbian /
  trans (or their `match_tags`) while it has no styles yet marks it `ask_styles`
  (`poems.add_tags` → `caption.on_tags_added`). The studio then opens the style picker on
  its next visit, in either shape, and asks once (closing it clears the mark). It also asks
  on Post when no hashtags are chosen.
- **Composer check (2026-11-03, assemblage):** the real Post flow put the caption into
  Instagram's composer with stanza blank lines, the `———` lines and the 5 tags intact.
  Stopped before Share; the draft was discarded. Not yet checked: whether the PUBLISHED
  post keeps the blank lines.

## Where we are + next steps (2026-10-18, evening)

**Done today:** engine page shapes (Letter / Instagram) + marginYIn + splitStanzas + paper
grain + every-slide images + text around images + `boxes()`; the paper library (12, subtle by
construction); the carousel page + PNG export; the pick-an-option grid in BOTH studios; voice
numbers on both pages; studio in apps.json. 29 tests in `test_poem_print.py`; all 481 poems
print byte-identical to before the refactor (checked with an old-vs-new engine diff).

**Next, in order:**

1. **Jamie tries it** on a real poem: `OpenPoemCarousel` / Miller "Instagram carousel…",
   say 1 / 2 / 3, pick, Export. Tune from her picks (which options she never takes = drop).
2. **More aspects** in the same grid: Font (the print font list, side by side), Title
   (size / hidden / own slide), maybe Image style (the clipart variants: ink / cut out /
   stamp…) for a placed picture.
3. ~~Print studio picker -> `clip_picker.js`~~ — done in the merge (2026-11-03).
4. **Art poster** -- a third page shape on the same engine; the grid + images() already do
   most of it (add bleed, opacity, rotation, layering).
5. **More subtle papers** if she wants them (ambientCG has more; `paper_textures.py import`).
6. **Retire poem-poster** (unchanged from before).
7. Small polish: voice phrases ("poem carousel", "print this poem"), Stream Deck button.

**Known facts to not re-learn:** stale-code -- the studio server holds poems.py /
poem_print.py / carousel.py / clipart_index.py / clip_search.py in memory:
`poem_studio_server.py restart` after editing them (pages + JS are read per request; the
browser may cache /static/*.js -- reload). CLIP warms ~25 s after a restart.
**This repo's worktree sessions can't edit the main folder with Write/Edit** (the desktop
app's worktree guard); start studio sessions WITHOUT a worktree.

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
