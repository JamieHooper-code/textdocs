---
tags: [programming, design-doc, reader, ocr, pdf, epub, zines, doctext]
created: 2026-10-16
updated: 2026-10-16
related: ["[[ZINES_SYSTEM]]", "[[READING_ROOM_BOOKS]]", "[[QUOTES_SYSTEM]]", "[[ART_SYSTEM]]"]
---

# Document text — any PDF / scan / EPUB into the reading room

Status: **built 2026-10-16.** Generic engine + reader type + zines as the first consumer. Voice: **`grab read`** in the art viewer (Jamie's pick -- the same phrase reads a grabbed chat elsewhere; `GrabChatRead` branches on the `art_viewer` context).

What Jamie asked for: a lot of zines are PDFs set in tiny type, and she'd rather read them in the reader. And: *"the plumbing for converting PDFs into OCR stuff that I will end up reading in the reader should be generic … the kind of thing that a bunch of different things can plug into."*

## The shape

```
any source                     doctext (generic)                 reader (generic)
  PDF (+ layout)    ─┐
  page-image folder ─┼─► Scripts\doctext\doctext.py ─► EDITION ─► doc:<edition path>
  EPUB              ─┘     per page: text layer or OCR   JSON       reader_collection.resolve_doc
                           reflow, columns, rotation                 AHK: OpenDocInReader[At]
                           quality score
          ▲
   a CONSUMER decides: where the edition lives, what to build it from,
   whether it is worth offering, and how she gets to it (zines: zine_text.py)
```

The **edition** is the contract. A consumer never parses a PDF; the reader never learns what a zine is. Adding a new kind of document = a new consumer (a dozen lines: pick source, pick out path, call `doctext.make`, open with `OpenDocInReaderAt`).

## The edition (`doctext/1`)

One JSON file: `title`, `creator`, `lang`, `method` (`epub`/`text`/`ocr`/`mixed`), `source` {path, kind, layout, mtime, size}, `quality` {real, words, words_per_page, pages, pages_text, pages_ocr, pages_blank}, and `sections` — each {id, ref, title, text, method, real, words, rotation}. `text` is the reader's own format: one paragraph per line, `[[h2 …]]` headings, so `blocks()` draws it with no new renderer.

`make()` **reuses** an edition of the same source (mtime + size), same `ENGINE`, made with at least the OCR permission asked for. Bump `ENGINE` whenever extraction/reflow changes output.

## Decisions, and why

- **Method is chosen per page.** Zines mix typed pages, scanned covers, outlined-text pages. Each page uses its text layer if that has ≥15 words at ≥70% real; otherwise OCR (when allowed). An edition can be `mixed`.
- **Quality = share of real words** (wordfreq, zipf ≥ 1, en/es auto-picked). Windows OCR gives no confidence and text layers can be junk too; one measure covers both and also **picks the rotation**. First scan tried (IA *Anarchism*): saved sideways → 4 words at 0°, 464 clean words at 90°. Every page tries 0° first, the other three only if 0° reads under 80% real.
- **OCR engine = Windows.Media.Ocr via `winsdk`** — same engine as the Kindle grab, already installed, ~0.3s a page. No Tesseract.
- **Reflow rule is the grab reflow's**: a wrapped line runs to the margin, so a paragraph ends on a short line that ends a sentence, a clear gap, or an indented next line (only in indent-style text). Big short lines → `[[h2]]`. Bare page numbers dropped. Wrap hyphens undone only before a lowercase letter.
- **Layout is rebuilt from WORDS, never from the engine's lines** (`ENGINE` 3). Found on the OCR test: Windows OCR glues a row straight across a gutter, so *Class War No 28* came out 96% real words and 0% readable — three columns interleaved fragment by fragment. A real-words score cannot see that; **check newsprint by eye.** Fix = recursive **XY-cut** on word boxes (`xy_cut`): split a region on a full-width horizontal gap (stacked stories, under a headline) or a full-height vertical gap (a gutter ≥0.9 word-heights, region ≥3 lines tall). Gaps come from a per-pixel coverage count that **tolerates up to 3 words across a gutter** — the curved text of a round logo was vetoing every column cut on the page. A horizontal gap ≥2.5 word-heights beats a column cut so stacked stories read one at a time. Leaves are single columns: lines rebuilt by running centre (`words_to_lines`; growing a box merged neighbours on a 1° skew and turned body lines into headings), each block reflowed with its own margins, and a paragraph running off one column onto the next rejoins (`join_blocks`). Text-layer PDFs go through the same path (`text_layer_words` via rawdict).
- **A text layer's words are never mended** (`_line_text`, size > 0). Two traps found when the word path went in (ENGINE 4): many PDFs store **no space characters**, only positions (Indigenous Action, Sonic Anarchy) — so `text_layer_words` also breaks a word at a gap > 0.25 of the font size; and OCR-style gap-joining on a text layer glued whole lines into one "word" (23% real). Faux-bold (a word drawn twice a hair apart, "TheTheNPICNPIC") is dropped. **After any layout change, diff `make --all` offer lists against the previous run** — that is how both were caught.
- **Split words mend by GAP, not dictionary.** The threshold is the page's word space *per word-height*, scaled by each line's own type size (ENGINE 5) — a page-wide pixel figure glued small-print captions ("ELNhas roundaClass Warcell"). Typewriter text OCRs as "Condi tions", "commun i tie s". Measured: a split gap ~3px, a word space ~27px — so a gap under 40% of the page's median word space is joined. The dictionary was tried first and failed: wordfreq knows "condi", "wil", "communi". Typewriter `1`→`l` / `0`→`o` is swapped only in a mixed letters+digits token when the swap is clearly commoner (`A11`→`All`; `1994`, `R2` untouched).
- **Print-and-fold**: `reading_order(doc, "saddle")` is the same arithmetic `zine_import.pdf_to_pages` cuts images with, so text page 5 = image page 5.
- **Paragraphs over page breaks rejoin** onto the earlier page (ends mid-sentence + next page opens lowercase).
- **EPUB**: reuses the reader's `book_text.chapters`. A short opening section ahead of a long one (amusewiki's title page: "Retrieved … from …") moves to the end as **About this text** — otherwise every open landed on it.
- **Item ids** `doc:<sha1(source path)[:12]>#<section id>` — highlights/grabs survive the edition being rebuilt.
- NFKC normalisation folds ligatures (`ﬀ` → `ff`).

## Zines — the first consumer (`Scripts\zines\zine_text.py`)

- Edition lives at `<zine>\_text\edition.json` — moves and deletes with the zine.
- Source, best first: **publisher EPUB** (The Anarchist Library + Biblioteca Anarquista publish one per text — all 26 of ours had one; saved into `_source\`), else the `_source` PDF with the sidecar's `layout`, else page images (OCR only).
- **Offer** when ≥85% real and ≥120 words/page (EPUB: ≥300 words — *The Fetish Speaks!* is a comic). Below that the pages are better than the text.
- **Pages stay the default** (Jamie). Switch = **`grab read`** only → `GrabChatRead` sees the `art_viewer` context → `ReadZineText`. **No key** (Jamie: letter keys are Vimium's); `/api/readtext` on art_server stays, unbound, for a future page button. ReadZineText → `zine_text.py open <current page file> --page N` → `OpenDocInReaderAt(edition, item)`, landing on the page she was on (EPUBs open at the start: chapters aren't pages).
- `open` builds a missing edition on demand but **never OCRs from a keypress**.
- New imports build their text version inline (`ingest_file` → `make_for`, failures logged, never fatal).
- CLI: `make <folder>|--all [--source s] [--ocr] [--force]`, `open`, `report`, `ocr-candidates [--n]`.

## Library survey (2026-10-16, 397 zines)

| Source | Zines | EPUB | Good text layer | Some text | Scan only |
|---|---|---|---|---|---|
| Anarchist Library (+es) | 26 | 26 | 13 | 12 | 1 |
| CrimethInc | 62 | – | 29 | 27 | 6 |
| Sprout Distro | 62 | – | 28 | 24 | 10 |
| Internet Archive | 172 | – | 21 | 36 | 115 |
| QZAP | 60 | – | 2 | 24 | 34 |
| Indigenous Action | 15 | – | 5 | 4 | 6 |

**Outcome (2026-10-16, ENGINE 5):** 192 offered from EPUB + text layers alone; then `ocr-candidates` probed one middle page of every scan, and the 136 that read ≥80% real / ≥150 words were OCR'd (10 test + 126) — 127 offered. **309 of 397 zines** now have a text version. Not offered: picture-heavy zines, and a few mixed IA items whose embedded OCR layer scores 77–84% (*Groove #81*, *Sniffin' Glue 12*). Known blemishes: running headers sometimes survive ("ANAR( )RAl.rry 1.35"), a lone page digit can read as `[[h2 1]]`.

Several CrimethInc print PDFs have their text as **outlines** (shapes, no text layer) — clean type, so prime OCR material.

## Traps

- `py -3 -I` (isolated) cannot see packages in the per-user site (`regex`, which wordfreq needs). Run doctext without `-I`.
- `art_server.py` and `read_server.py` hold code in memory — restart after editing either (`read_server.py restart`; kill the pythonw on :8288).
- Windows OCR with a `StorageFile` path wants a real Windows path; doctext feeds bytes through `InMemoryRandomAccessStream` instead, so it never touches that.
- A footnote between two halves of a sentence blocks the cross-page rejoin (seen on IA *Anarchism* p.2→3). Left as is.

## Next consumers (ideas, not built)

- PDF books in the library (`book_text.readability` refuses PDFs today) — `read:<id>` could fall back to a doctext edition.
- Scanned handouts / any "read this PDF" from Explorer.
- A Miller row "Make text version (OCR)" per zine, for the scans that aren't auto-offered.

## Files

`Scripts\doctext\doctext.py` · `Scripts\zines\zine_text.py` · `Scripts\reader\reader_collection.py` (`resolve_doc`, `DOC_TYPE`, `/docs/`) · `Helpers\ReaderFunctions.ahk` (`OpenDocInReader`, `OpenDocInReaderAt`) · `Helpers\ZineFunctions.ahk` (`ReadZineText`) · `Scripts\MediaCatalog\art_server.py` (`/api/readtext`, unbound) · tests `Scripts\codebase_tools\tests\test_doctext.py`.
