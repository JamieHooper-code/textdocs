---
tags: [ahk, chrome-extension, dom, design-doc, events, macros]
related: ["[[GLOBAL_CLICK_HINTS]]", "[[CHROME_EXTENSION_IDEAS]]", "[[CHROME_VIDEO_CONTROL]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[DOWNLOAD_MANAGER]]"]
status: BUILT 2026-10-18 -- generic event layer + topics tabs (migrated), page.ready, media, download, input.focus. Verified live.
created: 2026-10-18
---

# Chrome bridge events: listening to what happens in Chrome

The AHK Macro Bridge ([[GLOBAL_CLICK_HINTS]], "Chrome extension") was
request/response: AHK asks, the page answers. Jamie wanted the other direction
too: **everything the extension sees should be something any part of her setup
can listen to**, with no plumbing per feature.

Until 2026-10-18 there was exactly one push, and it was bespoke: tab changes
had their own message kind, their own `%TEMP%` file and a hardcoded wake to
Parent.ahk. Every new "tell me when X happens" would have needed another of
each. Now there is **one event shape and one path**, and a new topic is just an
`emit()` call in the extension.

> **Rule: new data from Chrome = a new event topic on this layer. Never a
> bespoke push, a new file, or a new message.**

## Using it

**AHK, from any persistent process** (always-on scripts, GuiHost, a Miller
viewer, a throwaway test script):

```ahk
ChromeBridge_On("media", (ev) => ToolTip(ev["data"]["event"] " " ev["data"]["title"]))
ChromeBridge_Off("media")             ; all handlers, or ChromeBridge_Off("media", fn)
ev := ChromeBridge_Latest("download") ; newest event, no subscription (a FileRead)
```

Includes: `Helpers\ChromeBridgeEvents.ahk` (On/Off) needs `ChromeBridge.ahk`
(Latest), `HttpFunctions.ahk` and `JsonFunctions.ahk`, nothing else.
`MAINFUNCTIONS` and the always-on closure already have it. `ev` is a Map:
`seq, topic, at, data, tab{id, windowId, url, title}, host_pid`. Topic `"*"`
means every topic.

**Python:** long-poll.

```python
import json, urllib.request as u
s = json.load(u.urlopen("http://127.0.0.1:8299/events?timeout=0"))["seq"]  # "now"
while True:
    r = json.load(u.urlopen(f"http://127.0.0.1:8299/events?since={s}&timeout=25", timeout=40))
    s = r["seq"]
    for e in r["events"]: print(e["topic"], e["data"])
```

Or `py Scripts\chrome_bridge\bridge_events.py listen --topics media,download`.

**Shell / anything:** read the day's log, `C:\Users\jamie\MacroData\chrome_events\<yyyy-MM-dd>.jsonl`,
or the latest per topic in `...\chrome_events\state\<topic>.json`.

**Diagnostics:**
- `MAINFUN.bat ShowChromeEvents`: latest per topic and who is listening
  (→ `%TEMP%\chrome_events.txt`).
- `bridge_events.py tail [--topic X] [-n 20]`, `latest <topic>`, `subscribers`, `prune [--days N]`.
- `GET /state` (every topic) or `GET /state?topic=X`. `GET /status` now carries
  `events_seq` and the subscriber count.

## The shape

```
{kind:'event', topic, data, at, tab:{id, windowId, url, title}}
```

- **background.js** builds it in `emit(topic, data, tab)`. When the host is
  down, events wait in a 200-deep queue and flush on `hello-ack`.
- **content.js** sends `chrome.runtime.sendMessage({kind:'event', topic, data})`.
  The service worker passes it to the same `emit()` with the sender's tab.
- **bridge_host.py** passes every `kind:'event'` to `bridge_events.EventHub`.
  Neither the host nor the hub branches on a topic.
- **The hub** adds `seq` (one counter across topics) and `host_pid`, then:
  1. appends the line to the day's jsonl;
  2. rewrites `state\<topic>.json` (atomic);
  3. wakes AHK subscribers (below) and long-pollers (a Condition);
  4. queues it for in-host consumers.

  Topics must match `^[a-z][a-z0-9_.-]{0,47}$`: they become filenames.

**seq survives host restarts.** The hub resumes from the highest seq in
`state\*.json`, so a Python `since` cursor stays valid across extension
reloads. A cursor from the future (state dir wiped) gets `reset: true` and
starts over. One that fell out of the 2000-event ring gets `gap: true`.

**Retention:** setting `chromebridge.event_keep_days` (default 30). Old day
files are deleted at host start and at each midnight rollover. Everything is
local on the SSD. Note the log holds the URL and title of every page she
loads; that is what makes it replayable, and it never leaves the machine.

## Delivery to AHK: one window message + a subscriber directory

- **Subscribing.** `ChromeBridge_On` writes
  `chrome_events\subscribers\<A_ScriptHwnd>.json` = `{hwnd, pid, topics,
  script, at}`.
- **Waking.** On each event the host PostMessages the registered message
  **`AHK_ChromeBridge_Event`** to every subscriber of that topic (or of `*`),
  with `wParam = crc32(topic) & 0x7FFFFFFF` and `lParam = seq`.

**Why a directory of files and not `POST /subscribe`:**
- **It survives the host restarting.** Chrome restarts the host with every
  extension reload and browser restart. An in-memory list would come back
  empty, and every always-on listener would go deaf without a sound.
- **It works before the host exists.** Parent.ahk boots before Chrome does.
- **No locking.** One file per window means no two processes ever write the
  same file, so there is nothing to lock and no lost update.

**Dead entries are pruned by the host.** On every publish it checks
`IsWindow(hwnd)` AND that the window still belongs to the pid in the file:
hwnds get reused, and a reused one would otherwise deliver another program's
messages. A failed PostMessage also prunes. AHK deletes its own file `OnExit`.
Pruning was verified live by killing a click-hints host: the next event
removed its file (`events: subscriber pruned` in `ahk_event.log`).

**Receiving (ChromeBridgeEvents.ahk).** The message handler only queues;
a `-1` timer drains the queue.
- The event is read from `state\<topic>.json`, so normally there is no HTTP.
- If the file has already moved PAST the message's seq (a burst: play then
  pause 20 ms apart), the missed ones are fetched in order with
  `GET /events?since=<last>&topics=<topic>`.
- If even that fails, the newest is delivered, so a burst coalesces rather
  than vanishing.

Every handler sees each seq exactly once and in order. A handler that throws
is logged and contained. The plan is a pure function (`_CBE_PlanDelivery`),
pinned by the tests.

**Not replayed:** a new subscription starts from "now", so a listener is
never handed the backlog. Read the jsonl for history.

`ChangeWindowMessageFilterEx` is called on the message, so an elevated AHK
still hears the unelevated host.

## In-host consumers

`Scripts\chrome_bridge\consumers\*.py`: any module with `TOPICS` and
`handle(event)` is loaded at host start and runs on one worker thread.
This is for work that should happen whenever Chrome is up, with no daemon to
manage. It is still generic: the host never names a topic. An exception is
logged, never fatal. First consumer: `downloads_consumer.py`.

## Topics

### `tabs` (background.js; migrated from the bespoke push)

Fires on `tabs.onActivated` / `onUpdated` (url, title, audible, load complete)
/ `onRemoved` / `windows.onFocusChanged`, debounced to 40 ms.

```
data: {why: activated|updated|removed|focus|hello, focusedWindowId,
       active: [tabFacts per window], audible: [tabFacts]}
tab:  the focused window's active tab
tabFacts = {id, windowId, url, title, audible, muted, active, index, incognito, pinned}
```

**The worked example.** `CtxDetector_Start` (Parent.ahk) calls
`ChromeBridge_On("tabs", _CtxDetector_OnChromeTabs)`, which wakes the detector
only when `ChromeBridge_FocusKey` (the focused window's windowId|tabId|url)
changes. That is what the host used to decide; it now lives with the listener.

`ChromeActiveAddress` reads `state\tabs.json` through `ChromeBridge_Snapshot()`.
It still uses a FileRead and raw `WinActive`, so the no-recursion rule holds,
and a dead host pid still means "ignore it". The old
`%TEMP%\chrome_bridge_tabs.json` and the `AHK_ChromeBridge_TabChange` message
are gone.

### `page.ready` (content.js)

The real load finished, or an SPA route settled. "Settled" means no DOM
mutation for 400 ms (capped at 4 s), so a listener that reads the page
afterwards sees the new route. Triggers:
- the first run (document_idle);
- the Navigation API's `currententrychange` (pushState / replaceState /
  back / forward);
- `popstate`, `hashchange`;
- YouTube's `yt-navigate-finish`;
- a bfcache `pageshow`.

One event per url.

```
data: {kind: load|spa|restore, url, title, host, settledMs, sinceNavMs, visible}
```

Live: YouTube and Google Voice route changes arrived as `kind: spa` (settled
in about 1.0 to 1.6 s).

### `media` (content.js)

`play` / `pause` / `ended` / `ratechange` / `seeked` on the page's MAIN media
element. That is `pickMedia()`, the same element `media.do` acts on (shared
with [[CHROME_VIDEO_CONTROL]], not duplicated), so a hover preview beside a
playing video is not reported.

`seeked` and `ratechange` are debounced (300 / 250 ms). Players in open shadow
roots get per-element listeners. A video that autoplayed before content.js
loaded is announced at page.ready as `{event:'play', atLoad:true}`.

```
data: {event, atLoad?, kind: video|audio, paused, ended, currentTime, duration,
       live, playbackRate, muted, volume, rateFrom, url, title, host, ok, found}
```

### `download` (background.js, `chrome.downloads`; manifest gained "downloads")

```
data: {event: created|filename|complete|interrupted|paused|resumed, id, url,
       finalUrl, referrer, filename, mime, state, error, totalBytes,
       bytesReceived, fileSize, startTime, endTime, danger, paused, incognito,
       byExtension}
tab:  {} (a download has no tab)
```

**Into the download manager** ([[DOWNLOAD_MANAGER]]): `downloads_consumer.py`
is a producer under its rules:
- **Guarded import**, and tracking never breaks anything.
- **Source `chrome`, stable id `<id>@<startTime>`.** Chrome's download id is
  persistent but per profile.
- **States:**
  - in progress → `active`;
  - `complete` → **`saved`**, a new DONE state in downloads.py. A browser
    download has no library step. Calling it `downloaded` would park every
    saved PDF in "never imported";
  - `interrupted` → `failed` at stage `download`, with Chrome's error in the note;
  - `USER_CANCELED` → `skipped`.
- **Kind:** `book` for ebook extensions, else `other`.

Live: an httpbin attachment went created → filename → complete and showed up
in `downloads.py list` as `saved`.

### `input.focus` (content.js)

A text field or contenteditable gained or lost focus. The value is never
included.

```
data: {why: focusin|focusout|ready|visible, url, visible,
       focused: true|false|null, kind: input|textarea|contenteditable|role|iframe|'',
       tag, type, id, name, label, password}
```

- `focused: null` means focus is inside an iframe this frame-0 script can't see
  (Google Docs). Callers treat that as "don't know".
- Coming back to a tab re-announces its state (`why: visible`), as does every
  page.ready.
- There is also an op, `focus.state`, for an on-demand answer.

**Click hints uses it.** `Scripts\ClickHintsHost.ahk` sets
`CH_TEXT_PROBE := ChromeBridge_InTextProbeStart()`, and the engine's 300 ms
watch calls `_CHInTextNow()`. That function asks the probe first and falls
back to the UIA `_CHFocusedIsTextField` on -1.

The probe keeps the last event PER TAB and answers for the active tab of the
foreground Chrome window (from the tabs snapshot). It gives -1 when:
- Chrome isn't in front;
- the bridge is down;
- the tab hasn't spoken since the probe started;
- focus is in an iframe.

A first version kept only the newest event overall and answered "don't know"
after every tab switch. The live test caught it.

The probe also gets contenteditable editors right (the ChatGPT composer),
which the UIA "edit" check misses. Coordinated with the click-hints session
before editing `ClickHintsEngine.ahk`.

### `pins` (content.js)

Which PINNED click numbers ("stay 13", see [[GLOBAL_CLICK_HINTS]] § Pinned
numbers) this tab answers to. Only tabs that have (or just lost) pins speak;
every other page stays quiet.

```
data: {url, title, visible, numbers: [13, 14, ...], names: [...]}
```

- Sent when the matched set changes (pins pushed, SPA route, the page
  re-rendering), and on every visibility change, so `visible: false` follows
  a tab switch away.
- **Caster reads the state file directly**: `rules\click_pins_commands.py`
  `_pins_here` stats `state\pins.json` per utterance (mtime-cached) and
  checks the foreground title starts with `data.title`. That is what scopes
  the bare-number rule to pinned pages with no AHK process in the loop.

## Extension reload behaviour

- **Re-injection.** On install/reload, `injectOpenTabs()` injects the fresh
  content.js into every open http(s)/file tab. Before this, every open tab went
  silent (orphaned script) until it was reloaded.
- **No fake page loads.** A copy injected over an orphan must not announce
  page.ready for 12 tabs that didn't load. The marker is the
  `data-ahk-bridge` attribute on `<html>`, not a `window` flag. **Measured:** a
  reloaded extension's script runs in a NEW isolated world and cannot see the
  old copy's globals, so a window flag missed all 12 tabs.

## Tests

- **Python:** `bridge_events.py selftest` (14 checks):
  - seq, the bad-topic refusal, the event shape, one line per event, latest;
  - posting only to subscribers of that topic or `*`, dead-hwnd pruning;
  - since / topic filter, long-poll wake, future-cursor reset;
  - seq resume after restart, prune keeps N days, consumers contained, the
    topic-id formula.
- **Python:** `consumers/downloads_consumer.py` (7) and `bridge_host.py
  --selftest`. All run in pytest as `test_chrome_bridge_events.py`.
- **AHK:** `Helpers\Tests\test_chrome_bridge_events.ahk`:
  - topic ids equal Python's;
  - topic validation;
  - the delivery plan (direct / burst / coalesce);
  - dispatch exactly once with a throwing handler contained;
  - the input.focus answer;
  - `ChromeBridge_FocusKey`.

## Verified live 2026-10-18

- A fresh AHK script with ONE `ChromeBridge_On("media", ...)` logged
  pause / play / seeked / pause as the bridge drove a test video.
- The Python one-liner long-poll printed tabs and download events as they
  happened.
- The day's jsonl holds every event (tabs, page.ready, media, download,
  input.focus).
- Parent.ahk re-subscribed to `tabs` after `ReloadWithNotice`.
- Not re-tested live after the migration: a url change under an unchanged
  title flipping a url context (Chrome was in use, and Jamie declined screen
  control). That path is the unchanged `_CtxDetector_OnChromeUrl`, now fed by
  the event, and `ChromeBridge_FocusKey` is unit-tested.

## Ideas this unlocks

These are now just topics. See [[CHROME_EXTENSION_IDEAS]].
- `selection` (grab quote / journal with no clipboard);
- time-on-page (`page.visible` / `page.hidden`, for the context log);
- the Google Voice in-page watcher (`gv.message`);
- replacing `WaitForScrapeReady` sleeps with `page.ready`.
