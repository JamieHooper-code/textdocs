---
tags: [design, lockout, meditation, context, architecture]
related: ["[[TIMER_OVERLAY]]", "[[SETTINGS_SYSTEM]]", "[[COMPLETION_LOG]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[CONTEXT_LOG]]", "[[VOICE_COMMAND_SYSTEM]]"]
status: built (v1) — 2026-08-05
updated: 2026-08-18
---

# Exception lockouts, the meditation envelope, and retargeting mid-session

**Status: BUILT.** Voice: **"lockout book"**, **"meditate book"**, **"meditate
IFS"**, **"meditate guide"**. Keys: **+/=** and **−** during any lockout.

Four changes shipped together on 2026-08-05 because they interlock — the `+` key
writes to the log, so what a meditation's logged length *means* had to be settled
first, and that same question decides the cool-down arithmetic.

## The problem (Jamie 2026-08-05)

Three separate frustrations, one root:

1. **A lockout was all-or-nothing.** Reading a book and writing poetry are things
   worth doing during a lockout; YouTube and programming are what it exists to
   bar. But the overlay claimed `Left`/`Right`/`Space` as global hotkeys and sat
   permanently on top, so a lockout made Kindle unusable — page-turn keys got
   eaten as track-skips.
2. **A planned meditation and an actual one drifted apart.** Sitting five extra
   minutes past the closing bell left the log saying she sat the planned amount.
3. **The sit had a warm-up but no way down.** 2 + 22 with an abrupt end.

## Decision 1 — the log records the ENVELOPE

The old `_LogMeditateCompletion` wrote the *spoken* number: "meditate 22" ran 24
minutes of wall clock and logged **22**. Jamie describes that sit as "the
24-minute meditation" and expected +5 to make it read 29.

**So the log now records warm-up + sit + cool-down — the wall clock.** The
overlay title states the same number, so the title and the log can never
disagree. This is the decision everything else hangs off.

## Decision 2 — cool-down is carved OUT, not added ON

Jamie's own arithmetic: *"currently it's 2 + 22. I want it to be 2 + 20 + 2."*
Both total 24. The cool-down comes out of the spoken block; **the total is
unchanged**, so no existing habit gets quietly longer.

```
meditate 22   ->   2 warm-up  +  20 sit  +  2 cool-down  =  24 total, logged 24
```

`cooldown_for()` clamps so the sit can never vanish on a short session — a
3-minute meditation yields the cool-down rather than leaving a 1-minute sit
bracketed by silence. Tested in `test_meditation_phases.py`.

### The bells

Warm-up is silence-then-bell×3. The mirror image is *not* symmetric, on purpose:

| Boundary | Bell |
|---|---|
| End of warm-up | ×3 — opens the sit |
| Every 10 min during the sit | ×1 (suppressed for guided styles) |
| End of the sit | **×1** |
| End of cool-down | **×3** |

Jamie's call, and it's the right one: the three-bell ceremony stays at the true
end, so **the session still closes exactly the way it always has**. The cool-down
is inserted without changing how the ending feels.

## Decision 3 — one key, two jobs

`+`/`=` (and `NumpadAdd`), split on which side of the target you're on:

- **Counting down** → move the target out a minute. Title repaints, and the
  eventual log entry inherits the new length.
- **Counting up** → the target is already spent, so "+" means *what I actually
  sat is the real length*: it rewrites the log entry that was already written.

`−` gives a minute back, and refuses to bring the target within a minute of time
already served — a lockout you can shorten to zero is not a lockout.

**Retargeting is IPC, not just display.** The audio process owns its own clock,
so the overlay writes the new absolute total to `timer_extend.txt` and TimerAudio
re-derives its phase boundaries. Absolute rather than a delta so a missed poll
can't accumulate an error. Without this the bells would still fire at the
original time and the display would be lying.

**Rewriting the log needs the record id**, which `clog add` didn't emit. Added
`--print-id`; the overlay redirects it to `meditate_log_id.txt` and the `+` key
feeds it to `clog edit`.

## Exception lockouts — the design

A profile is a normal lockout that **whitelists contexts**. Registry:
`INIDATA/lockout_profiles.json`, one row per exception:

```json
"book": {
  "label": "Book",
  "aliases": ["kindle"],
  "allow": ["kindle"],
  "open": "OpenKindleAt",
  "log_template": "reading"
}
```

### Why context TOKENS and not exe names

`allow` lists tokens from `INIDATA/Contexts/`, so a profile inherits everything
the context system already knows about identifying a window — exe, class, title
regex. Whitelisting Kindle and later whitelisting one Google Doc for a poetry
lockout are the same amount of work: **one JSON row, no AHK, no Caster rule, no
reboot.** That was the explicit requirement — "easy to add more of them".

### How the passthrough actually works

This is the load-bearing detail. A no-op handler would *swallow* the key; the
only way a key reaches Kindle untouched is for **no hotkey variant to be
eligible at all**. So the overlay's three hotkey scopes are all criterion-bearing
and registered most-specific-first:

1. `mirror` — arrows/Space drive Spotify
2. `picker` — the track picker owns navigation
3. `pack` — the normal lockout keys, scoped to `!_InAllowedApp()`, registered
   **last**

AHK fires the first *created* eligible variant, so mirror and picker still beat
pack. When a whitelisted window is in front, none of the three match, the key is
never hooked, and Kindle's own bindings work. Outside an exception lockout
`_InAllowedApp()` is `false` by construction (`g_AllowSpecs` is empty), so an
ordinary lockout behaves precisely as it always did.

**Escape and `+`/`−` stay unscoped.** They're lockout controls with no meaning
inside a reader, and being able to add a minute without leaving the book is the
point.

### Stacking — and the bug that clearing TOPMOST doesn't lower a window

Jamie chose: stays always-on-top normally, drops **only while** the whitelisted
app is in front. Driven by its own 250ms timer (started only for an exception
lockout) — a full second of the overlay sitting on the book after an alt-tab is
very visible. It compares against the window's *actual* style rather than a
cached flag, because `_ShowItemImage` destroys and rebuilds the loot tooltip with
`+AlwaysOnTop` on every reroll (and re-syncs immediately after showing, rather
than waiting up to 250ms for the poll to catch it).

**v1 shipped broken here.** Clearing `WS_EX_TOPMOST` does *not* lower a window —
Windows moves it to the **top of the non-topmost band**, which is still above the
app you just switched back to. It looked correct on the first tab-in only because
the overlay had never been raised above Kindle yet; on the second and every later
one it covered the book again, and so did the loot tooltips. The fix is an
explicit `WinMoveBottom` whenever the topmost flag is cleared. Safe during a
lockout because everything else is minimised.

### Controlling the music from inside the book

Handing the plain keys to the whitelisted app has a consequence that isn't
obvious until you use it: there is then **no way to touch the music at all**.
Every other window is minimised, so there is nowhere to tab to that would make
the pack keys eligible again — the lockout traps you in the one app that ignores
them.

So an exception lockout also registers **Ctrl+Alt** variants of every media key
(Space, arrows, M, F, S, P, R, L), unscoped, so they work inside Kindle. Mirror
variants are registered first for the same creation-order reason as the plain
keys. The overlay prints the binding under the song line — an invisible binding
is a binding she doesn't have.

### Music at launch

`pause_music` on the profile. Default **true for a meditation** (silence + bells
is the point of sitting with a book) and **false for a plain lockout** (which is
a music lockout like any other, so it mirrors whatever is playing).

## Books that ARE a practice

"meditate book" with No Bad Parts open is an IFS sit, and Jamie wanted it logged
as one without saying the word. Each book carries a `meditation_style` in the
library; `style_from_book` on the profile makes the launcher resolve the open
Kindle window to a book id and read that field.

```
Kindle window title
  -> quotes.py resolve-window-book --id      (the shared ASIN/title ladder)
  -> clog book-meditation-style <id>         (library.json)
  -> meditation_styles.json                  (label + bell behaviour)
```

`--id` was added to `resolve-window-book` because the existing form resolves
through the *quote store*, which can only name books she has already quoted from
— the wrong gate when the question is "what kind of book is this", since a book
with no quotes yet is still a book she is reading.

Every step is best-effort: no Kindle, an uncatalogued book, or a book with no
style all fall through to an ordinary meditation rather than blocking the sit.
Cost is two Python cold starts (~1.5s) before the overlay appears, which is in
line with the other lockout launchers.

Set it in the reading Miller: **open read → the book → Meditation style**. An
in-line pick list over the style registry with a real "(none)" row, following
`_ReadingBookGenreNodes` — a value from a known set is picked, never typed.

### Known limitation

A context matched **only by URL** degrades to its exe — the whole browser becomes
exempt rather than one tab. The overlay can't read Chrome's URL on the keypress
hot path, and a UIA query per page-turn would cost tens of milliseconds. A
context wanting tab-level precision should carry a `title_regex`; a Google Doc's
name is in its window title, so the poetry case works.

### Resume

`resume_state.ini` stores the profile and style **keys**, and `ResumeLockout`
re-resolves them from the registries. Without this, resuming a book lockout would
come back sealed and lock her out of the very book she was reading — the one
failure mode here that actually hurts.

## Meditation styles

Same registry shape, different axis:
`INIDATA/VoiceChoices/meditation_styles.json`. The key is *both* the spoken word
and the value written to the log's `style` field, so `meditate IFS` needs no
translation layer. `guided` is the generic recording-led style; the named ones
are specific practices. Phrasing is `meditate <style>` directly — **not**
`meditate guide <style>`, at Jamie's request.

`suppress_interim_bells` defaults true for the guided family (a recording paces
itself) and false for `silent`, which would lose its whole cadence without them.

A test asserts the registry keys and the completion template's `style` options
stay identical — otherwise the log grows values its own picker can never show.

## Files

| Layer | File |
|---|---|
| Registries | `INIDATA/lockout_profiles.json`, `INIDATA/VoiceChoices/meditation_styles.json` |
| Settings | `meditation.warmup_minutes`, `meditation.cooldown_minutes` |
| Launchers | `Helpers/LockoutTimerFunctions.ahk` |
| Overlay | `Scripts/LockoutTimer/TimerOverlay.ahk` |
| Bells | `Scripts/LockoutTimer/TimerAudio.py` |
| Log | `Scripts/completion_log/clog.py` (`--print-id`) |
| Voice | `rules/lockout_commands.py` |
| Tests | `Scripts/codebase_tools/tests/test_meditation_phases.py` |

## Still open

- **Guided meditation pack.** The backend is built and logs a style; the audio
  pack doesn't exist yet. When it does it plugs in as an audio source.
- **A poetry profile.** The mechanism is there; it needs a context with a
  `title_regex` for the doc.

## The work-session model (rewritten 2026-08-18)

*What decides whether the bell rings and the auto-lockout prompt fires.*
Code: `Helpers/MindfulnessBellPoller.ahk`, config `INIDATA/MindfulnessBell.ini`.

### The problem

Three bugs stacked into "it nags me when I am not programming":

1. **The gate was process existence.** `ProcessExist("Code.exe")` — VS Code
   parked in the background for three hours counted as programming.
2. **Activity was machine-wide.** The session clock advanced on any input
   anywhere, so turning Kindle pages counted. This is why a lockout was offered
   *while reading a book*.
3. **The bell ignored activity entirely** — it rang every 15 minutes as long as
   VS Code was running, which is the "I haven't programmed in an hour and it is
   still bonging at me" complaint.

And the 90-minute threshold was **wall-clock since session start**, so a session
opened at 1:00 hit 90 minutes at 2:30 regardless of what happened in between.

### The rule

A session is **LIVE** when both hold:

1. a **work context** was FOREGROUND within `visit_gap_minutes` (default 20), and
2. there was input **anywhere** within the same window.

While live it accrues wall-clock; `auto_lockout_threshold_minutes` (90) of that
fires the prompt. The bell only rings while live.

### Why visits, not keystrokes

The first draft gated on *active input inside VS Code*. Jamie corrected it:
**much of her programming is passive — waiting on an AI assistant — and that is
still work.** Checking in on VS Code every six minutes for ninety minutes IS a
ninety-minute session and should prompt.

So the signal is the **visit**: touch a work context and the session stays alive
for another `visit_gap_minutes`, however little was typed. Reading a book for two
hours registers no visits, so the session died 20 minutes in — no bell, no
prompt. No separate "reset after N minutes" rule is needed; it falls out.

Condition 2 exists solely to kill the asleep-at-the-keyboard case: a foreground
VS Code with nobody in the chair would otherwise hold the gap at zero forever.
Both conditions share one window, so there is a single number to tune.

`work_contexts` (default `code,typing_box,terminal,claude_web`) is a CSV of
context tokens — adding one is a config line. **Cursor.exe has no context token
yet**, so it only satisfies the `gate_processes` check and cannot register a
visit; add a `cursor` context if that becomes real.

The visit signal is the in-memory `CtxLog_SecondsSinceContext()` from
[[CONTEXT_LOG]] — never a read of the log file.

### The prompt counts UP and never closes itself

Leaving the prompt open **is** the break. At `prompt_break_after_minutes`
(default 10) the session is reset exactly as a real lockout would reset it — no
keypress required — and the window **stays open, still counting**, with its
message changed to say the break was credited.

That is the answer to *"the timer annoys me and I often don't want to do a
lockout"*: ignoring it became a valid way to take the break instead of a way to
be nagged again in 30 minutes. It does **not** lock the screen — it resets the
bookkeeping only.

Whatever is on the counter is **time already served**. Pressing Enter afterwards
passes it to the overlay as `--resume-total`, which needed no new overlay code:
`priorTotal` collapses `remaining` to `Max(0, target - credit)`, so a prompt sat
on for 15 minutes opens a 5-minute lockout **straight into excess time** with
Escape already unlocked. Entry point `StartLockoutTimerCredited()`.

### Retired

`idle_reset_minutes`, `auto_lockout_prompt_timeout_seconds`,
`auto_lockout_default_yes`, and the `MB_WasIdleLastPoll` global — all described a
count-DOWN and an idle-edge model that no longer exist. The three copy-pasted
session-reset blocks were folded into `_MB_ResetSession(nowYmd, reason)`; they
had drifted, and the VS-Code-closed path notably failed to clear
`lockout_dismissed_at`, so a dismissal could survive a full quit-and-reopen and
suppress the next prompt.

## Count-up lockouts ("lockout 0") — 2026-08-18

`StartLockoutTimer 0` starts an **open lockout**: no target, Escape unlocked
from the first second, and the big number counts UP. An open-ended sit you leave
when you leave, with loot accruing the whole time. The title says
`open lockout — <pack>` rather than the nonsense `0 min lockout`.

Almost all of this already existed. With `totalSeconds = 0` the overlay's
`timerDone` is true at init, so it opens already-done and the fresh-lockout
branch renders `overtime` counting from zero. The only thing that had to change
was the **loot ladder**.

### The loot delay

`GetOvertimeRarity` measures seconds past the target. With no target every
second is overtime, so a count-up sit would hand out grey instantly and white at
one minute — trivialising a ladder that exists to reward sitting longer than you
had to.

So a count-up lockout pushes the whole ladder back by `COUNTUP_RARITY_DELAY`:

| tier | normal (past target) | count-up |
|---|---|---|
| grey | 0–1 min | **0–5 min** |
| white | 1–5 min | **5–9 min** |
| green | 5–15 min | 9–19 min |
| blue | 15–30 min | 19–34 min |
| purple | 30–60 min | 34–64 min |
| orange | 60+ min | 64+ min |

**240 seconds, not 300.** The grey band is normally 60s wide; shifting by 240
makes it 300s wide and puts the grey→white boundary exactly on 5:00, which is
what Jamie asked for ("from 0 to 5 minutes it will be a grey item… at minute
five I get a white item"). Everything later is staggered by the same 240s.

`_LootSeconds(overtime)` is the single place this applies; both display branches
(fresh and resumed) route colour + rarity through it, while the on-screen
overtime stays the real number. It derives the delay from `totalSeconds = 0`
**live** rather than latching at startup, so retargeting a count-up sit with
`+`/`−` into a real countdown drops the delay automatically. `--rarity-delay N`
overrides it in any mode.

### The -1 sentinel

`StartLockoutTimer(minutes := -1)`: `-1` = pack default, `0` = count-up, `>0` =
literal minutes. It cannot be `0` = default, because AHK cannot tell "called
with no argument" from "called with an explicit 0" and **both spellings are
live** — voice `"lockout"` dispatches `StartLockoutTimer()` while the Stream
Deck button at 7,0 sends the literal `0`. `StartLockoutPack` keeps its own
`minutes <= 0 = pack default` rule (the picker and genre/tag entry points rely
on it), so the count-up path launches the overlay directly via
`_LaunchRotationLockout`.

**Two Stream Deck buttons pass `0`** (the same button on Deck A and B); they are
now open lockouts. No button uses the bare form, so nothing else shifted.

### Bug found while testing: the stale kill flag

`CancelLockoutTimer` writes `timer_kill.flag`, which the overlay polls once a
second and exits on. Its `timer_audio.pid` guard is supposed to stop the flag
being written with no overlay running — but **a crashed overlay leaves that pid
file behind**, so the guard passes, the flag is orphaned, and the *next* lockout
started dies about a second after opening. Symptom: "I said lockout and it
flashed and vanished."

Fixed at the consumer: the overlay now deletes any pre-existing `timer_kill.flag`
at startup. A flag raised before this process existed cannot have been meant for
it. Pre-existing bug, unrelated to count-up, but it is what the count-up
regression test tripped over.

