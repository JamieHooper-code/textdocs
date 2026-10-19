---
tags: [programming, design-doc, events, gainesville, scraping, calendar, local-viewer, chrome-extension, local-llm, music]
created: 2026-10-18
updated: 2026-10-18
status: phase 1 built 2026-10-18 (31 sources, scoring, page on :8300, calendar feed, nightly task, new tab rows); springs + stargazing generated readers added same day; Google Calendar publishing, band-first lookups next
related: ["[[LOCAL_APP_HUB]]", "[[NEW_TAB_PAGE]]", "[[GLOBAL_CLICK_HINTS]]", "[[SPECIAL_TODAY]]", "[[MUSIC_GRAPH]]", "[[HEALTH_SYSTEM]]"]
---

# Gainesville events — scrape the town's calendars, keep only what she'd go to

Jamie misses local events: queer nights, leftist organizing, DIY/folk-punk shows,
readings, nature things. Most of them get announced on Instagram, which she avoids.
This system reads venue and community calendars directly, merges them, and keeps
only what matches her interests, with **why** next to every pick so a wrong one
points straight at the keyword to change.

> "build a system where it's very easy to add different categories and keywords and
> also different event sources, and then not have to look through like a million
> events that I don't care about."

## The shape

```
INIDATA/events/sources.json ──► events_readers.py (one reader per KIND of site)
                                     │  plain HTTP, or her real Chrome (Macro Bridge ext.fetchpage)
                                     ▼
                     ~/MacroData/events/events.db  (raw events, her marks, run log)
                                     │
INIDATA/events/categories.json ─► events_score.py  (scored at READ time, never stored)
INIDATA/events/bands.json ──────►        │
                                         ├─► events_page.html on :8300 (Picks · Calendar · Everything · Settings)
                                         ├─► picks.ics / http://127.0.0.1:8300/feed.ics
                                         └─► new tab Special Today, "events" section (≤4 rows)
```

| Piece | File |
|---|---|
| CLI: sync / test / picks / status / feed / today | `Scripts/events/events.py` |
| Readers | `Scripts/events/events_readers.py` |
| Scoring, duplicates, repeats | `Scripts/events/events_score.py` |
| Store + config IO | `Scripts/events/events_store.py` |
| Page server (:8300) + page | `Scripts/events/events_server.py`, `events_page.html` |
| Check a new site's platform | `Scripts/events/probe_source.py` (`py -I probe_source.py candidates.json`) |
| Nightly task | `GainesvilleEventsSync` (5:15am + 5:15pm, pythonw), definition `Scripts/events/GainesvilleEventsSync.xml` |
| AHK | `Helpers/EventsFunctions.ahk`: `OpenGainesvilleEvents`, `SyncGainesvilleEvents` |
| Chrome reads | Macro Bridge op `ext.fetchpage` in `ChromeExtensions/macro-bridge/background.js` |
| Local model | task `event_extract` in `Scripts/local_llm/tasks.json` (qwen2.5:14b) |
| Recon (every candidate site + how hard to read) | § "Source recon" below |

## Readers: build by PLATFORM, not by site

Most places sit on one of a few platforms, so one reader covers many. A new site
is one row in `sources.json`; only a new platform needs code.

| kind | Covers | How |
|---|---|---|
| `ics` | Curia complex, Civic Media Center (any public Google Calendar) | iCal file; RRULE expanded with dateutil, EXDATE + RECURRENCE-ID honoured |
| `tribe` | Visit Gainesville (the firehose, ~850 events), Florida Museum | WordPress "The Events Calendar" REST API, paged |
| `squarespace` | First Magnitude, Pride Community Center, SAW, Working Food, ACT | `?format=json` on the events page, `upcoming` list |
| `jsonld` | Eventbrite search pages | schema.org Event objects in the page |
| `mobilize` | Mobilize near 32601 (Labor Coalition, campaigns) | public API |
| `glorydays` | High Dive, The Wooly, Signal | one HTML list; filtered to `city: Gainesville` |
| `heartwood` | Heartwood Soundstage | Webflow list, **no year on the cards**: the list is in date order, so the year ticks over the first time the month goes backwards |
| `gainesvilleshows` | gainesvilleshows.com | Next.js page data (`self.__next_f.push` chunks), objects with `isUpcoming` |
| `ox` | The Ox (theoxfl.com) | the page carries a hand-kept `SHOWS=[...]` array (JSON with trailing commas) |
| `wildapricot` | Alachua Astronomy Club | events list: `eventDetailsLink`, then `When <d Mon yyyy> <time>` |
| `springs` | Springs days (Gilchrist Blue, Rum Island) | **generated**, not scraped: Open-Meteo daily forecast; see § Springs + stargazing |
| `stargazing` | New moon weekends, clear dark nights | **generated**: moon phase computed locally + Open-Meteo night cloud cover |
| `page` | everything else | fetch (plain, or `"via": "chrome"`), then JSON-LD, then any embedded Google Calendar id, then the local model reads the text (cached by content hash for a week) |

**Found while building, worth knowing:**
- The Civic Media Center site sits behind a Cloudflare check, but its page links a
  Google Calendar. Read once through Chrome, the calendar id became a plain `ics`
  source that needs no Chrome at all. **Always look for an embedded calendar first.**
- Depot Village's calendar is a Prekindle widget; the widget page itself
  (`prekindle.com/organizer-grid-widget-main/id/<org>/`) is plain HTML with JSON-LD.
  The library's is an `attend.aclib.us/widget?...` iframe, also plain.
- `theatlanticgainesville.com` is a fake SEO site, not The Atlantic.
- Alachua Audubon's robot check does not clear in a background tab. Off, with a note.
- Main Street Daily News' events widget doesn't draw in a background tab. Off for now.

## Reading through her Chrome (`"via": "chrome"`)

`ext.fetchpage {url, settleMs, timeoutMs}` (background.js) opens the URL in a
**background tab** (never focused), waits for `complete`, waits `settleMs` for
scripts, keeps waiting while the title says "Just a moment" / "Bot verification",
returns `{html, text, title}` and closes the tab. Python calls it through the bridge
host: `POST http://127.0.0.1:8299/cmd` with `Origin: http://127.0.0.1:8299`
(`chrome_fetch` in events_readers.py). Caps: the bridge's 20s per call, and html is
cut at 800k chars because a native-messaging reply over 1 MB kills the port.

A sync with Chrome closed (bridge not `connected`) skips those sources ("waiting for
Chrome") instead of failing them, so they stay due and the next sync with Chrome
open catches up. It never solves a CAPTCHA, only waits for automatic checks.

## Scoring: what counts as a pick

Computed **at read time** from `categories.json` + `bands.json`, never stored, so
a keyword edit on the page re-sorts the page and the feed on the next load.

- Six categories (queer, left/organizing, folk/acoustic, DIY/punk, community/art,
  nature/sky), each a dict of `word: weight`. A title hit counts 1.5×. A trailing
  `*` is a prefix (`organiz*`). Phrases are runs of words.
- **Matching is on WORDS, not regexes:** the text is split once and each keyword is
  a dictionary lookup. Regex-per-keyword took 1.3s for ~1200 events. One giant
  alternation took **13s** (lookbehinds at every position). Word lookup takes 0.3s.
- `avoid` words subtract from every category (kids, football, webinar, committee
  meeting, open hours…). `ignore_phrases` are removed first ("Curia on the Drag"
  must not count as drag).
- Source `boost` and venue boosts help, but **together they stop one point short of
  the line**. Before that cap, every committee meeting and "OPEN HOURS" at the CMC
  was a pick (138 of 323). Now an event always needs at least one real word.
- **Picked** = any category ≥ `threshold` (3), or a band from bands.json plays, or she
  pressed Keep / Going. Hide beats everything.
- **Duplicates** (venue + aggregator listing the same thing): same day, and one
  normalised title contains the other or their first 6 words match. The most
  specific source wins; the rest show as "via X + Y".
- **Repeats** (weekly open mic, Food Not Bombs, the Hipp's film run) fold into one
  card on their next date with "Also Oct 25, Nov 1…". Keyed on the WHOLE title +
  venue. A 4-word prefix folded every "First Magnitude Brewing Company – …" event
  into one series.

Tuning on the first full read (1,200 events): 323 picks before the boost cap and
the word fixes → **85–97 distinct picks over four months**. `py events.py picks`
and the Everything view are where to look when tuning.

## Learning from marks (2026-10-18)

Her Going / Keep / Hide marks TRAIN the ranking: `Scripts/events/events_learn.py`,
recomputed from the marks table on every scoring pass (milliseconds), so it learns
the moment she clicks. No "update Claude" step.

- Labels: Going +1, Keep +0.7, Hide -1. Features of each marked event: its venue,
  its title words and word pairs ("book club", "run club"), the keywords that fired.
- Affinity per feature = sum(labels) / (count + 1.5): one mark nudges, many marks move it.
- Bonus = venue × 1.5 + title words × 0.7 + keywords × 1.5, each kind capped, total ±2.5.
  **Keywords learn DOWNWARD only:** her marks are mostly positive, so letting them
  learn upward double-counted every common word ("all ages" rose on every show she
  kept; picks jumped from 97 to 142). Venues and title phrases carry the positive side.
- `rank` = keyword score + learned bonus (+4 for a band she likes). A pick = keyword
  score + learned bonus reaching the threshold. The card says why: "you liked: the
  lynx, book club" / "you hid: rally".
- First training run (36 marks, 2026-10-18): 115 picks, 25 added by learning (The Lynx,
  CMC, Heartwood, The Bull listening rooms). Pushed down: run clubs, pickleball,
  school theatre, Gator football.

**A 0-5 rating was considered and parked.** The three buttons already map onto
it (Hide = 0, Keep ≈ 4, Going = 5), and with this little data the gain is
direction, not grade. More Hides on wrong picks teach the most.

**Weekly things are marked as a SERIES.** A mark on anything with more than one
date goes to `series:<hash of title+venue>` (`series_mark_id`) and covers every date;
the occurrence's own mark is cleared. A series counts once in training. A
**regular** = a series marked Going with 3+ dates spanning 2+ weeks (the Monday
Grove Street market, Poetry Jam, Books 2 Prisoners). A two-night Fest show is a plan,
not a regular. Every date of a regular is still in the calendar feed (she goes).

## The page (:8300)

Picks (by day, with why + Going / Keep / Hide), Calendar (a rolling six weeks from
this week, arrows move four), Everything (non-picks dimmed, with search, to catch
what the words missed), Settings (threshold, each category's words as removable
chips + add with a weight, avoid words, bands, sources with last run / count /
error + Read now + on/off, the feed URL). All controls are real buttons with
`data-jump`. Scoring is cached in the server on the mtimes of the db + config files.

**For you** (the default view, 2026-10-18) is for "I'm bored today" and for
planning the week or month: chips Today / This week / Two weeks / This month pick
the best 8 / 12 / 15 / 20 by `rank`, shown in date order. Going events always make
the cut (they're plans); hidden ones never do; one date per repeating thing; events
that started over two hours ago today drop out. Below a heavy rule come **More
picks** (every other pick in the range) and then **Everything else** (the rest,
dimmed, always drawn: a button at the bottom of a long scroll is buried), each in date order. Jamie: "95% of
the time I'll be done by the time I get past the first day", but all of it is
reachable.

**Regulars** sit on one line at the top, each with two switches stored in
`INIDATA/events/regulars.json` (keyed by series id): **Show** (list it as a card like
anything else; default off) and **Calendar** (every date in the feed; default on).
`POST /api/regular {series_id, title, show?, calendar?}`.

**Jump numbers are drawn by the page** (`data-jump-pos="inline"` on every control +
a `[data-jump-n]::before` pill): small and grey on buttons, accent-coloured before
each event title. Jamie asked for them to look nicer than kit's floating badges,
which sat over titles.

The page triggers a sync when the last one is >12h old. Stale-code trap: the server
holds `events*.py` in memory, so `py Scripts/events/events_server.py restart`
after editing them. The page itself and the JSON config are re-read on every request.

## Not built yet

- ~~Google Calendar subscription~~ **Built 2026-10-18 (her OK):** Google only
  subscribes to a public URL, so `_publish()` pushes picks.ics to a SECRET GitHub
  gist (account JamieHooper-code) after every sync and every mark. Config
  `INIDATA/events/publish.json`; the subscribed address is the raw link with no
  revision hash, so it always serves the latest. The page's upload is fire-and-forget
  (a click must not wait on GitHub); the scheduled sync waits for it. Google
  re-fetches on its own schedule (~8-24h), so a mark shows there up to a day later.
  **The feed carries ONLY events marked Going** (Jamie, 2026-10-18: the calendar is
  her plans; browsing picks is the page's job). A subscribed calendar is its own
  read-only layer in Google, never the primary calendar; writing Going events INTO
  the primary calendar needs the Calendar API (open question for her).
- **Band-first lookups** (Bandsintown per artist for touring bands she likes). The
  bands list is empty so far; local bands she adds already match by name.
- **Instagram** (deliberately parked by Jamie, 2026-10-18).
- Gainesville Iguana (calendar inside a PDF newsletter; doctext could read it),
  Alligator roundups, Main Street, Audubon.
- **Springs + stargazing extras**: water level / flow (USGS gauge on the Santa Fe at
  Fort White, for when the river is high and tannic), park capacity closures, and a
  dark-sky spot closer than Chiefland. Built so far: § Springs + stargazing below.

## Springs + stargazing (generated events, 2026-10-18)

Her back-burner idea: propose a springs day when the weekend weather is good, and
flag dark nights for stargazing. Neither is scraped; two reader kinds MAKE events
from conditions, so they ride the same page, feed and new tab rows, with Going /
Keep / Hide like anything else. Both read Open-Meteo (free, no key, one call each).
Thresholds sit on their rows in `sources.json`.

| Row | Emits | Settings on the row |
|---|---|---|
| `springs` | A Sat/Sun in the next `days` (7) whose high ≥ `min_high_f` (80) and rain chance ≤ `max_rain_pct` (30) → one all-day card *"Springs day? Gilchrist Blue / Rum Island — 84°, 10% rain"*, each spot's high / rain / sky / UV / wind in the text | `spots` (name, lat, lon, url), `days_of_week` |
| `stargazing` | Fri/Sat nights within `new_moon_days` (3) of a new moon → *"New moon weekend (Nov 9)"*, 9pm–2am. Inside the 16-day forecast: cloud ≤ `max_cloud_pct` (25) and moon ≤ `max_moon_lit` (0.3) → *"Clear dark night — good for stargazing"*; a cloudy new-moon night says *"— cloudy forecast"* | `spot` (Chiefland Astronomy Village), `nights`, `night_start`, `night_hours` |

- **One springs card, not two.** Gilchrist Blue and Rum Island are half a mile apart
  on the Santa Fe, so their forecasts are the same grid cell. One card names both.
- **The new-moon date is in the title** on purpose: the page folds same-title repeats
  into one card ("Also …"), which hid every later month under November. With the
  date, each month is its own card and that weekend's Fri + Sat fold together.
- **Same uid per date** (`springs:<day>`, `sky:<day>`): a forecast change updates the
  card in place, an upgrade to "Clear dark night" keeps her Keep/Going mark, and a day
  that stops qualifying is flagged gone on the next sync like a cancelled show.
- **Moon phase** is the mean-synodic-month formula from a known new moon (2000-01-06
  18:14 UTC), good to about half a day, which is plenty for picking weekends. No
  skyfield ephemeris download. Checked: Nov 9, Dec 9, Jan 7, Feb 6 all land right.
- **Scoring**: "new moon", "stargaz*" were already nature words; added
  `springs day` and `clear dark night` (5 each). Each row also boosts nature by 2,
  but the boost cap means the title word is what makes them picks.
- On build day the springs reader returned nothing (highs of 70–76° in late October),
  which is correct; lower `min_high_f` if she'd go in cooler weather.

## Source recon (2026-10-18)

Graded by how hard each was to read. A = feed/API/JSON, B = steady HTML,
C = model reads it, D = Instagram only / blocks reads.

| Source | Grade | Status |
|---|---|---|
| Visit Gainesville | A | on (tribe) |
| gainesvilleshows.com | B | on |
| Glory Days (High Dive, Wooly, Signal) | B | on |
| Eventbrite | A- | on (jsonld, 5 pages) |
| Mobilize | A | on |
| Bandsintown / Songkick | C | not built (band-first later) |
| Gainesville Iguana | C | not built (PDF) |
| Alligator roundups | C | not built |
| Main Street Daily News | D | off (widget won't draw in a background tab) |
| City of Gainesville | D → Chrome | on |
| Heartwood Soundstage | B | on |
| The Ox | A (found theoxfl.com) | on |
| The Atlantic | D | off (fake site; Instagram only) |
| Loosey's | B → model | on |
| Depot Village | A (Prekindle widget) | on |
| The Bull, Lightnin' Salvage, South Main, Satchel's | D | off |
| The Fest | B | off (carried by Heartwood + Glory Days) |
| Hippodrome | Chrome | on |
| UF Performing Arts | model | on |
| Curia complex | A (Google Calendar) | on |
| Civic Media Center | A (Google Calendar, found via Chrome) | on |
| The Lynx | Chrome + model | on |
| First Magnitude | A | on |
| Pride Community Center | A (empty calendar) | on |
| Sequential Artists Workshop | A (empty now) | on |
| Working Food | A | on |
| Repurpose Project | model (nothing listed) | on, weekly |
| Labor Coalition | — | off (news posts; events come via Mobilize) |
| DSA / PSL / SJP / JVP | D | off (Instagram only) |
| Alachua County Library | widget, model | on |
| Matheson Museum | Chrome + model | on |
| Alachua Astronomy Club | B | on |
| Alachua Conservation Trust | A | on |
| Alachua Audubon | D | off (robot check) |
| Florida Museum | A | on |
| Springs days (Open-Meteo) | A (generated) | on |
| Stargazing nights (Open-Meteo + moon) | A (generated) | on |
