---
tags: [programming, new-tab, music, lastfm, musicbrainz, media-catalog, anniversaries]
created: 2026-10-07
updated: 2026-10-08
related: ["[[NEW_TAB_PAGE]]", "[[MEDIA_SYSTEM]]", "[[SPOTIFY_SCRAPING_INTERNALS]]", "[[PEOPLE_SYSTEM]]"]
---

# Special Today

A new tab panel of things that make *today* unique: an album that came out N
years ago today, the day she first played something, an artist's birthday. Music
is the first source. The name was kept generic on purpose (Jamie, 2026-10-07):
the panel is expected to grow beyond music.

## What the data looked like (audit, 2026-10-07)

| Source | Size | Release dates? |
|---|---|---|
| Last.fm scrobbles (`E:/Media/Music/DataBackend/lastfm/scrobbles.db`) | 123k plays, 7.8k albums, 2.2k with 5+ plays | none, but **every play is timestamped** and ~half carry a MusicBrainz release id |
| `music.json` catalog | 6,833 albums | **year only** on all 6,754 that have one -- useless for "on this day" |
| Downloaded albums (`E:/Media/Music`) | ~15 artists | subset of the above |
| Spotify embed (`spotify_scraper.lookup`) | -- | empty for 4 of 5 albums tested, and gives the REISSUE date when present |

**Decision:** the scrobble history is the master list (her call) and MusicBrainz
release-group `first-release-date` is the date (the original, day-precise).

## The engine -- `Scripts/MediaCatalog/release_dates.py`

Store: `E:/Media/Music/DataBackend/release_dates/release_dates.db` (SQLite, WAL).
Tables `release_groups`, `albums` (scrobbled album -> release group, with how it
matched), `artists` (MusicBrainz life-span), `catalog_links` (release group ->
the catalog album's Spotify url).

**Why its own store and not fields in music.json:** two-thirds of the albums it
covers aren't catalog records (writing them would add ~7,000 stubs to the file the
Spotify pipeline and voice Choices treat as the library); it records misses,
retry counts and match provenance, which is what makes a 3-hour sync resumable;
and a background writer + the web server reading it at once wants SQLite. The
catalog still gets the dates via `backfill-catalog`.

Matching, in order:
1. The scrobble's release MBID -> `/release/<id>?inc=release-groups` (exact).
2. Release-group search on artist + title; accepts only an exact *core* title
   (edition noise stripped) and a matching artist credit; prefers Album > EP >
   Single, then the earliest date (a same-named single must not win).
3. Release search -> its group (catches deluxe/anniversary pressing titles:
   "OK Computer OKNOTOK 1997 2017" -> *OK Computer*, 1997-05-21).

Artists: their MBID, else an UNFIELDED search (name OR alias) -- `artist:"Kanye
West"` returns a tribute band and never Ye; Chopin is filed as "Fryderyk Chopin".

```
py release_dates.py sync [--min-plays N] [--artist-min-plays 5] [--budget-minutes M]
py release_dates.py status
py release_dates.py today [--date YYYY-MM-DD] [--json]
py release_dates.py lookup "<artist>" "<album>"
py release_dates.py backfill-catalog [--commit]
```

`sync` is most-played-first and resumable: ok/nomatch are final, errors retry up
to 3 times on later runs. Progress log: `sync.log` next to the db. 1.1s between
MusicBrainz calls (their hard limit is 1/sec).

## Scoring

> "If it's one of my favourite albums, the seven year anniversary is more
> important than the 25 year anniversary of an album I've listened to six times."

> "If it's a really important album to me or even just a really important
> artist... even if I have no listens on an album, if the album is in my library,
> or the artist has a lot of listens, I want those to show up." And: look a week
> ahead -- "if I know one of my favourite albums came out 4 days from now, I'll
> probably listen to it up until then." (2026-10-08)

`score = affinity x milestone x kind weight x proximity`

- **affinity** (albums) = the BEST of three signals, capped at 1:
  - her plays of the album: log(1+plays) / log(1+the 99th-percentile album);
  - her plays of the ARTIST, measured against the 99.5th-percentile artist and
    SQUARED, x 0.8 (`ARTIST_SHARE`): a favourite artist's unplayed album rates
    ~0.8, a middling one drops away (with the 99th percentile and no squaring, a
    269-play artist's unplayed live album outranked In Rainbows). Discounted for
    EPs (x0.85) and halved for live/compilation/remix/demo groups -- including
    untyped ones with "live" in the title, since MusicBrainz often leaves live
    records as plain Albums. Her OWN plays are never discounted;
  - +0.1 if the album is in her catalog library (`LIBRARY_BONUS`) -- never
    enough on its own to show an album by an artist she doesn't play.
  Artists' birthdays use the artist curve alone.
- **milestone** = 1.5 at 25/50/75/100 years, 1.35 at other decades, 1.25 at year
  one, 1.2 at other fives, else 1.0. A boost, never enough to carry an album she
  barely knows.
- **kind** = release 1.0, birthday/formed 0.9, first play 0.85, died 0.8.
- **proximity** = 1 - 0.06 x days ahead: today 1.0, tomorrow 0.94, a week 0.58.
  The window is today + 7 days (`LOOKAHEAD_DAYS`).
- Gates: affinity under 0.3 is never shown; a FUTURE date needs 0.6 -- only
  really important things are worth flagging ahead.
- Spelling variants that resolve to one release group pool their plays.
- First-play anniversaries in the first 60 days of the scrobble history are
  skipped -- that's the backlog of albums she already knew when she started
  scrobbling (2015), not discoveries.

### Where unplayed albums come from: discographies

`sync --kinds discog` browses each resolved artist with 10+ plays
(`DISCOG_MIN_PLAYS`): `/release-group?artist=<mbid>&type=album|ep`, 100 groups
per request, dates included -- ~900 artists in ~20 minutes, where searching the
6,800 catalog albums one by one would take hours. Stored in
`artist_release_groups`; a group a scrobble already resolved is never overwritten.
Artists with more than 250 groups (`DISCOG_MAX`) are skipped: that's a composer or
a reissue factory (Chopin is credited on thousands of other people's
recordings), and those dates mean nothing. "In library" = the release group
appears in `catalog_links`, which `backfill-catalog` fills by matching catalog
albums against both scrobbled albums and discographies.

Known soft spot: a classical album she plays a lot (Satie's *Gymnopédies*) is
dated by the recording she played, not the composition -- shown as-is.

Pinned by `Scripts/codebase_tools/tests/test_release_dates.py`.

**First-play anniversaries are HIDDEN (Jamie, 2026-10-07):** a first-scrobble
date is usually just when she started scrobbling, not when she found an album.
Still computed -- `DISPLAY_KINDS` in release_dates.py leaves `first_listen` out,
`today --all-kinds` brings it back.

## The panel

`newtab_server.py` source `special_today` -> `release_dates.py today --json`.
Rows are `external` plain links (Spotify album if the catalog has it, else a
Spotify search) opened in a new tab -- no AHK action. `external` is a separate flag
because Watching rows also carry an `href` (for Vimium) but must still run their
action on click. Stale-code trap applies: `newtab_server.py restart` after edits.

## People's birthdays (source 2)

`people_sync.py birthdays [--date D] [--json]` reads the person store's existing
`birthday` field (`1994-03-11`, or `--03-11` with no year). IFS parts are person
records and are skipped; an unparseable value ("October 7") is skipped, never
guessed. Feb 29 shows on Feb 28 in common years. In the panel a birthday scores
10 -- above any music row (max 1.5) -- so a person always heads the list. The
server merges sources in `fetch_special`; one failing source is logged and does
not blank the others. **As of 2026-10-07 no person has a birthday set**, so this
source is wired but silent until they are added (`people_sync.py set <id> birthday YYYY-MM-DD`).

## Open threads

- **Scrobbling stopped 2026-07-25** (newest play on JamieHooper). Diagnosed
  2026-10-07: she listens in the Spotify WEB player almost daily (context log
  since 2026-08-18; the desktop app is not installed), so the listening is
  happening and Last.fm is not receiving it -- the Spotify connection on Last.fm
  dropped. Fix: last.fm/settings/applications -> Spotify -> reconnect. Last.fm
  cannot backfill the gap; Spotify's "Extended streaming history" export (Privacy
  settings, up to 30 days) has every play and could be imported into
  scrobbles.db directly.
- Person birthdays look ahead the same 7 days (`people_sync.py birthdays --days 7`);
  a birthday always outranks music, nearer first.
- Nightly upkeep (Last.fm delta sync + `release_dates.py sync`) is not scheduled.
- Catalog albums she has never played are not resolved.
- Future sources for the panel: journal entries / quotes from N years ago today,
  book finish dates from the completion log.
