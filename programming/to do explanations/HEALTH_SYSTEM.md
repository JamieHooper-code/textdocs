---
tags: [programming, design-doc, health, sleep, journal, checkin, local-viewer, ahk, caster, people, weather]
created: 2026-10-13
updated: 2026-10-13
status: phases 1-2 built 2026-10-13 (sleep capture, store, Open Health page, check-in window, journal morning/night/health, boot + shutdown triggers); phase 3 (pain-after-activity + weather analysis) designed
related: ["[[JOURNAL_SYSTEM]]", "[[PEOPLE_SYSTEM]]", "[[REMINDERS_SYSTEM]]", "[[LOCAL_APP_HUB]]", "[[SETTINGS_SYSTEM]]", "[[COMPLETION_LOG]]"]
---

# Health system — sleep, daily check-ins, Open Health

Jamie wants to keep track of some of her health **without it feeling like a chore
or a judgement**: *"helpful, gentle, and informative without being judgmental"*,
and *"if this ends up being too much of a chore to fill out, I'm never going to do
it."* Those two sentences decide most of the design.

## Pieces

| Piece | Where | Status |
|---|---|---|
| Engine (every rule + store) | `AutoHotkey/Scripts/health/health.py` | built |
| Power history (append-only) | `E:/Media/catalog/health/power_events.jsonl` | built |
| Answers + confirmed sleep times | `E:/Media/catalog/health/health.json` | built |
| Answer inbox (window appends, engine folds) | `E:/Media/catalog/health/inbox.jsonl` | built |
| Weather | `E:/Media/catalog/health/weather.json` (Open-Meteo, Gainesville) | built |
| Questions (data, not code) | `AutoHotkey/INIDATA/health_questions.json` | draft v1 |
| Voice (bare numbers in the window) | `caster/rules/health_checkin_commands.py` (`health checkin rules`) | built |
| Weekly + seasonal templates | `INIDATA/CheckinTemplates.json` types `weekly`, `season` | draft |
| Page server | `Scripts/health/health_server.py` :8298 + `health_page.html` | built |
| AHK | `Helpers/HealthFunctions.ahk` (`OpenHealth`, `HealthCheckin*`) | built |
| Boot | `Startup/Startup.ahk` runs `MAINFUN HealthCheckinMorning auto` (captures first) + `weather` | built |
| Shutdown | `ShutDownComputer` calls `HealthCheckinNight shutdown` first (by name) | built |
| Tests | `Scripts/codebase_tools/tests/test_health_nights.py` | built |
| Settings | `health.weather_lat`, `health.weather_lon` | declared |

## Sleep — from the computer's own history

**Source: the Windows System event log**, not a log of ours. Kernel-General 12 =
boot, 13 = shutdown, User32 1074 names *who asked*: `shutdown.exe` is her
"shut down computer" command (`ShutDownComputer` → `shutdown /s /t 0`),
`winlogon.exe` the Start-menu power button, `TrustedInstaller.exe` an update.
On the build day this reached back to 26 July 2026 — 72 nights of history with no
waiting. **The System log is size-capped and rolls**, so `capture` copies it into
`power_events.jsonl` (deduped on time+kind) at every boot and whenever the page
loads; from then on the Windows log only matters for the recent tail.

**A night is a chain of off-periods** (`assemble_nights`):
- off < 20 min = a restart; invisible, part of the surrounding session
- an off-period may start a night from 18:00 to 08:00
- night off-periods joined by on-sessions < 4.5 h are ONE night; the sessions are
  `wakings` and are subtracted from `asleep_min`. 4.5 not 3: on 4 Oct 2026 she was
  up 02:44–05:56 (3h12m) and went back to bed.
- a chain needs ≥ 3 h off and must span ≤ 16 h (longer = away/ill → no data)
- `trusted` = began with `shutdown.exe`. Untrusted nights are drawn faintly and
  kept out of averages (Jamie: only count nights her command ended).
- `waking_now()` — a boot before 05:00 within 2 h of a night shutdown is a
  middle-of-the-night one; the morning check-in stays quiet and the night
  check-in skips its questions on the way back to bed.

**Computer-off is a guess, and is stored as one.** The morning check-in shows
the guess and asks; confirmed times go in `health.json` → `nights[wake_date]
.confirmed` and win wherever they exist, never overwriting the guess.

## The page (Open Health)

**A gentle check-in, not a dashboard.** The first version (stat tiles, a
clock-axis timeline, an hours chart with a usual-range band, a weather chart)
was overwhelming — Jamie: *"light and sparse ... often with words, rather than
being tons of intense data. I want intense data on the backend, but I don't
necessarily want to see it."* So the page is now:

- a greeting by time of day
- one sentence about last night, rounded the way a friend would say it
  ("about 8 and a half hours, from around 11 until around 8"); a line if she was
  up in the night, without saying how long
- "your evenings have had a steady rhythm this week" — ONLY when it is true
  (bedtimes within 90 minutes); otherwise nothing is said
- today's Gainesville high/low in words
- one quiet picture: the last two weeks as a row of moons (fuller = longer rest),
  no axis, no numbers; hover gives "Tuesday: about 9 hours"

- **check-in words (an experiment Jamie okayed, "not sure how it'll go"):** the
  latest check-in said back in plain words -- elbows as settled / a little sore /
  sore / really sore / very painful, "you said you were feeling okay", and
  "Yesterday had skating and time with Cedar in it". Never the numbers, never
  advice; each line only when there's an answer behind it (`/api/data`
  `checkins`, phrasing in `checkinWords` in the page)

It never grades: no target, no red, no "worst", no minutes or decimals, nothing
worded as a shortfall. All the detail (wakings, trust, weather, averages) stays
in `/api/data` and the store for the questions she'll ask later. Copy follows
the `gentle-lines` skill.

## Pain after activity (future -- ASK JAMIE BEFORE BUILDING)

The questions she wants answered one day: *how long and how bad are my flare-ups
after skating / swimming? how many points raised, for how many days?* Pain lasts
several days, so skating on Wednesday and pain on Friday is data. **Not designed
yet on purpose** (Jamie 2026-10-13: "I want to build that in the future and ask
you questions before we build that with the data"). What is already in place so
the data will be there:

- **pain is asked first in every check-in** (priority 100 -- "in case I don't
  answer anything else") and **daily**, morning AND night
- activities are stored per day. **The activity list moved on 2026-10-16** to the
  one activity store, `INIDATA/activities.json` (shared with "log skating" and the
  completion log; see [[COMPLETION_LOG]] § "Activities with questions"). Each
  activity carries its own questions (skating: intensity, managing body, elbow
  cost), asked in the check-in when it is ticked and stored as
  `act.<id>.<key>` answers. That is the per-session data the flare-up question
  will need. (The old `watch` field was never read and did not move.)
- weather (high/low/rain/humidity/pressure) is captured daily alongside

## The check-in window (built)

**Answers go through an inbox.** The window never shells Python per keypress:
each answer is one JSON line FileAppended to `inbox.jsonl` (instant, and quick
answers like 7-5-3-9 cannot race a load-modify-save), and `health.py` folds the
inbox into `health.json` whenever anything reads the store (rename-first, so a
line appended mid-fold lands in a fresh inbox). `HEALTH_STORE_DIR` redirects
the whole store; `HEALTH_TEST_KEYS` replays a key sequence into the window's own
handlers — together they test the window without touching real data or focus.

Three sets from one registry: **morning** (on boot, with AHK — no extra delay,
Windows already staggers startup), **night** (inside `ShutDownComputer`, before
the Dragon save; Esc never blocks the shutdown; with a delay, asked NOW), and
**journal** (any time). Voice: `journal morning` / `journal night` /
`journal health`; saying it again **resumes** — answers are written one at a
time, so quitting halfway loses nothing.

**Never a chore:**
- `priority` per question; most important first, so half a check-in is the good half
- `group` keeps pairs together (the elbows)
- `every` (daily / often / sometimes), picked by date, so it isn't the same every day
- `0` skips a question, `00` ends the check-in, Esc closes, nothing is required

**Keys:** 2–9 answer instantly and advance. `1` waits a beat: `1 1` = **10**
(same key twice, her choice over `1 0`), `1` alone = 1. `0` waits a beat: `0 0`
= done for today, `0` alone = skip. Voice: bare numbers while the window is up
(function_context rule, like reading-room bare numbers); "ten" is unambiguous.
Tab / "note" opens an optional one-line note on any question.

**Direction:** default is 10 = good ("how calm" over "stress level" — more
pleasant to answer); pain keeps 10 = terrible (`good: low`), which is how she
already thinks about it. Charts read the per-question direction.

**Morning sleep confirm:** "in bed ~11:05pm → up ~7:55am (woke 3:10–4:05)" —
`1` right, `2` fix bedtime, `3` fix wake; fixing is a numbered list in 15-minute
steps around the guess, plus typing `1130` for anything further off. Skipped
mornings ask only about the latest night. Then "how did last night's wind-down
ritual go?" — stored on that night.

**Activities:** a toggle pick list + a free-text box for one-offs. **Hangout**
opens a people picker (the person store, most-frequent first) and stores
`person_id`s — "Cody hangout", "Charlie hangout".

**At the end** (morning and night, INCLUDING at shutdown; the journal set ends in
its long-form box instead): offer the bigger reflective check-in when one is due — the
weekly one from Monday, or the moon / seasonal one (new, first quarter, full,
last quarter + solstices and equinoxes) via `JournalCheckin.ahk`. One key to
start, one to skip; never offered twice in a day. At shutdown the writing box it
opens is waited out (`_HcWaitForWritingBox`: up to 30s to appear, then as long as
she writes) so it is finished BEFORE the shutdown confirm appears. The `weekly`
and `season` question sets are Claude's drafts -- Jamie hasn't reviewed them yet
and will later.

**Long-form** (journal set only): after the numbers, the existing check-in
writing box with the long questions as labelled blanks — same machinery as
`check <type>`, `GuiGoToNextBlank` to move between them; saved as a journal
entry linked to that day's health record. A generic "jump N" to a numbered blank
would serve the ChatGPT question rounds too.
