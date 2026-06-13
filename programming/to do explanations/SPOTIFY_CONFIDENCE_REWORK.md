---
tags: [todo, autohotkey, spotify, scraping, confidence, music-catalog, session-plan]
---

# Spotify scraping — confidence rework + next-session plan

Working plan for the next session (2026-06-12). Headline: replace the scattered
confidence **gates** with a single **unified scoring model** where signals work in
concert. Plus the data-completeness and UX threads that feed it. Builds on
[[SPOTIFY_SCRAPING_INTERNALS]], [[TRIAGE_AUTO_RESOLVE]], [[MILLER_TREE_ANALYZER]].

## Status (2026-06-12) — evaluator + pagination DONE

- ✅ **A. Pagination scroll-and-accumulate** — built last session as a UIA
  `ScrollPattern` engine (`_SpotifyDiscographyScrollDumps`): scrolls the
  virtualized list directly, uses the real scrollbar position
  (`VerticalScrollPercent` ≥ 99.5) as the done-signal, single-dump fast path
  when there's no vertical scroll (<~21 albums), Esc-cancellable,
  Spotify-page-guarded, per-step vPct logged. Dumps union by album AID
  Python-side (`_albums_union_from_dumps` + `--extra-dumps`). Wired into both
  the batch path (`_PrefetchOneQueued`) and on-demand
  (`RescrapeArtistFullDiscography`). Proven: Grateful Dead 0→100% / 15 windows /
  174 albums; Chopin 24→648.
- ✅ **Unified evaluator** — `prefetch.evaluate_artist(evidence)` is live and
  replaces `compute_confidence` (now a thin shim), the separate
  `confirmed_ogtitle` marking, and the wrong-page name-Jaccard HARD reject.
  Signals **combine** additively into one 0..1 confidence (`CONF_BASE=0.70`),
  thresholded `T_SAVE=0.62` / `T_REVIEW=0.38` → `save` / `review` / `reject`.
  Returns `{level, decision, confidence, signals, reason, reasons, ...}` — a
  superset of the old dict, so the AHK review UI keeps working. Signal weights:
  overlap `confirmed +0.30` / `wrong_artist −0.45` (×0.35 if deep, ×0.30 if
  classical, ×1.5 if sparse); album_count `sparse −0.25` (suppressed for
  classical) / `deep +0.15`; `ogtitle match +0.35 / mismatch −0.20`;
  `page_name_mismatch −0.40`; soft nudges for lastfm/mb/wiki/related; voice-
  phrase collision −0.20 (also forces ≥ review). The ONLY hard short-circuit
  left is blank-discography (a genuine scrape failure), upstream in
  `prefetch_artist`.
- ✅ **D. Jay-Z name restyle** — `_name_jaccard` now folds diacritics (NFKD +
  drop combining marks), so "Jaÿ-Z" vs "Jay-Z" = 1.00 and never trips the
  wrong-page guard. For folds it can't catch (e.g. "P!nk" vs "Pink"), the
  evaluator's og:title-match signal rescues the name mismatch.
- ✅ **Calibration** — `Scripts/MediaCatalog/confidence_calib.py` is a runnable
  self-check; the corpus below all passes (run after changing any weight).
- ⏳ **Still pending:** **C** post-batch triage view, **B** classical album
  *parsing* (the evaluator already down-weights classical; parsing would fix the
  under-count at the source so a real Rachmaninoff reads deep, not sparse).

## Why this is needed (the problem Jamie named)

Right now "is this the right artist, and how good is the data?" is decided in **3+
disjoint places** with different logic, and they don't talk to each other:

- **`prefetch.prefetch_artist`** — hard short-circuits: blank discography (fail),
  wrong-page name-Jaccard < 0.5 (fail), and the `confirmed_ogtitle` branch that marks
  `needs_review` SEPARATELY from the score.
- **`prefetch.compute_confidence`** — its own pile of `hard_blockers` + soft `reasons`
  + roll-up, plus the new album-count heuristic (which only acts HERE, so it missed the
  `confirmed_ogtitle`-flagged Rachmaninoff / Bob Marley this round).
- **`lastfm_data.album_identity_overlap`** — the play-weighted verdict, consumed as a
  boolean by one of the above.

Result: weird gates that fight each other. The album-count rule helps in one path but
not the path that actually flagged the classical artists. A signal that should *combine*
(low overlap BUT deep discography BUT og:title match BUT classical genre = probably
fine) instead hard-blocks or double-flags.

## The target: one unified evaluator

A single function takes ALL evidence gathered once and returns a single confidence with
an explanation:

```
evaluate_artist(evidence) -> {
    confidence: 0.0-1.0,
    decision:   "save" | "review" | "reject",
    signals:    [ {name, value, weight, contribution} ],   # sorted by |contribution|
    reason:     "<top contributing signals, human-readable>",
}
```

**Evidence bundle** (collected once in `prefetch_artist`, passed in whole):
scraped album titles + count, page-artist name, og:title verify result, play-weighted
overlap verdict+number, Last.fm bio/similar/name, MB candidates/ambiguity, Wikipedia
kind, voice-phrase collision, genre/tags (is-classical?), `resolved_via`.

**Signals combine, they don't short-circuit.** Each contributes a weighted +/- to the
score. Examples of working *in concert*:

| Signal | Effect |
|---|---|
| play-weighted overlap | strong +/- ; **neutral** when `unknown` (no scrobbles) |
| album count | ≤3 = negative MULTIPLIER on negatives; ≥20 = DAMPENS an overlap-negative |
| og:title match | strong + ; can **rescue** a name-Jaccard fail (Jay-Z) |
| name-Jaccard | soft − (NOT a hard reject) — overridden by og:title |
| genre = classical | **down-weights** the album-overlap signal (wrong signal for classical) |
| bio / similar / MB / wiki | minor data-quality nudges |
| blank discography | the ONLY true hard short-circuit (genuine scrape failure) |

**Decision thresholds:** `confidence ≥ T_save` → auto-save; `T_review ≤ c < T_save` →
save + flag for triage; `c < T_reject` → reject (wrong page). The `reason` string is the
top contributing signals, so triage shows *why* — no more opaque "kept on og:title".

**What it replaces:** the `hard_blockers` list, the separate `confirmed_ogtitle`
needs_review marking, the wrong-page Jaccard hard reject (→ a strong negative signal og:title
can offset), and the album-count patch — all become weighted inputs to one function.

**Calibration corpus** (use this round's artists as fixtures to tune weights/thresholds):
- Jack Johnson namesake (1 album, overlap 0) → **reject**
- Bob Marley & The Wailers (21 albums, low overlap) → **save** (deep dampens)
- Sergei Rachmaninoff (classical, overlap fails on title basis) → **save/review**, not reject
- Jay-Z (name restyle "JAY-Z") → **save** (og:title rescues name mismatch)

## Supporting threads (data + UX that feed the model)

### A. Pagination cap — prolific artists truncate at ~21 albums
Rush = 21, Grateful Dead = 21 (identical → a virtualized-list cap). The scraper has **no
scroll handling**; it dumps the first paint (~20-21 cards). Fix: **scroll-and-accumulate**
the discography page (PgDn/End loop, re-dump, merge album AIDs until the count stops
growing). Caveat: Spotify's list is **virtualized** — cards unmount when scrolled off —
so accumulate DURING the scroll, not one dump at the end. Improves every downstream
signal (a complete discography matches scrobbles far better).

### B. Classical album parsing — Rachmaninoff saved 3 but the dump has dozens
Separate from pagination: the data is in the dump but the parser keeps only 3. Classical
"albums" are often credited to performers/orchestras or classified as compilation /
various-artists and fail the `kind=="album"` filter. Fix: inspect a real Rachmaninoff
dump, see how the entries are structured, and loosen the classifier for classical. Pairs
with the model's "classical → down-weight overlap" signal.

### C. Post-batch triage view — review a whole round
After a batch, a **"triage last batch"** button opens a triage view scoped to *that
round's 10* — flagged ones first, then the clean ones — each with the resolve actions
plus a **"mark reviewed"** stamp, so Jamie can sweep the entire pull (including the fine
ones). Needs: the batch records the artist ids it touched (a last-run manifest), then a
Miller variation that reads it. Reuses the triage UI + `MlActions`.

### D. Jay-Z name restyle
Rejected because the page name ("JAY-Z", restyled) didn't match the queued "Jay-Z" under
the name-Jaccard gate. In the unified model this is solved for free: name-Jaccard becomes
a **soft** signal that og:title-match **overrides**. Add name normalization
(case/punct/diacritic/hyphen folding) so restyled names compare cleanly too.

## Recommended order
1. ✅ **A — pagination scroll-and-accumulate** (better data first; everything depends on it).
2. ✅ **Confidence rework** (the unified evaluator; folds in album-count, classical
   down-weight, and Jay-Z name handling as signals).
3. ⏳ **C — post-batch triage view**.
4. ⏳ **B — classical album parsing** (improves the classical case further).
5. ⏳ Spot-check **Jay-Z** (`music:jay_z` shows 0 albums — confirm where they landed).

## Open questions for the rework
- Weights: hand-tuned from the calibration corpus, or a tiny scored ruleset we can
  adjust by eye? (Lean hand-tuned + explainable — no ML.)
- Thresholds `T_save / T_review / T_reject`: pick from the corpus, revisit after a few
  batches.
- Does the unified evaluator live in `prefetch.py` (replacing `compute_confidence`) with
  the scrape guards refactored to feed it evidence? (Yes — single home.)

See also: [[SPOTIFY_SCRAPING_INTERNALS]], [[TRIAGE_AUTO_RESOLVE]],
[[MUSIC_CATALOG_BACKLOG]].
