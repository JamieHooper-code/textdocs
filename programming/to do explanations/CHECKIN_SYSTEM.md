---
tags: [programming, design-doc, health, checkin, ahk, caster, gui, completion-log, activities]
created: 2026-10-19
updated: 2026-10-19
status: designed + built 2026-10-19 (one question window; check-ins, "log X" and "add activity" moved onto it)
related: ["[[HEALTH_SYSTEM]]", "[[COMPLETION_LOG]]", "[[QUESTIONS_SYSTEM]]"]
---

# One question window

Jamie, 2026-10-19: *"I want to standardize this check-in processing user
interface so that it's the same everywhere and different things can just use
it."* Before this there were three ways of asking her questions:

| Where | What it was |
|---|---|
| Morning / evening check-in | one window, digits answer straight away (the good one) |
| `log skating` | a NEW text box per question, type the number, Enter |
| `add activity` / `log <anything>` PgUp | text box, popup, popup per question; only 0-10 or short text |

*"It pops up a text box, which is fucking ridiculous, just to have me put in a
number."* Now there is ONE window, `QuestionWindow` in
`AutoHotkey/Helpers/Gui/QuestionWindow.ahk`, and every caller only hands it
questions.

## Shape: a window you ask one question at a time

```ahk
qw := QuestionWindow(Map("title", "Log: Skating"))
r := qw.Ask(question, Map("header", "Skating  ·  1 of 3"))
; r["status"]: answer | skip | back | done | close ; r["value"], r["note"]
qw.Close()
```

The window STAYS OPEN between questions (no flashing boxes) and `Ask` returns
when she answers. Callers stay linear code: the check-in is a loop with
back/skip/done, an activity log is a for-loop, building a new activity is a
loop that adds questions as she goes. That is the whole reason for the
"ask one at a time" shape over "hand it a list": the flows that need to branch
(splice the questions of a ticked activity in, ask who she hung out with,
keep adding questions until she stops) branch in their own code.

**A fixed list needs no loop (2026-10-28):** `res := qw.AskAll(questions,
Map("header", name))` asks them in order with PgUp back, 0 skip, 0 0 stop
(keeps answers) and Esc-on-the-first = cancel already handled, and returns
`Map(ok, answers: key -> value, notes, ended)`. This is the default way any new
feature asks Jamie a few things: a list of question Maps, never a new GUI
(hand-built ones are hard for her to drive by voice and keyboard). Reference
caller: `MarkPoem` (`Helpers/PoemMarks.ahk`, voice "mark poem") -- rank (scale,
`ends` = ["weakest", "best"] labels the scale), status (choice, Enter keeps the
current one), special / in progress (multi), title (text, pre-filled), saved as
one `poems.py set-marks` write.

## Answer types (data, `kind` on the question)

| kind | keys |
|---|---|
| `scale` (1-10) | 1-9 answer at once. Enter = 10. Arrows ← 2 · ↓ 4 · → 6 · ↑ 8. 0 = skip. A letter as the first key starts a NOTE in the box (no Tab); Enter keeps the note, then answer. |
| `duration` | numbered list from `INIDATA/durations.json`. A number picks (waits a beat only when a longer number could still follow: 4 when there are 40+ rows). Tab = type one ("80", "1h20", "1:20", "1.5h"). Answer = minutes. |
| `time_range` | 15-minute times in columns, the guess marked. Enter alone = "that's right". N Enter M Enter = start and end (either order). Voice "4 to 28" does both. |
| `choice` | numbered options, a number picks. `other: true` adds "Something else..." (or Tab) to type one; the answer is then `Map(other: text)`. Built 2026-11-13 for today's goal ([[DAILY_GOALS]]). |
| `multi` | numbered options to tick; the last can be "Something else..." (typed). Enter finishes. |
| `text` | box focused, Enter saves, Enter on empty skips. |

Everywhere: **0** skip · **0 0** done (check-in: done for today; an activity
log: stop asking, keep what's answered) · **PgUp** back one question · **Esc**
close (answers so far kept).

**Scale is 1-10 everywhere** with 0 = skip (the activity questions used to be
0-10 typed; "elbow cost" now reads 1 as none). Enter = 10 is what let "1"
answer at once — the old "1 waits a beat for 1 1 = ten" is gone.

**Not always on top** (Jamie, 2026-10-19). It opens activated, so it starts in
front; it just doesn't sit over everything after she clicks away.

**Never shows yesterday's answer** — it would pull her toward the same number.

## Voice

`rules/health_checkin_commands.py` is scoped to any window whose title ends in
`  ·  questions` (the shared suffix — so `Log: Skating` gets it free). A text
box with text in it adds ` (writing)` to the title, which switches the rule off
so dictation reaches the box. Phrases: `<n>` (ten = Enter), `<n> to <m>`,
`skip`, `done for today`, `done | next`, `back`, `close check-in`.

## The time-lengths file

`INIDATA/durations.json` is THE list of time lengths (Jamie thought there was
one; there wasn't — the lockout menu had its own hard-coded 5..240). Any
duration question and the wait-gate lock menu read it. Edit freely.

## Sleep times

One `time_range` from **2 hours before the bedtime guess to 2 hours after the
wake guess** (*"I have a huge screen. It won't be a problem."*), every 15
minutes, in columns. Enter = the guess is right. Night wakings, when the power
log saw any, are a separate question after it.

## The questions (INIDATA/health_questions.json)

Every question has a time frame and the evening one is **today**: *"Getting
this snapshot right as I'm going to bed is not useful to me at all."*

- **Morning** — only what is knowable on waking: sleep times, how well you
  slept, how rested you feel, last night's wind-down, elbows *this morning*
  (stiffness after a skating day is recovery data), and sometimes what you're
  hoping to get done. "How calm / how good do you feel right now" were dropped:
  the first five minutes awake are noise.
- **Evening** — elbows today (pain still first), anything intense today, how
  today was, how calm you felt today, how well you paced yourself today,
  energy today, how your body felt today.

**Old answers that meant something else were deleted** (2026-10-19, Jamie:
*"you can just delete all the old data... we are still iterating"*): every
snapshot elbow / calm / feel / body answer from 2026-10-13..18. Sleep times,
ritual, rested, "how was today", energy and activities stayed.

## Activities

`INIDATA/activities.json` question kinds are now `scale | text | duration |
time_range | choice` (`activities.py q-add --kind ... --option ...`). A
`time_range` question ("when?") is anchored on NOW: the last few hours in 15
minutes, guess = ending now. Its answer is `HH:MM-HH:MM` and the log also gets
`<key>_min`, so a nap asks "when?" and gets its length free.

Building one (`add activity`, the PgUp in `log <anything>`, a typed new one in
the picker) runs in the same window: name → voice command → questions, each
question = wording (text) → kind (choice) → 10 good/bad (choice, scales only)
→ options (choice only, one per question, empty finishes).

## Not done / later

- Typing an exact time in a time range (the 2-hour margin covers it so far).
- The question store ([[QUESTIONS_SYSTEM]]) designs a `checkin` kind in; the
  health + activity questions are not in it yet.
