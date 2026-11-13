---
tags: [programming, poetry, clipart, line-art, search, research, todo]
created: 2026-11-06
status: A + B built 2026-11-09; C (the archive) and the variant cleanup built 2026-11-10; keyed sources and book pages still open
related: ["[[POEM_PRINT_AND_DESIGN]]", "[[POETRY_SYSTEM]]", "[[ART_SYSTEM]]"]
---

# More and better line art: where the studio finds pictures

**The ask (Jamie, 2026-11-06):** "How do we increase the available searches for where we
are getting our line art from, not just specifically for marionettes but also in general?
What might we have access to that could drastically increase the quality of a search ...
so that when I search something like marionette, I'm getting the best possible quality and
quantity of diverse sources (with a focus on line art, like the kind of stuff I would want
to put over a poem)?" She wants the **options laid out first**, not built straight away.

Lives on her Poetry stack (systems, s-8dae7058). Part of [[POEM_PRINT_AND_DESIGN]]
(§ "Clipart search + picker", § "Images, checking and the panel").

## What she wants from a picture

Black-and-white **line art**: engravings, woodcuts, etchings, pen drawings, botanical and
natural-history plates, old book illustrations, patent drawings. Classy, natural, physical
objects, not goofy and not photos. It sits over a poem as **Ink** (paper made transparent,
lines kept), so a clean scan on plain paper matters more than resolution. It has to be
**public domain or CC0**, so it can go on a print or on Instagram with no attribution to
manage.

## What the studio searches today

| Where | How | Notes |
|---|---|---|
| Her library | ~3,200 pictures on E: (`E:\Media\Poetry\poster\clipart\_samples`), tags + title words + CLIP (`clipart_index.py`, `clip_search.py`) | Instant, offline, all Ink-processed. Most were collected in bulk from Wikimedia Commons categories. |
| Wikimedia Commons | MediaWiki search: "q engraving", "q illustration", "q" + old-caption synonyms | PD / CC0 only, ≥300px, bitmap only (PDF/DjVu book scans are skipped). |
| Openverse | cc0 + pdm; line-art call limited to museum/archive sources | Anonymous: 20 requests/min, 200/day. |

On 2026-11-06 `clip_online.py` gained: CLIP re-ranking of thumbnails by what they show
(line drawing vs photo), puppet/marionette synonyms ("wire master", "puppet master",
fantoccini), and HTML-free Openverse titles.

## What the marionette research found (2026-11-06)

- **The words don't match.** Old captions say "wire master", "puppets", "fantoccini",
  "marionnettes", not "puppet control bar". The best hit (British Museum 1767, *The Wire
  Master, and his Puppets*) only came up by luck.
- **The best material is inside books and patents.** *The Tony Sarg Marionette Book*
  (1921, archive.org `tonysargmarion00mcis`), *A Book of Marionettes* (1920, Gutenberg
  66391), and US patents 2,860,446 / 2,662,338 / 4,179,843. Commons only has these as
  PDF/DjVu, which the search skips.
- **Openverse is mostly photos and silhouettes** unless filtered.
- **No search looks at the picture**, which CLIP re-ranking now partly fixes.

## Options

### A. More sources with free APIs (searched live, like Commons)

| Source | Access | Strength for line art |
|---|---|---|
| **Art Institute of Chicago** | JSON API, no key, `is_public_domain`, IIIF images | Prints and drawings dept; CC0. |
| **Cleveland Museum of Art** | Open Access API, no key, CC0 | Prints and drawings, well tagged. |
| **The Met** | Collection API, no key (`/public/collection/v1.1/search`, check `isPublicDomain` per object) | Huge print collection; slow (one call per object). |
| **Library of Congress** | loc.gov JSON (`fo=json`), no key; keep "No known restrictions" | Cartoons, engravings, book illustrations, patents-era. |
| **Wellcome Collection** | Catalogue + IIIF API, no key | Thousands of PD engravings (anatomy, hands, plants, animals, objects). |
| **Biodiversity Heritage Library** | Free API key | Botanical and natural-history plates: exactly her "naturey" taste. |
| **Smithsonian Open Access** | api.data.gov free key | CC0, includes Cooper Hewitt drawings. |
| **Rijksmuseum** | Free key | Etchings, prints; CC0. |
| **NYPL Digital Collections** | Free token | Book illustrations, prints; "public domain" filter. |
| **Europeana** | Free key | Aggregates hundreds of European museums; PD filter. |
| **Internet Archive** | advancedsearch + page images, no key | Scanned illustrated books (Sarg, nature guides): needs page-level picking. |
| **Flickr Commons** (British Library 1M images, Internet Archive Book Images) | Flickr API key | Millions of book illustrations, tagged by volunteers. |
| **Google Patents** | Per-patent drawing PNGs | Clean mechanical line drawings of objects. |
| **Openverse with a key** | Free registration | Raises the 20/min / 200/day limit. |

### B. Make every search smarter

- **Query expansion by the local model** (`llm_gateway`): turn "marionette" into period
  words (puppet show, fantoccini, Punch and Judy, wire master) plus French, German and
  Italian, generally rather than a hand-written synonym table.
- **A line-art filter:** score each result for "is this line art" (CLIP plus a cheap pixel test: few
  gray tones, high edge density, white background) and hide photos and colour paintings by default.
- **"More like this":** CLIP image-to-image from a picture she likes, across her library
  and online.
- **De-duplicate across sources** by perceptual hash; the same print is on Commons,
  the Met and Europeana.
- **Search inside scanned books:** find illustrated pages (CLIP / layout detection), crop
  the plates.

### C. Grow the local library in bulk (fast, offline, already Ink)

Pull curated public-domain line-art sets onto E: ahead of time (CLIP-embedded and
Ink-processed), so every search is instant and has depth. Candidates: Wellcome engravings,
BHL plates, British Library Flickr illustrations, Old Book Illustrations, AIC/Cleveland
prints and drawings, patents by subject. Storage is on E:
([[feedback_external_drive_storage]]). Probably the biggest jump in **quantity**.

### D. Paid / other (for completeness)

Rawpixel premium PD, Dover clip-art books (royalty-free within limits), stock vectors. Also
AI-generated line art as a last resort. Probably not her taste or licensing comfort; ask
first.

## How to judge it

A fixed test set of her real queries (marionette, hands, moth, key, ladder, fig, knife,
string, bird, door…). Run every source and count **good line-art results in the top 20**
(CLIP line-art score + `layout_check`-style Ink check). Compare before and after each
change. This is the same idea as her "debugging harness" ask: measure, don't eyeball.

## Built 2026-11-09 (first round)

All in `Scripts/poems/studio/clip_online.py`; the picker is `clip_picker.js`.

- **Four more live sources, no keys:** Wellcome Collection, Art Institute of Chicago,
  Cleveland Museum of Art, Library of Congress (only "No known restrictions"). AIC only
  serves 200/400/843px and blocks browsers with a Cloudflare check, so its thumbnails go
  to the picker inline and downloads are 843px. LoC is often slow: 8 s, best effort.
- **Words from the local model** (not `llm_gateway`, which is billed now): task
  `clip_query_words` in `Scripts/local_llm/tasks.json`, qwen 14B, ~3 s the first time.
  Saved for good in `E:\Media\Poetry\poster\online_query_words.json`; fix a bad
  one by hand there. The picker's footer shows "also searched: ...".
- **Line art or not**, by what the thumbnail shows: CLIP picks between line drawing /
  photo / colour painting / text page / poster, plus a colourfulness number. Not-line-art
  is folded behind "Show N that aren't line art"; "Download all" and "Get more" skip it.
- **Same print twice** across sources is dropped by a picture hash.
- **Measuring:** `py clip_online.py bench [words] [--old]`, saved in `online_bench\`.

| 10 test queries | Before | After |
|---|---|---|
| line art found per search | 14 | 80 |
| line art in the top 20 | 14 | 20 |
| line art in the top 50 | 14 | 50 |
| seconds per search | 4.6 | 7.5 |

Wellcome and AIC bring the most line art; Openverse brings almost none (mostly photos).

Not done yet: keyed sources (BHL, Smithsonian, Rijksmuseum, NYPL, Europeana, Flickr
Commons, Openverse key), "more like this", searching inside scanned books / patents, and
**C, the bulk local library**, which is the next big jump.

## Since then (2026-11-10)

- **Online search runs on every search**, about a second after she stops typing
  (`poems.clip_online_below` = 200; the library never returns more than 120). Results sit
  under her own pictures as before.
- **Ranking is by subject**: "a drawing of a <q>" plus a small bonus when the title names
  it. Being an engraving is already handled by the line-art check; scoring it twice put
  Art Institute ships above keys for "key".
- Slow-source timeouts (LoC, Cleveland) no longer show in the picker footer.

## C, the bulk local library: decided and built 2026-11-10

The constraint: **E: has 56 GB free (95% full)** and the clipart library already takes
**28 GB for ~3,200 pictures** (~9 MB each), because every picture is kept as the original
plus 4-5 processed copies (alpha-light, alpha-dark, otsu, rembg/cutout).

Her answers (numbered as asked):

- **1 yes, preview-only library.** Per picture keep a ~15 KB preview, its CLIP vector and
  title / tags / source, NOT the image. ~60,000 pictures = about 1 GB. Search is instant
  and offline; picking one downloads the full image and makes the Ink copies (~10 s, as
  online picks do now), after which it is an ordinary library picture.
  (2 = originals only, 3 = everything as now: no.)
- **4 yes, measure first:** find which processed copies the studio ever opens. Probably
  over 10 GB to free. Nothing is deleted without her OK.
- **5-8 yes, first sources:** Wellcome engravings (~23,000; 75,000 PD images in all), Art
  Institute prints and drawings (~27,000), Cleveland prints and drawings (~13,000), the
  rest of Old Book Illustrations. About 60,000 in, an estimated 35,000-40,000 pass the
  line-art check.
- **9 later:** BHL plates, British Library Flickr (1M), Smithsonian Open Access.
- **10 yes, automatic filtering only:** line-art check, no blank or text pages, nothing
  already in the library; she hides bad ones over time with a "not this" key.
  (11 = hand review: no.)
- **12 yes, its own picker section**, "From the archive", between her library and Online.
  (13 = mixed in: no.)
- **14 yes:** a stoppable, resumable background harvest, overnight, tooltip when done,
  voice to pause / resume. About 2-3 hours for 5-7.
- **15 yes:** "more like this" (image-to-image over the vectors) in the same session.
- **16 later:** cropping plates out of scanned books and patents.

Estimate: one 3-4 hour session. Touches `clip_online.py` (fetchers already exist),
`clipart_index.py` / `clip_search.py` (a second, preview-only index), `clip_picker.js`
(the new section), and a harvest script beside them.

## How C was built (2026-11-10)

- **Engine:** `Scripts/poems/studio/clip_archive.py` (store, harvest, search, more like
  this). **Routes:** `archive_routes.py`, mounted by the studio server. **Picker:**
  `clip_picker.js`, the "From the archive" section. **AHK:** `ArchiveHarvestStart`,
  `ArchiveHarvestPause`, `ArchiveHarvestStatus` in `Helpers/PoemsFunctions.ahk`.
- **On E:** `E:\Media\Poetry\poster\archive\` (records.jsonl, vectors\, previews\,
  seen.txt, hidden.json, harvest_state.json, harvest.log).
- **Harvest:** 47 parts (13 Wellcome technique words, 24 Art Institute date slices of 25
  years, Cleveland prints and drawings, 9 Old Book subjects), about 16 pictures a second
  looked at, ~85% kept as line art for Wellcome engravings. Previews ~14 KB each.
  Resumable; one at a time; tooltip at the end.
- **Search** is instant (one matrix multiply over the vectors in the server). It shows
  only pictures within 0.06 of the best match, so a poor match shows few.
- **More like this** works from any tile: archive (its vector), library (the library's
  CLIP vectors), online (the thumbnail embedded on the spot).
- **Not this** hides a key for good, archive and online alike (hidden.json).
- **Tests:** `Scripts/codebase_tools/tests/test_clip_archive.py`.

### The space finding (option 4), measured 2026-11-10

| Copy | Size | Placed on a poem |
|---|---|---|
| alpha-light (Ink) | 11.3 GB | 15 times |
| alpha-dark (White ink) | 11.2 GB | never |
| original | 3.5 GB | once |
| rembg (Cut out ML) | 2.8 GB | never |
| cutout | small | 7 times |

White ink and Cut out (ML) are offered in the Style menu but never used. **Done
2026-11-10 (her OK):** both are now made on demand (`clipart_index.ON_DEMAND`; file_for
makes one the first time it is asked for: 0.3-0.6 s usually, ~3 s for a 36 MB scan,
+3.5 s once for the cut-out model) and their 30 folders were deleted: 13 GB freed,
E: went to 67.9 GB free. Every picture still lists both styles.

### Also 2026-11-10

- **≈ is numbered on every tile** (her: "it's fine if the numbers go up a little");
  **✕ is not numbered** (a click or X).
- **No voice phrases** for the harvest. Instead it **finishes itself**: opening the
  poetry system (OpenPoemStudio / OpenPoetryStudio / OpenPoetryStudioPath) runs
  `clip_archive.py kick`, and so does the studio server starting. kick starts a quiet
  harvest only if parts are left and none is running.
- **First full harvest finished:** 49,738 kept of 66,488 looked at (Wellcome 24,980,
  AIC 12,322, Cleveland 9,312, Old Book 3,124; 651 MB of previews). 10 AIC date ranges
  stopped at 1,000: AIC's search won't page past the 1,000th result (a 403). Those
  ranges are now cut into 5-year pieces, or single years for 1800-05 and 1890-1900
  (`_aic_pieces`, max 10 pages each), and the run went on to walk them.

## Next step (as written 2026-11-06)

Show her these options numbered (numpad), with a recommendation. The likely order is a
quick win (AIC + Cleveland + LoC + Wellcome live sources, plus LLM query expansion, plus a
line-art filter), then a bulk local library growth project.
