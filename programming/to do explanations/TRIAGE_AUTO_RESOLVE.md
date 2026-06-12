---
tags: [todo, autohotkey, spotify, scraping, triage, music-catalog]
---

# Triage auto-resolve & compare — design plan

Make the [[Spotify scraping pipeline]] resolve weak-identity artists itself instead of
just flagging them, then present the options for a one-keystroke human pick. Builds on
the manual triage UI (`Helpers/SpotifyTriage.ahk`) and reuses the existing play-weighted
scorer (`lastfm_data.album_identity_overlap`) + confidence roll-up
(`prefetch.compute_confidence`).

## The core idea

For a low/medium-confidence artist, **try several candidate URLs automatically**, scrape
each candidate's discography, **score each** against Jamie's scrobbles, and present a
ranked comparison. Jamie picks the winner with one digit, or shoves it to the triage
queue to decide later. Candidates:

1. **Current** saved URL (the baseline being questioned).
2. **Related-graph** URL — `recover-url` (vouched by saved artists' /related grids).
3. **Spotify-search-first** — drive Chrome to `/search/<name>/artists`, grab the first
   hit (already built: `_TriageSpotifySearchFirstUrl`).
4. *(maybe)* a fresh **MusicBrainz** re-resolve.

Each candidate yields `{url, album_count, overlap, verdict, sample_albums}`. Present them
side by side with a **confidence number each**; the winner is the highest overlap that
clears threshold.

## Album-count heuristic (Jamie's observation — bake into scoring)

Empirically: **namesake / wrong-lookup artists almost always have < 5 albums**, while
**real classical artists that fail the overlap check have 20+** (overlap is low only
because box-set / opus / remaster naming doesn't match scrobbles, not because it's the
wrong artist). So album count is a strong prior:

- **≤ 3 albums → negative multiplier.** Amplifies every other weak signal — a sparse
  discography next to any other red flag should push hard toward "wrong artist /
  remove". Treat as a multiplier on the stacked soft-signal score, not a standalone.
- **≥ 20 albums → false-positive counter-signal.** Downgrade an `album_identity_mismatch`
  HARD blocker to a SOFT "needs review" (or loosen the overlap threshold), because a
  deep discography that merely fails name-overlap is far more likely a real artist with
  messy album naming than a namesake. Don't auto-remove these.
- 4–19 albums → neutral; rely on the play-weighted overlap as today.

Implementation: thread `scraped_album_count` into `compute_confidence` and into the
`album_identity_overlap` → `identity_mismatch` decision. Keep the play-weighted overlap
as the primary signal; album count is a **modifier** that strengthens or softens it.

## Two trigger modes (Jamie wants BOTH)

1. **On-demand** — a triage row-action `N.7 = Auto-resolve & compare` on a single flagged
   artist. Runs the candidate sweep now, shows the compare picker.
2. **Batch / during initial scraping** — when the main scrape (`BatchPrefetchAndSaveAuto`)
   scores an artist **low or medium** confidence, automatically run the candidate sweep
   inline. At the END of the batch, present a **decision queue**: for each auto-resolved
   artist, show "original vs best candidate" and let Jamie **pick one** or **shove it to
   the triage queue** to handle later. So the batch ends with a short, high-signal review
   instead of a pile of silent flags.

## Compare-picker UX (the "present both, I pick" surface)

A modal (or a small Miller) per decision showing:

```
Jack Johnson — which is right?

  [1] CURRENT   2i7o…  ·  1 album   ·  overlap 0.04  ·  Wurlitzer Pipe Organ
  [2] SEARCH    3GBP…  · 16 albums  ·  overlap 0.91  ·  In Between Dreams, On and On, …
  [3] RELATED   (none vouched)
  ───
  [0] Send to triage queue (decide later)    [9] Keep current, clear flag
```

Album count + overlap + sample titles make the right pick obvious at a glance. Digit to
choose; chosen candidate is already scraped, so applying is instant.

## Reused / new pieces

- **Reuse:** `album_identity_overlap` (scorer), `recover-url`, `verify-spotify-url`,
  `_TriageSpotifySearchFirstUrl`, `_TriageApplyUrl` (apply + re-scrape), the scrape
  helpers (`WaitForScrapeReady` + `_SpotifyScrapeSpec*`).
- **New Python:** `identity-overlap --name <n> --albums "a|b|c"` → `{overlap, verdict,
  matched_plays, total_plays}` so AHK can score an arbitrary scraped album set; plus the
  album-count modifier inside `compute_confidence`.
- **New AHK:** `_TriageAutoResolve(id)` (candidate sweep + compare picker), the
  batch-end decision queue, and folding `/related` into `_TriageApplyUrl` (see gap
  below).

## Known gap to fix alongside
`_TriageApplyUrl` currently scrapes only `/discography/album`, **not `/related`**, so a
triage-resolved artist (e.g. the re-fixed Jack Johnson) contributes nothing to the
related/placeholder graph. The apply path should do the **dual scrape** the prefetch
pipeline does (`_PrefetchOneQueued`) and `save-placeholder-artists`, so resolved artists
re-enter the related graph.

## Cost note
Each candidate is a full Chrome+UIA discography scrape (~30s). 2–3 candidates ⇒ ~1–2 min
per artist. Fine on-demand; for the batch mode, only low/med-confidence artists trigger
it, and it runs unattended — Jamie only spends time on the end-of-batch picks.

## Phasing
- **P1:** album-count heuristic in `compute_confidence` (cheap, improves flagging now).
- **P2:** `identity-overlap` Python command + on-demand `_TriageAutoResolve` + compare
  picker.
- **P3:** fold `/related` into `_TriageApplyUrl`.
- **P4:** batch / during-scrape auto-resolve + end-of-batch decision queue.

See also: [[SPOTIFY_SCRAPING_INTERNALS]], [[MUSIC_CATALOG_BACKLOG]],
[[MILLER_TREE_ANALYZER]].
