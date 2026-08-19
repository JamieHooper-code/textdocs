---
tags: [autohotkey, context, logging, activity-tracking, always-on, design, mindfulness]
related: ["[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[EXCEPTION_LOCKOUTS]]", "[[COMPLETION_LOG]]", "[[MEDIA_SYSTEM]]"]
created: 2026-08-18
status: built (v1) — 2026-08-18
owner: Jamie
---

# Context activity log

An append-only record of **what was in the foreground, for how long, and whether
Jamie was actually there** — one JSON Lines span per context stretch. Built as a
generic system other things plug into; its first consumer is the mindfulness /
lockout timer (see [[EXCEPTION_LOCKOUTS]]), but it is deliberately independent of
it.

Engine (writer) `AutoHotkey/Helpers/ContextLog.ahk` · reader
`AutoHotkey/Scripts/context_log/context_log.py` · store
`AutoHotkey/INIDATA/context_log/YYYY-MM-DD.jsonl` (gitignored) · registered from
`AlwaysOn/ContextDetector.ahk`.

## Why it is nearly free

It is a **passive rider** on [[ALWAYS_ON_CONTEXT_DETECTOR]], which already
resolves the foreground context chain every 60ms and already fans changes out to
observers. The log adds one `FileAppend` per *change* plus one integer update per
tick — nothing computes a context that wasn't being computed anyway.

Measured before building, from a real 26-hour stretch of `ahk_event.log`: **609
context changes, ≈24/hour, one per 2.5 minutes.** At ~200 bytes per span that is
~120 KB/day. The "is this overkill / will it cost me on every context switch?"
worry was worth checking and the answer is no by two orders of magnitude.

## The record

```json
{"start":"2026-08-18T14:03:22","end":"2026-08-18T14:03:25","dur_s":3,
 "ctx":"chrome","chain":"spotify,chrome","exe":"chrome.exe",
 "title":"...","active_s":3,"src":"ctx"}
```

A span is filed under the date it **started**, so one crossing midnight stays in
one piece.

## Four design decisions

### 1. Raw at write time; de-glitch at read time

Real data shows `[code] → [] → [code]` transitions **61ms apart**. The obvious
move is a minimum-dwell filter in the writer. That is wrong: a two-second glance
at VS Code is a **genuine check-in**, and under the visit-gap session model it is
exactly what keeps a work session alive. Filtering it at write time would destroy
the signal the timer depends on, irreversibly.

So the writer keeps every span verbatim and `context_log.py --deglitch N`
collapses noise at read time, where the threshold is tunable and nothing has been
lost. **`--deglitch` defaults to 0.**

### 2. Record exe + title even when no context matched

Roughly **40% of real transitions pass through the empty chain `[]`** — an
unregistered window, or focus in flight. A span reading only `ctx: ""` is
unanswerable. Recording the exe and window title keeps those spans meaningful and
doubles as the shortlist of contexts worth defining. The reader's `label_for()`
falls back to `(exe.exe)`.

### 3. `active_s` — present vs. actually there

The point Jamie asked for: *"track what times I am actually active versus when
something is just sitting open for three hours while I'm AFK."*

Each span carries `active_s`: wall time during which input had happened within
`CTXLOG_ACTIVE_GRACE_MS` (60s). A minute of grace bridges reading a paragraph or
thinking, and still marks a three-hour unattended window as idle.

Sampling is **per detector tick**, not on a coarse window. A 5-second sampling
window was tried first and mis-attributed badly across span boundaries — a
73-second stretch measured `active_s: 0` while a 3-second one measured 3.
Per-tick accounting has no boundary error and costs one subtract plus one Map
update.

**Known limitation, accepted:** passive media (watching a video, hands off) reads
as idle once the grace expires. `dur_s` still records presence, so the
present-vs-active pair stays honest. A media-aware producer can plug in via
`CtxLog_WriteSpan` if that ever matters.

### 4. The log is for analysis; behavior reads a live signal

Consumers that must decide *right now* call the in-memory
`CtxLog_SecondsSinceContext(tokens*)` and **never read the JSONL back**. One
concept, two code paths — so a slow disk, a rotated file, or a schema change can
never alter what the lockout timer does.

## Plugging in

**Write:** `CtxLog_WriteSpan(rec, dayYmd)` — any system can append a span-shaped
record; set `rec["src"]` to name the producer so readers can filter.

**Read live:** `CtxLog_SecondsSinceContext("code", "terminal")` → seconds since
any listed token was foreground; `0` = right now, `-1` = never seen.

**Query:** `py Scripts/context_log/context_log.py {spans|rollup|active}` with
`--date` / `--days` / `--deglitch` / `--json`.

```
py context_log.py rollup --days 7          # where the week went
py context_log.py rollup --by exe          # group by executable
py context_log.py spans --ctx code         # every VS Code stretch
py context_log.py active                   # present vs actually-there
```

## Crash + reload resilience

`INIDATA/context_log/_state.json` holds the open span and the per-token
`last_seen` map, rewritten every 30s and on every change (write-to-temp then
move, so a reader never sees a half-written file). On boot `CtxLog_Start()`:

1. closes any span orphaned by the previous process, tagged `src: "recovered"`
   (a crash costs at most 30s), and
2. **restores `last_seen`** — which is what stops a mid-session
   `ReloadWithNotice` from looking like a fresh boot and restarting the work
   session clock. That class of bug previously chopped reload-heavy afternoons
   into bogus 3-minute sessions.

## Traps

- **`UTF-8-RAW`, not `UTF-8`.** AHK writes a BOM when creating a UTF-8 file, and
  a BOM on line 1 makes Python's `json.loads` throw on the first record. Same
  trap the Spotify spool hit. The reader also opens `utf-8-sig` defensively.
- **A forced detector refresh splits a span** even when the context didn't
  change (`CtxDetector_Notify` fans out with `force=true`). Harmless by design —
  the reader merges adjacent same-context spans unconditionally.
- **Titles are recorded** (capped at 200 chars) and carry document names, email
  subjects and chat titles. Local-only and gitignored, but it is a real record.

## Future

The eventual goal is the wider question — *how long did I spend reading, or on
YouTube, today* — which is a reader-side aggregation over data already being
written. Natural next steps: daily rollups + a retention policy (raw ~90 days,
then compact to per-day per-context totals), and a Miller hub node.
