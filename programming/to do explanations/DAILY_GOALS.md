---
tags: [programming, design-doc, new-tab, checkin, health, completion-log, goals]
created: 2026-11-13
updated: 2026-11-13
status: built 2026-11-13 (systems item t-0bd04f76, New tab page) -- one goal a day, general goals built but off
related: ["[[NEW_TAB_PAGE]]", "[[CHECKIN_SYSTEM]]", "[[HEALTH_SYSTEM]]", "[[COMPLETION_LOG]]"]
---

# Daily goals

Jamie, 2026-06-15: *"Get daily goals added and make a little user interface I
can easily modify… like meditating twice today was a big goal. I can have like
general daily goals and specific ones."*

Jamie, 2026-11-13: *"I am able to state a daily goal when I do my morning
check-in, and so we'll need to beef up that user interface as well so that I
can pick from a list or input something manually. Everything that I input
manually will be saved so that we can theoretically add it to the list later."*

Jamie, 2026-11-13, after the design: *"At this moment, I just want the single
thing that I pick per day so that I don't get overwhelmed and can slowly
integrate this into my life. I don't want these things to start feeling like
chores. I want them to feel organic to how I actually feel today, but I do
want all of that shit built."* And: *"I want all of this data to be logged,
but I don't want to see it."*

## As built (2026-11-13)

| Piece | Where | State |
|---|---|---|
| Store + rules | `Scripts/goals/goals.py` (`ctl goals`), `INIDATA/goals.json`, `E:/Media/catalog/goals/` (days/, typed.json, events.jsonl) | built, 17 tests (`test_goals.py`) |
| Morning question | `INIDATA/health_questions.json` `goal_today` (kind `goals`, every day) -> `GoalQuestion` in `Helpers/GoalsFunctions.ahk` | **one pick or type one** (`goals.morning_max` = 1) |
| Pick-or-type control | QuestionWindow `choice` + `other` (new) / `multi` when max > 1 | built |
| "How many times?" | spliced after a pick whose list goal normally counts > 1 | built |
| New tab panel | "Today's Goals", very top left (`panels.js` `place: "top-left"`), `view: "goals"` in app.js, source `goals` in newtab_server.py | built: tick / take back / not today / + goal (chips + type) / Edit (list + typed) |
| General goals | `general` per goal + `goals.general_on` | **built, OFF**: setting false, nothing marked general; the "every day" toggle only shows in Edit once the setting is on |
| Evening line | `goals.py evening-line`, shown on the night check-in's first question | **built, OFF** (`goals.evening_line` false) |
| Voice | "add goal", "add goal <words>", "open goals" (generic store) | live |
| Old answers | the 6 old `hope_today` answers imported into the typed pool | done |

Nothing shows her any history, count or streak. Everything is recorded (the
day files, the typed pool, events.jsonl, the health answer `goal_today`, the
completion log for ticks) for later, not displayed.

Settings (Goals system in the settings page): `goals.morning_max` (1),
`goals.general_on` (false), `goals.evening_line` (false).

The library was seeded from the completion-log templates (Meditate,
Stretches, Reading...), each counted by its own log entries. "Worked on
Atlantic Logistics" / "Went outside" were renamed "Atlantic Logistics" /
"Go outside" as goals. Dailies were NOT made general goals (that was the
design's decision 3; with general goals off it would have meant nothing).

## What already existed (built on it, not duplicated)

| Piece | Where | What it gives goals |
|---|---|---|
| Morning goal question | `INIDATA/health_questions.json` key `hope_today`, kind `text`, `every: sometimes`, priority 30 | The slot in the morning check-in. Today it is a free-text line asked about twice a week. |
| Pick-from-list + type-your-own | Question window kind `multi` with `other: true` (`Helpers/Gui/QuestionWindow.ahk`) | The exact control the morning question needs. Activities already use it. |
| Follow-up questions after a pick | `_HcSpliceAfterActivities` in `Helpers/HealthFunctions.ahk` | The pattern for "how many times?" after she picks a counted goal. |
| Dailies | completion-log templates with `daily: true` (`clog.py daily-status / set-dailies`, voice "edit dailies", "log daily", the Dailies X/N meter) | Her **general** daily goals already, by another name. |
| Automatic counting | the completion log: the meditation page logs EVERY session as template `meditate`; `log skating` etc. log activities; `_matches()` matches a template OR a tag | "Meditate twice" can fill itself in: 1 of 2 after the first session, done after the second. |
| New tab live panels | `Scripts/newtab/newtab_server.py` `SOURCES`, `web/panels.js`, views `checklist` / `recurring` / `pool` / `links` | The panel to show and tick today's goals. |

## The model

**Goal library** (`INIDATA/goals.json`, one writer: `Scripts/goals/goals.py`,
reached through `ctl goals ...`). Each goal:

- `id`, `label` ("Meditate", "Call Mom", "Go outside")
- `target`: how many times counts as done (default 1; "meditate twice" = 2)
- `counts_from` (optional): what fills it in automatically. Either a
  completion-log template (`meditate`, `reading`), an activity (`walk`,
  `skating`), or a tag (`exercise` = any exercise). Picked from a list, never
  typed.
- `general: true` = asked for every day (a **general** goal). Otherwise it is
  a **specific** goal: only on days she picks it.
- `last_picked`, `times_picked`: for ordering the morning list.

**Dailies become general goals.** Rather than a second list, every template
already marked daily shows up as a general goal with `counts_from` = itself.
"Edit dailies" and the goal editor change the same flag. (Decision 3 below.)

**Typed pool** (same file, `typed` section). Every goal she types instead of
picks is saved: the text, every date she used it, and whether it was done.
It is never shown as a choice until she promotes it. The editor shows the pool
newest first with a use count, and promotes one with a pick. A line typed 3
or more times gets a quiet "add to your list?" row in the editor (decision 5).

**Today** (`INIDATA/goals/days/YYYY-MM-DD.json`, written by goals.py). Today's
goals with each one's target, progress and where it came from (morning,
new tab, voice). The day boundary is the health system's (`health.py plan`'s
`date`), so a 1am session still counts toward "today".

Progress = manual ticks + matching completion-log records for today. Ticking a
goal that has `counts_from` logs that template (it lands in Done Lately the
normal way) instead of keeping a separate tick, so the log stays the one truth.
A goal with no `counts_from` keeps its ticks in the day file and logs one
"Goal: <label>" line to the completion log when it is reached.

## Morning check-in

`hope_today` becomes a `goals` question:

- kind `multi`, with `other: true` for typing. **Asked every morning**, not
  "sometimes" (decision 2). Label stays close to her own words: *"Anything
  you're hoping to do today?"*
- The list: general goals first, already ticked (untick = not today), then
  specific goals by how recently she picked them. Numbered like every other
  pick, so the numpad works.
- "Something else" takes one or several, split on commas and "and" (the same
  split the people question already uses). Each becomes a goal for today and
  a typed-pool entry.
- For each picked goal with a `target` other than 1, a spliced follow-up:
  *"How many times for Meditate today?"* as a choice of 1 / 2 / 3 / 4, the
  library default highlighted, so Enter keeps it.
- Old `hope_today` answers are imported into the typed pool once, so months of
  stated hopes are there to promote.

The window code changes are small: a `goals` branch in `_HcToWindow` (build
the options) and `_HcAnswer` (save + splice the count questions), modelled on
the activities branch.

## New tab panel: Today's Goals

A new live panel near the top, `view: "goals"`:

- One row per goal: the label, then progress as dots (●●○ for 2 of 3) or a
  plain tick for target 1. Automatic ones fill in as she does them; the page
  re-reads when it comes back into view, like every other live panel.
- Clicking a row (or saying its jump number) adds one. A reached goal goes
  quiet, dimmed and kept in place, so nothing shifts under her.
- An add row at the bottom: pick from the library (numbered list), or type.
  Typed ones go to the typed pool exactly like the morning ones.
- Edit (the panel's Edit button, as on Check-In): the library. Rename, set
  the target with a stepper, choose `counts_from` from a list, toggle general,
  remove; the typed pool below it with "add to list".
- Gentle by design (see the gentle-lines skill): no red, no "missed", no
  streak counts. An unreached goal at the end of the day just isn't ticked.

## Evening

Optional, decision 6: the night check-in shows one line before its
questions, *"Today: Meditate 2 of 2 · Call Mom ✓ · Stretches"*. Nothing to
answer, just seen. Any unreached specific goal can be offered for tomorrow
morning's list (it is near the top anyway because it was picked recently).

## Voice (small, after the rest)

- "goal <words>": add a goal for today (typed-pool rules apply).
- "goals": open the new tab and scroll to the panel.

## Files

| File | Change |
|---|---|
| `Scripts/goals/goals.py` (new) | library, typed pool, today, progress from the completion log; CLI `list / today / add / tick / set / promote / import-hopes`; `--json` everywhere |
| `INIDATA/goals.json`, `INIDATA/goals/days/` (new) | the store |
| `Scripts/codebase_tools/ctl_areas.json` | `goals` area |
| `INIDATA/health_questions.json` | `hope_today` → `goals` kind, every day |
| `Scripts/health/health.py` | `plan` hands over the goal options; the answer goes to goals.py |
| `Helpers/HealthFunctions.ahk` | `goals` branch in `_HcToWindow` / `_HcAnswer`, count follow-ups |
| `Scripts/newtab/newtab_server.py` | `goals` source + write ops (tick, add, set, promote) |
| `Scripts/newtab/web/app.js`, `panels.js`, `index.html` | `goals` view + its editor |
| `Scripts/completion_log/clog.py` | dailies read through as general goals (or move into goals.json, decision 3) |
| caster rule | "goal <words>", "goals" |

## Build order

1. goals.py + store + tests (nothing visible yet).
2. Morning check-in question (the part she uses first thing).
3. New tab panel, then its editor.
4. Evening line, voice, the typed-pool nudge.

## Decisions (2026-11-13)

1. Build order, morning first: yes (all of it built in one go).
2. Ask the goals question every morning: **yes**.
3. Dailies are general goals: **changed** -- general goals are built but
   switched off and empty; one goal she picks a day for now.
4. Counted goals ask "how many times?": yes.
5. Typed 3+ times -> quiet "often" mark beside it in Edit: yes.
6. Evening line: built, **off by default** ("logged, but I don't want to see it").
7. Panel position: **the very top left**.
8. Voice: "add goal", "add goal <words>", "open goals".
