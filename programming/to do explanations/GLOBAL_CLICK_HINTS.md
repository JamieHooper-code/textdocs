---
tags: [todo, ahk, uia, caster, voice, accessibility, hints, design-doc, overlay, numpad, chrome-extension, dom]
related: ["[[CHROME_VIDEO_CONTROL]]", "[[CHROME_BRIDGE_EVENTS]]", "[[GRAB_TEXT_ENGINE]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[VOICE_COMMAND_SYSTEM]]", "[[ANNAS_ARCHIVE_PIPELINE]]", "[[UIA_RECORDER_AND_CODEGEN]]", "[[LOCAL_APP_HUB]]"]
status: REBUILT 2026-10-15 -- sayable stable numbers, voice/keys modes, always-on click journal, make record = a marked stretch of it. Chrome now numbers from the DOM via the AHK Macro Bridge extension (built 2026-10-15). Recording -> macro codegen built 2026-10-15 (recorded_macro.py + RecordedMacroStep). v1 (2026-07-27) history below.
created: 2026-07-27
---

# Global Click Hints — "make click"

## 2026-10-15 rebuild — read this first

Jamie never used v1: the numbers were ugly and high. v1 labels were prefix-free
one-handed pairs (11-15, 21-25 ... 66-60), then triples like `155`. They were
built for typing on the split keyboard and were terrible to SAY. The local pages'
jump labels (kit.js, 1-30 in reading order) showed what she actually wants.
Speech is atomic, so voice never needed prefix-freeness.

**Numbers.** Plain 1..N in reading order. Typed digits COMMIT ON ENTER (voice
sends digits + Enter, see `click_hints_active_commands.py`). `0` still never
starts a number, so the 00-05 control codes are unchanged. Not capped (Jamie:
no hiding past 40).

**Stable** (`_CHAssignNumbers`, pinned by `Helpers/Tests/test_click_hints_numbers.ahk`).
Fingerprint = type|name|AutomationId; a coarse 40px position is used only when
name and AID are both blank, and duplicates get `#2`, `#3` in reading order.
Three passes:
1. Session memo: anything still on screen keeps its number.
2. Remembered: an element clicked at least twice in this exe gets its old number
   (`click_numbers.json`).
3. Fill: everything else takes the lowest free number.

The CURRENT holder of a number owns it: when something else takes a vanished
element's number, the vanished element's memo entry is dropped, so it can never
come back and steal the number.

**Runs until dismissed; follows her** (fixed the same evening). In the Claude app
the overlay quit after every first click, because the click left focus in the
chat box and v1 exited whenever focus landed in a text field. Now:
- There's no exit on a text field and no inactivity timeout. Only Esc, Backspace
  or Delete end it.
- The keyboard digit/Enter hotkeys step aside while a text field has focus, so
  typing still works. They're gated on a cached `CH_IN_TEXT` flag, kept off the
  keyboard hook because a UIA call there could stall typing.
- Voice picks arrive by window message instead of keystrokes:
  `PostMessage(0x8001, n, action)` to the ClickHintsBar window, handled by
  `_CHOnVoiceMsg`. A number said mid-rescan waits for the scan to finish rather
  than being dropped.
- A 300ms watcher (`_CHWatch`) rescans when the foreground window changes, so
  the numbers follow her across apps. It stands down while a click is running.
- Verified live: pick by message, click into a text box (overlay stays, typed
  "42" lands in the box), switch windows (rescans and keeps going), quit by message.

**Two modes.**
- `make click` = voice: numbers only. No letter hotkeys are registered, so typing
  still works.
- `make click keys` = a number plus two letters on each badge ("12 SD"). Say the
  number or type the letters. Letters are DERIVED from the number (the Nth pair
  of the left-hand alphabet), so they're just as stable, and they fire immediately.

Parked: two-character labels like "6 Bravo" for very high numbers (Jamie: revisit
if needed).

**Look.** Same badge as kit.js `.kit-jl`: a dark pill (16161C) with a faint warm
edge and cream semibold text. The font is Segoe UI because Atkinson isn't
installed system-wide. The typed exact match turns amber (`.kit-hit`); badges
that can no longer match dim.

**Click journal — always on.** Code: `Helpers/ClickHints/ClickJournal.ahk`.
Store: `Cfg.MacroData\click_journal\`, on the SSD on purpose because it writes
on every click. Each session's `events.jsonl` has two kinds of line:
- `scan`: every numbered element (n, letters, fp, type, name, aid, rect), plus a
  JPEG of the window WITH the badges. The status bar is hidden for the capture.
- `pick`: read BEFORE the click. It holds:
  - the element's properties and its parent path (live, up to 12 levels);
  - `selector_matches`: how many raw elements share its Type+Name+AID (1 = safe);
  - up to 8 lookalikes (same name or AID), each with its own path;
  - a clean, badge-free close-up.

  Measured cost: about 16ms per pick.

`Scripts/click_journal/click_journal.py` is the ONLY reader:
- `stats`: what she clicks most.
- `prune`: deletes journal shots older than 7 days. It runs once a day, detached,
  from the first overlay of the day.
- `list`, `payload`, `selftest`.

Only overlay activity is journaled. There's no global keylogging outside "make record".

**make record** (`Helpers/UIAClickRecorder.ahk`) is now a MARKED STRETCH of the
journal.
- First "make record": `record-start` writes `click_journal\recording.json` (the
  bar shows ● REC). It then starts `UIAClickRecorderHost.ahk embedded` to catch
  keys, typing, taps and macros outside the overlay, and opens click hints.
- Second "make record": stops that host and closes the overlay. `record-stop`
  copies the steps and pictures into `MacroData\recordings\<id>\` (recording.json
  schema 1, `payload.md`, and `step_NN_before.jpg / _target.png / _after.jpg`).
  The payload goes on the clipboard and into the Claude slot.

For each step the payload gives:
- the selector, marked "unique" or "!! NOT UNIQUE -- N on screen";
- the parent path and lookalikes;
- what appeared, what disappeared, and any title change.

Verified live: a Spotify "Fleet Foxes" link reported 13 matches, plus the
`complementary "Now playing bar"` anchor that tells them apart. That is exactly
the data whose absence caused the account-switch trial-and-error.

**Recording -> macro (built 2026-10-15).**
`py Scripts/click_journal/recorded_macro.py build last <Name>` does a dry run;
add `--write` to save. It writes the steps to `INIDATA/recorded_macros/<Name>.json`
and makes a Key Action whose steps all call ONE runner, `RecordedMacroStep`
(`Helpers/RecordedMacroRunner.ahk`). Each step:
- finds the element by its selector;
- uses the path anchor when the step was NOT UNIQUE;
- waits for the elements that appeared as its result check;
- logs every signal.

A Chrome DOM pick becomes a `dom_click` step that carries the CSS selector. When
a step fails, the runner writes a dump shaped like a recording step and puts it
on the clipboard and in the Claude slot. The macro is a normal function, and
voice phrases are only offered. Full design: [[UIA_RECORDER_AND_CODEGEN]]
§ "Stage 5".

Two journal facts the codegen depends on:
- A recorded UIA name can be a BORROWED label. `_CHDeriveNames` writes a
  heading's text onto an unnamed card, and the journal records that label. The
  runner falls back to "an unnamed element of that type containing that heading".
- `other_activity` rows now carry `kind` + `value`, the exact typed text and key
  combo. The `text` line is truncated, so it can't be replayed.

## Chrome extension — DOM hints (built 2026-10-15)

In Chrome, "make click" numbers the page from the **DOM** instead of UIA. The
DOM knows what UIA guesses at: what is really on top, the URL, hrefs, aria
labels, and a CSS selector that can be checked unique. Everything outside
Chrome, and Chrome's own tabs/toolbar ("00 wider"), still uses UIA unchanged.

**Pieces.**
- Extension: `AutoHotkey/ChromeExtensions/macro-bridge/` ("AHK Macro Bridge",
  MV3, unpacked). `content.js` does the page work; `background.js` routes.
  The `key` in `manifest.json` pins the id to `kajmlamcgkjcifcmmnajgmfnnckcnoea`
  wherever the folder is loaded from.
- Native host: `Scripts/chrome_bridge/bridge_host.py`, registered by
  `install_bridge.py` (HKCU NativeMessagingHosts `com.jamie.ahk_bridge`;
  `install_bridge.py check` verifies).
- AHK client: `Helpers/ChromeBridge.ahk` (`ChromeBridge_Call(op, args, title)`,
  diagnostic `MAINFUN.bat ShowChromeBridgeStatus`).
- Click-hints DOM mode: `Helpers/ClickHints/ClickHintsDom.ahk`, loaded by the
  host next to the engine. `CHRescan` asks `_CHDomWanted` first, and anything
  that doesn't fit falls through to UIA.

**Transport: a native-messaging host that also listens on 127.0.0.1:8299.**
```
AHK --POST /cmd--> bridge_host.py --stdio--> background.js --tabs.sendMessage--> content.js
```
Why not a plain localhost server like the viewers: an MV3 service worker
can't accept an inbound connection, and it SLEEPS after about 30s idle. A
server would need the worker to poll or hold a socket open, and nothing wakes
it when AHK asks.
- `connectNative()` solves both. Chrome launches the host with a private stdio
  pipe, and an open native port keeps the worker alive (Chrome 105+).
- Chrome starts and stops the host with the browser, so there's no daemon to
  start, babysit or restart. A once-a-minute alarm, and every page load,
  reconnect it if it dies.
- The host's HTTP side follows `local_viewer.py`'s rules exactly. `POST /cmd`
  is served only with the server's own Origin, which AHK names itself, the
  same as `LocalViewer_Post`. `GET /status` is the only read. Any website can
  reach localhost, but none can forge Origin.

The host knows nothing about click hints. It forwards `{op, args, title}` and
returns the answer. **A new DOM feature is a new op in content.js plus an AHK
caller, never a host change.** That's what makes this "the extension space"
for macros, not a one-off.

**Second feature: video control** (2026-10-18). Play, pause, seek, speed,
volume, mute and captions on whatever video is playing, with Chrome in the
background, via ops `media.state` / `media.do` sent to the AUDIBLE tab. Its
own doc: [[CHROME_VIDEO_CONTROL]].

**What stays in AHK, on purpose.**
- The keyboard. The global digit / Enter / Esc hotkeys mean neither the page
  nor Vimium ever sees a digit, so kit's Ctrl+Alt+Shift+F9 off-screen-input
  trick isn't needed here.
- The ClickHintsBar, so the Caster scope
  `window_exists(title="ClickHintsBar")` is untouched. Voice works with no rule
  change: bare "N", plus right / double / hover / middle.
- The numbering. DOM hints go through the same `_CHAssignNumbers`: same
  fingerprint (type|name|AutomationId, where AutomationId = the DOM id), same
  session memo, same "current holder owns it" rule. Remembered numbers are
  keyed **per site** (`chrome:<host>` in `click_numbers.json`), not per
  chrome.exe.
- The real mouse click and the journal.

The extension only scans, draws and describes. Ops: `scan`, `draw`, `prefix`,
`state`, `pick`, `pointer`, `domclick`, `hide`, `clear`, `ping`, and for recorded-macro replay `query` (match count through `host >> inner`) and `selectorclick` (click exactly one match; a middle click on a link opens a background tab via the service worker).

**Scan rules (content.js).**
- Candidates:
  - real controls: links, buttons, inputs, select, summary, `label[for]`;
  - ARIA roles (button, link, tab, menuitem, option, checkbox, …);
  - `[onclick]`, `tabindex >= 0` and contenteditable;
  - the outermost `cursor:pointer` element, for JS-bound divs.
- Open shadow roots are searched too.
- Dropped:
  - disabled or invisible elements (`checkVisibility`);
  - anything off screen;
  - anything inside a stronger control;
  - a wrapper the same size as its child;
  - anything NOT ON TOP: `elementFromPoint` at the centre and two corners must
    land on it, which drops things under sticky headers and modals.
- Order is kit.js `readingOrder`: rows by top with half a row of tolerance,
  then left to right.
- "Name" approximates UIA's Name: aria-label → aria-labelledby → label /
  placeholder → innerText → img alt → title.

**Look.** The badges are the kit `.kit-jl` CSS verbatim (amber `.hit` for the
exact typed match, plus a dim `.dead`).
- They live in a closed shadow root, so page CSS can't restyle them, on a
  `pointer-events:none` layer, so they never eat a click.
- They straddle the top-left corner on short controls and sit inside tall ones.
- Dense rows slide right out of a collision.
- They follow their elements on scroll.
- On a local page, kit's own jump labels are hidden while the hints are up.

**Re-renders.**
- content.js marks itself dirty after a DOM mutation, scroll or resize, once
  the page has been quiet for 150ms.
- The host polls `state` every 400ms (`_CHDomPoll`) and rescans when dirty.
  Survivors keep their numbers, and new things take the lowest free ones.
- An identical signature skips the redraw, so a ticking clock doesn't make the
  badges flicker.
- After a click, the same stability rule as UIA applies: two identical scans
  in a row, and different from before the click.

**Aiming: verified, never trusted.** The page reports CSS-px rects. The
viewport's screen origin is worked out from the window rect plus
innerWidth×devicePixelRatio (dpr includes zoom; the viewport sits at the
bottom between equal side borders). That is pure and tested
(`_CHDomOriginFrom`).
- On every pick the real pointer moves there and content.js reports what the
  `pointermove` landed on.
- On a miss, the offset is learned from where it actually landed, and the
  pointer tries once more.
- If it still misses, a left click falls back to `el.click()`, logged as
  `ClickHints/dom`. A bad guess (docked devtools, odd frames) can't click the
  wrong thing.
- Chrome's `Chrome_RenderWidgetHostHWND` children were checked as a source for
  the origin, and rejected: on Chrome 155 every one of them is invisible with
  stale coordinates.

**Journal: the same event shape, plus DOM fields.** `scan` lines gain
`source: "dom"` and `url`, and each hint gains `tag` / `href` / `text` /
`aria_label` / `role`. A DOM `pick` line (`_CJPickDom`) has the same keys as a
UIA pick, plus:
- `url`;
- `element.selector`: a robust CSS selector built at pick time. It tries
  stable `#id`, data-testid / name / aria-label / title / placeholder, then
  `a[href]`, then walks up adding tag + stable classes + `:nth-of-type` until
  it's unique. Machine-made classes (`css-1x9q2zd`, `sc-…`, hashes) and
  generated ids are never used.
- `selector_matches`: `querySelectorAll` count at that moment, 1 = unique;
- `name_matches`: how many clickables share its type+name;
- `element.text` / `href` / `aria_label` / `outer` (outerHTML, 300 chars);
- `path`: DOM ancestors as tag / id / stable classes / landmark role;
- lookalikes: same name+type or same href, each with its own selector, href
  and path.

Shadow-DOM elements get `host >> inner` (Playwright notation). `click_journal.py`
renders all of it: `css: … -> unique`, url, href, aria-label, html, a CSS-ish
path, lookalike selectors, and `url -> …` as a result. Its `selftest` covers
it.

**Setup (once, done 2026-10-18).**
1. `py Scripts/chrome_bridge/install_bridge.py` registers the host and checks
   it the way Chrome sees it (see the packaged-app trap below).
2. chrome://extensions → Developer mode → Load unpacked →
   `AutoHotkey\ChromeExtensions\macro-bridge`. Chrome 137+ ignores
   `--load-extension`, so this first load is the one manual step.
3. Check it with `MAINFUN.bat ShowChromeBridgeStatus`.

**Managing it without touching chrome://extensions** (`Helpers/ChromeBridgeManage.ahk`):
- `MAINFUN.bat ChromeBridgeReloadExtension`: after ANY edit to the extension.
  It asks the extension to reload itself (`ext.reload`, answered by the service
  worker). If the bridge is down, it real-clicks the extension's own Reload
  button (`#dev-reload-button`; matched by AutomationId because Chrome's
  toolbar Reload has the same name). Either way it waits for a NEW host pid,
  and returns true when one connects.
- `MAINFUN.bat ChromeBridgeReloadTab`: reloads the front tab so it gets the new
  content.js.
- `MAINFUN.bat ChromeBridgeStatusPage`: opens `status.html` (the extension's own
  diagnostics page) and reads it through UIA into
  `%TEMP%\chrome_bridge_status.txt`. It shows connected or not and **the last
  error Chrome gave for the native host**. It works when the bridge is down,
  which is when it's needed.
- `MAINFUN.bat ChromeBridgeExtensionErrors`: copies the extension's error list
  into `%TEMP%\chrome_bridge_errors.txt`.

**Traps.**
- **The packaged-app registry trap (cost the first live run, 2026-10-18).**
  `install_bridge.py` was run from a shell inside the Claude desktop app, an
  MSIX package. Its `winreg` write to HKCU landed in the package's PRIVATE
  registry. Reading it back from the same shell said it was there, but Chrome
  couldn't see it, and the extension looped on "Specified native messaging
  host not found" (71 reconnects).
  - That is also why Anthropic's own host manifest lives under
    `Packages\Claude_…\LocalCache`.
  - The installer now writes and checks the key with a .bat launched through
    `explorer.exe`, which runs outside any package. Never go back to `winreg`
    there.
  - The same applies to ANY HKCU write meant for another program, made from a
    Claude desktop shell.
- **"Connected" means the host answered.** The service worker used to mark
  itself connected as soon as `connectNative()` didn't throw. A missing host
  fails asynchronously, so the host now sends `hello-ack`, and only that sets
  connected and resets the reconnect backoff.
- **A FocusChromeTab URL match is a substring match.** The status page is
  matched on `status.html`, never the extension id: the chrome://extensions
  details tab has the id in its URL too, and got "focused" instead.
- **Stale code.** Chrome holds `bridge_host.py` and the service worker for
  the life of the extension. Run `ChromeBridgeReloadExtension` after editing
  anything here, then `ChromeBridgeReloadTab`. The content.js guard lets a
  fresh copy replace an orphaned one after a reload.
- **chrome:// pages, the Web Store and PDFs can't be scripted.** The bridge
  answers "restricted page" and UIA takes over.
- **Iframes aren't numbered yet** (top frame only).
- **Two Chrome profiles** each start a host. The first one to bind :8299
  serves AHK, and the other idles (logged).
- If the bridge doesn't answer, DOM mode backs off for 20s
  (`CH_DOM_DOWN_AT`), so a closed Chrome never makes "make click" wait on it.
- **New page, numbering.** On a URL change, `_CHDomForgetPositional` drops
  session numbers that only meant "the 2nd 'edit' link" or "whatever was at
  this spot". Kept, they scattered the new page's numbers: a Talk page's
  section "edit" links came up as 36 / 81 / 92 instead of following reading
  order. Uniquely named things (Search, Article, Talk) keep their numbers
  across pages.

**Live test (2026-10-18, Chrome 155, portrait right monitor).**
- Wikipedia "AutoHotkey": 102 hints, DOM scan in 78ms.
- Picking 7 ("Talk") by the voice channel: the real pointer hit on the first
  try, with no aim correction and no DOM-click fallback. The page changed and
  was renumbered (64 → 67 hints as it settled), and Article / Talk / Read /
  Edit kept 6 / 7 / 8 / 9.
- A `make record` of one pick on section "edit" #81 produced:
  - `css: a[title="Edit section: Error in script?"] -> unique`;
  - "3 clickables share its type+name";
  - two lookalikes with their own selectors and hrefs;
  - a CSS-ish path;
  - `url ->` the edit page;
  - a target crop exactly on the link.
- Quit through the voice channel cleared the badges (`state.hasBadges = false`).

**Replay ops for recorded macros** (asked for by the recording→macro session):
- `query {selector}` → `{ok:1, count}` (+ name / type / rect when count is 1).
  `host >> inner` pierces shadow roots.
- `selectorclick {selector, action}` clicks only when exactly one element
  matches, and otherwise returns `{ok:0, count, error}`. A middle click on a
  link opens a background tab through the service worker.
- Caller: `Helpers/RecordedMacroRunner.ahk`.

**Testing.**
- Pure: `Helpers/Tests/test_click_hints_dom.ahk` covers the origin math,
  CSS→screen scaling, numbers staying stable across a rescan with a new
  element above, remembered numbers being per site, and the new-page forget.
- Page: `ChromeExtensions/macro-bridge/test/fixture.html` has every awkward
  shape (3× "Reply", a fixed header, a cursor:pointer div, a shadow-DOM
  button, generated classes, disabled, off-screen).
- Host: `bridge_host.py --selftest` (framing + origin rule).

### Tab facts and URL events (built 2026-10-18)

This replaces three UIA workarounds for reading Chrome. UIA shows every tab's
NAME, but a URL only for the SELECTED tab, and only through the address bar.
A fullscreen video has no address bar, so the old fallback was Ctrl+L, which
flashes and drops her out of fullscreen. Reading the audible tab's URL meant
selecting it first. The extension sees every tab's url, title and audible
state directly.

**Service-worker ops** (answered in background.js; no content script, so they
work on any page, chrome:// included):
- `ext.active {title}`: the active tab of the window AHK names, matched the
  same way as every other op (`pickTab`).
- `ext.tabs`: every tab, plus the focused window id.
- `ext.focus {url, title, focusWindow}`: select the first tab whose url (else
  title) contains the text, case-insensitive, in ANY window. It does not raise
  the window: AHK does that (below).

**AHK side, callers unchanged** (`Helpers/ChromeBridge.ahk` holds the client
functions and the pure pickers):
- `ChromeCurrentUrl()`: bridge (`ChromeBridgeActiveUrl`), then
  `ChromeActiveAddress`, then Ctrl+L. It targets the foreground Chrome window
  by its title.
- `ChromeActiveAddress()`: the pushed snapshot (a FileRead), then UIA. It still
  uses raw `WinActive`, and the snapshot reader calls nothing context-related,
  so the DetectContextChain no-recursion rule holds.
- `FocusChromeTab()`: `ext.focus` selects the tab, then
  `_CBChromeHwndForTabTitle` finds its window by title across desktops and
  `_FocusChromeWindow` raises it. That keeps the virtual-desktop pull, which
  Chrome's own `windows.update({focused})` can't do. That call is only the
  fallback when no hwnd matches. If the bridge doesn't answer or finds no match,
  the old UIA passes run, so tabs in incognito or another profile are still found.
- `ChromeAudibleTab()`: the soundtrack tab's url without selecting it. Same
  rule as before: audible and not muted, and an ambience-titled tab wins.
  "grab music" in Kindle and the reading room now leaves focus where it was.
  `ChromeSelectAudibleTab` is the no-bridge fallback.

> **Superseded 2026-10-18 by the generic event layer:** this push is now topic
> `tabs`, the snapshot lives in `MacroData\chrome_events\state\tabs.json` and
> the detector subscribes with `ChromeBridge_On("tabs")`. The reasoning below
> still holds. See "Events" just after this section and [[CHROME_BRIDGE_EVENTS]].

**URL events: pushed snapshot + a posted window message.** On
`tabs.onActivated` / `onUpdated` (url, title, audible, load complete) /
`onRemoved` / `windows.onFocusChanged`, the extension sends `{kind:"tabs"}`,
debounced to 40 ms. The snapshot holds each window's active tab, the audible
tabs and the focused window. bridge_host.py:
1. writes it atomically to `%TEMP%\chrome_bridge_tabs.json` with its own pid;
2. when the focused window's (windowId, tabId, url) changed, PostMessages
   `AHK_ChromeBridge_TabChange` to Parent.ahk's hidden window. Parent hosts the
   detector.

The detector (`_CtxDetector_OnChromeUrl`) schedules a tick that re-detects
WITHOUT force. Observers only hear about a real chain change (logged as reason
`chrome-url`), and the url comes from the snapshot rather than UIA.

Why this delivery:
- **A message is the event and the file is for reads.** A flag file nobody
  is woken to read is a poll by another name (process-traps.md "Prefer events
  over polling").
- **The existing `CtxDetector_Notify` was not reused:** a notify FORCES a
  fan-out, which would rebind passthrough on every YouTube navigation.
- **What this adds:** the title-change WinEvent already caught tab switches.
  What it missed was a url change under an unchanged title (SPA routes,
  same-title pages).

Guards:
- The snapshot is ignored when its host pid is dead, so a stale file can't
  pin a context.
- It is matched to the foreground window by title: an exact match, or the tab
  title followed by `" - "`, with the longest title winning. A miss (the title
  changed before the snapshot did) falls through to UIA.
- The 2 s backup poll and the UIA read are unchanged.

**Verified live 2026-10-18:**
- A fullscreen YouTube tab returned its URL with fullscreen still on before
  and after (`ShowChromeTabFacts`).
- A background audible tab's URL was read without selecting it.
- `FocusChromeTab` worked across two Chrome windows: an already-selected tab,
  a background tab in the other window (selected and raised), and the
  no-match fallback to UIA.
- example.org → `#bridgetest` → `#other` (same title "Example Domain") flipped
  a temporary url context both ways with reason `chrome-url`.
- Not tested live: a Chrome cloaked on another virtual desktop. That path is
  the unchanged `_FocusChromeWindow`.

**Tests:**
- `Helpers/Tests/test_chrome_bridge_tabs.ahk`: title matching, longest/tie
  pick, and the audible pick (mute, ambience).
- `bridge_host.py --selftest`: focus key and atomic snapshot.

**Diagnostic:** `MAINFUN.bat ShowChromeTabFacts` → `%TEMP%\chrome_tab_facts.txt`.

### Events: listening to Chrome (built 2026-10-18)

Everything the extension pushes goes through ONE generic layer: the shape
`{topic, data, at, tab}`, a day log
`MacroData\chrome_events\<date>.jsonl`, the latest event per topic, and live
delivery to AHK (`ChromeBridge_On(topic, fn)`) and Python
(`GET /events?since=`).

Topics:
- `tabs` (the push above, migrated);
- `page.ready` (load + SPA routes, once the DOM settles);
- `media` (play / pause / ended / rate / seek);
- `download` (feeds the download manager);
- `input.focus` (click hints uses it instead of the UIA text-field guess).

Rule: new data = a new topic, never a bespoke push. Full design, data shapes
and tests: [[CHROME_BRIDGE_EVENTS]].

## Pinned numbers — "stay 13" (built 2026-10-19)

**What it is.** While click hints are up, saying **"stay 13"** pins hint 13: on
that kind of page, 13 always means that element. By default the number stays
drawn on the page with no "make click" running, and saying the bare **"13"**
clicks it. Built for Amazon, where Vimium keys misbehave and "Add to Cart" is
the button she always wants.

**UPDATE, same day: the pin window is gone; "open pins" is a Miller.** Jamie
flagged the picker as a lazy UI. It closed and reopened its window on every
toggle, and needed a "bare Enter always saves" workaround. Now:

- **`Helpers\ClickPinsMenu.ahk` + `Scripts\ClickPinsViewer.ahk`** ("Click
  Pins", its own process, scaffolded). The tree is site / app → pin → its
  settings: All like this · Always show · Look › (Normal / Minimal ✓) ·
  Which pages › (kind / page / site ✓, or any window / this title) · Number ·
  Name · Delete (asks first through `_ConfirmationModalGui`).
- **Every setting saves the moment it changes.** A leaf that doesn't close
  makes the engine re-run that level in place. Builders re-read the store via
  `ClickPins_Get`, so there is no Save step and Esc loses nothing.
- **Values from a set are picked from a list, never cycled or typed** (the
  gui-conventions rule). Only Number and Name are typed.
- **"stay N" pins at once** with the defaults: the lowest free number plus the
  `clickpins.*` settings. It shows a "pinned 1 — …" tooltip and opens the
  Miller drilled straight into the new pin (`OpenClickPins(pinId)` →
  `initial_path`). The family it saw is kept as `family_offer`, so "All like
  this" can be turned on later from the Miller.
- **Settings (⚙ at the root):** `clickpins.default_always`,
  `clickpins.default_look`, `clickpins.default_pages`,
  `clickpins.host_scan_ms` (the app-pin host reads it).
- Edits go through `ClickPins_Update(id, fn)` / `ClickPins_Delete` /
  `_CPSetGroup` / `_CPSetPages` in ClickPins.ahk.
- The old pin-window voice rule (`click_pin_window_commands.py`) is deleted.
  The Miller is voice-pickable for free ("3", "look", ...).
- **Verified:** opening on a pin; "1" turned All like this on (1–7, family
  selector applied); Always show toggled; Look › Minimal ✓ redrew in place;
  Delete › confirm emptied the store; "stay" with a test draft pinned it as 1
  (not the hint's 13) and opened on it. GUI tests: 2/2.

The historical design of the replaced picker window follows.

**The pin window** (numpad picker, `_PersistentLoopPickGui`; the title is
`Pin 13 — amazon.com`):

| # | Line | Notes |
|---|---|---|
| 1 | Save | **Bare Enter always saves**, whatever was just changed (the selection snaps back to line 1 every loop; an early build left it on the changed line, and Enter re-toggled it) |
| 2 | Always show | on = drawn on the page all the time; off = only reserves the number inside make click |
| 3 | Number | any number, not just the one she said |
| 4 | Look | normal (amber-bordered badge) / minimal (small amber number, no box) |
| 5 | Which pages | cycles: this kind of page → this page only → whole site |
| 6 | Name | shown in "open pins" |
| 7 | All like this | only when the element has a family: number all N of them n, n+1, ... |
| 8 | Delete | existing pins only |

Saying "stay N" on an already-pinned element opens THAT pin for editing.
Voice inside the window: bare numbers, "save pin", "pin cancel"
(`rules\click_pin_window_commands.py`, scoped by the window title).

**"This kind of page"** keeps only the path words that name a kind of page:
short, lowercase, no digits, at most one hyphen. So
`/Laptop-Stand/dp/B0ABC/ref=sr_1_3` becomes `(^|/)dp(/|$)`, which matches any
product page and not the cart. Pure and tested (`_CPKindPathRe`,
`Helpers\Tests	est_click_pins.ahk`).

**"All like this"** (`pins.family` in content.js): drop one `:nth-of-type` at a
time from the element's selector, deepest first, and also try tag plus stable
classes. The smallest family of 2..60 same-tag, visible elements wins. On
Amazon this gave `span.a-button.a-button-thumbnail`, 7 thumbnails, numbered
13–19. A single pin keeps its exact selector; if that selector later matches
several, the one with the pinned name wins.

**Amazon trap:** `a-autoid-N` ids are handed out in page order, so the same
id is a different button on the next product. `stableId` in content.js now
rejects any id containing `autoid`, which also protects recorded macros.

**Pieces:**
- Store: `INIDATA\click_pins.json`. The only writer is `Helpers\ClickPins.ahk`.
- Copy: `pins.set` puts it in `chrome.storage.local`, so every tab and every
  browser start has it. If the JSON is edited by hand, say "sync pins".
- Page: content.js § pins draws the badges in their own closed shadow root,
  which steps aside while make click is drawing. `pins.press` clicks by number
  with a DOM click, so the pointer stays put and it works when scrolled off
  screen. The page emits the `pins` event topic.
- make click: `scan` tags candidates with `pin`, and pass 0 of
  `_CHAssignNumbers` gives them their number before the session memo or the
  remembered numbers can.
- "stay N" is window message code 6 → `_CHStay` (ClickHintsDom.ahk), which
  writes `%TEMP%\click_pin_draft.json`, quits the overlay (its digit hotkeys
  would eat the window's input) and runs `MAINFUN ClickPinFromHint`.
- Bare "N": `rules\click_pins_commands.py` → `ClickPinPress`. It is scoped by
  the `pins` event state file plus the foreground title, and is off while the
  overlay is up.
- "open pins" / "sync pins" are generic-store commands.

**Verified live 2026-10-19** on an Echo Dot product page. The 7 thumbnails
were pinned as a group 13–19 and drawn. `ClickPinPress 16` switched the main
image to the 4th picture. "stay 14" through a real overlay (message code 6)
opened the existing 13–19 pin. The look toggled to minimal and rendered. The
scope check is true on the page.

### In other apps (built 2026-10-19, same day)

"stay N" works in any app click hints can number. The pin window is the same;
line 5 becomes "Which windows" (any window of the app, or only windows with
this exact title).

- **Identity = a recorded-macro step.** An app pin carries `sel`
  {type, name, aid}, an `anchor` ancestor when the name isn't unique, and
  `rel` (window-relative spot). It is re-found with the recorded-macro
  runner's own tested rules (`RecMacro_Candidates` / `RecMacro_Dedup` /
  `RecMacro_Pick`). So an ambiguous pin draws no badge and clicks nothing,
  rather than guessing. Shared resolver: `Helpers\ClickPinsUia.ahk`, with pure
  tests in `test_click_pins_uia.ahk`.
- **All like this** = the same type under the same direct parent (Notepad:
  File/Edit/View under the menu bar), numbered in reading order.
- **The host** `Scripts\ClickPinsHost.ahk` is its own process. It ticks every
  600 ms and costs one WinGetProcessName unless a pinned app is in front.
  Then it UIA-scans that window every 1.5 s and draws click-through badge
  windows, which hide while make click is up. Its hidden window is titled
  `ClickPinsLive` while pins resolve (otherwise `ClickPinsIdle`). Caster
  scopes the bare number on that title and posts WM_APP+2 (wParam = number)
  straight to it. The click is `_RmClick`, the runner's guarded click.
- **Who starts it:** saving an app pin, "sync pins", and the context
  detector's `ClickPinsWake` observer (`AlwaysOn\ContextDetector.ahk`) when an
  app with pins comes to the front. So nothing runs for people with no app pins.
- **Stale-code trap:** the host holds its code from when it started. After
  editing it, stop its AutoHotkey64.exe process; the next wake starts the new
  code.
- make click keeps app pins' numbers too (`_CHTagAppPins` before pass 0).

**Verified live 2026-10-19 in Notepad:** "stay 7" on File (window scope)
opened the pin window, found the family (3) and saved it as 7–9. The host
started on save and drew 7/8/9 beside File/Edit/View. The voice scope check
was true. Posting 8 exactly as Caster does opened the Edit menu. The test pin
was deleted afterwards.

**Trap hit while testing:** a `try WinGetPos(&x, ...)` that throws can leave
the output variables UNSET, even when they were assigned defaults first. A
fake hwnd of 0 in the tests made the next read throw "local variable has not
been assigned". The resolver now reads into scratch variables and copies them
on success.

### Numbering, right and double (2026-10-19, later the same day)

**Lowest free numbers.** A new pin no longer takes the hint number it was
said on. It gets the lowest free block on that site or app
(`ClickPins_LowestFree`, tested): the first pin is 1, a group of 7 is 1–7, and
the next pin is 8. While the number has never been set by hand (`n_auto`),
turning "all like this" on or off re-picks the lowest block that fits the new
size. Typing a number on line 3 fixes it.

**"right 13".** Right-clicks the pinned element, then opens click hints on
the menu that appeared (`ClickHintsPopup` → host `start ... popups=1` →
`CH_POPUPS`). Every rescan numbers the app's open popups first, if any:
untitled same-process windows, or ones owned by the window (`_CHPopupWindows`
/ `_CHPopupRescan`, clipped to the whole screen because a menu can hang
outside its window). Once the menu closes the overlay carries on as normal
make click over the window. Backspace or Escape ends it, as always.

**"double 13".** Double-clicks.

**How Chrome does right and double.** A synthetic event never opens
Chrome's own context menu, so these use the REAL mouse. The content op
`pins.rect` scrolls the element into view and reports its rect plus viewport
facts. `ChromeBridge_ViewportOrigin` (moved into ChromeBridge.ahk from
ClickHintsDom so click hints and pins share one copy) turns that into a
screen point. The `pointer` op checks the aim and corrects it once, the same
way click hints aim. A plain left press is still the DOM click, so the
pointer doesn't move.

**Apps:** the host already took an action code in lParam; a right-click now
also runs `MAINFUN ClickHintsPopup`.

**`RecMacroFind.ahk` split.** The recorded-macro runner's re-finding rules
(candidates, dedup, anchor, pick, scan, ancestors, click) moved into
`Helpers\RecMacroFind.ahk`. The runner `#Include`s it, MAINFUNCTIONS lists it
explicitly (the closure checker doesn't follow a helper's own includes), and
both click-pin hosts include only it. Without the split, the hosts dragged in
the runner's Chrome / key-action / capture references, and `ahk_warnings`
flagged them.

**Verified live 2026-10-19:**
- Chrome, Amazon thumbnails renumbered 1–7: "double 3" landed with the aim
  verified by the page.
- "right 3" opened Chrome's own context menu, and the hints numbered it 1–16.
- Picking 11 (Bitwarden) opened its submenu, numbered 17–23, while the first
  menu kept its numbers. Backspace ended the overlay.
- Notepad (a temporary pin on its tab): "right 1" opened the WinUI tab menu,
  numbered 1–2, and double-clicking worked. The test pin was removed and the
  host went idle.

**Bug found live:** `_CHPopupRescan` filtered before loading the profile, so
`CH_PROFILE["types"]` was empty and the host died ("Item has no value" at
_CHFilter). The profile now loads before the scan loop.

---

## v1 (2026-07-27) — original design, kept for history

Say **"make click"** (or hit a hotkey). Every clickable thing on screen gets a number
badge. Type the number — no Enter — and it clicks, then re-scans and re-labels. Loops
until Escape. Works in every application, not just the browser.

This is [[GRAB_TEXT_ENGINE]] with the unit source swapped from OCR words to UI
Automation elements, and the overlay unbound from a single window.

---

## Why build it rather than install something

Nothing off the shelf does all four of *global + element-accurate + numbered + loops*.

| Tool | Verdict |
|---|---|
| **Windows 11 Voice Access "show numbers"** | Literally this feature, and genuinely global. But voice-only, no scripting API of any kind, overlay dismisses after each action, and it would sit alongside Dragon competing for the same utterances. |
| **Fluent Search** (Screen Search, Ctrl+M) | Strongest shipped product. UIA engine *plus* a computer-vision fallback engine, whole-screen or focused-window. But letters not numbers, closed source, loop mode unconfirmed, takes no voice command. |
| **hunt-and-peck** | The canonical open-source prior art (C#). Letters, active window only, Invoke-pattern elements only. Its own README calls it sporadically maintained. |
| **win-vind** EasyClick | Actively maintained, scans Win32 + UWP, has an async cache claimed 30× faster. Letters, vim-modal, heavier than we want. |
| **Talon + Rango** | Excellent, browser-only. Talon's Windows accessibility API is experimental and is a scripting surface, not a hint overlay. |
| **warpd** | Not Windows, and its hints are a blind grid rather than element-derived. |

**What changed recently and makes this viable now:** Chrome 138+ ships **native UI
Automation enabled by default on Windows**, replacing the lossy, slow MSAA→UIA emulation
bridge. Electron apps inherit it as they rebase. UIA on Chromium now returns a rich,
fast tree.

### Vimium stays

Not being replaced. The two systems are independently usable and overlap in Chrome by
design.

⚠️ **A correction worth preserving, because it was a reasoning error, not a typo.** An
earlier draft argued the global system was needed in Chrome because a Google Voice dump
had 981 UIA elements but "exactly 6 of type `link`", concluding that was why Vimium finds
so little there. **That conclusion is wrong and the premise doesn't support it.** Vimium
hints the DOM — `a`, `button`, `[role=button]`, `[onclick]`, `[tabindex]` and friends —
so a UIA *control-type* histogram says nothing about its coverage. Jamie confirmed Vimium
in fact works well in Google Voice. Never infer one tool's coverage from another tool's
tree.

What the global system genuinely adds in Chrome is **browser chrome** — tab strip,
toolbar, extension icons, menus — which a page-scoped extension can never reach, and
**every non-browser application**.

**Planned diagnostic:** when a page hints poorly here, Jamie will send a dump captured
with Vimium's hints visible, so we can diff what Vimium labelled against what our filter
kept and find the actual cause. That comparison — not a control-type histogram — is the
right instrument.

---

## Locked decisions

- **Labels are prefix-free, minimum two digits, and never need Enter.** Typing completes
  the moment the label is unambiguous. If `13` exists then `1` cannot be a label.
- **Leading digit 1–9 → 90 two-digit labels (10–99).** `0` is never a label prefix, so it
  is free as a control key.
- **Three-digit badges are permitted** when a window genuinely needs them, but the
  filtering should keep that uncommon. (See the measurement below — this needs the scope
  ladder to actually hold.)
- **Reading order**, banded into rows top-to-bottom then left-to-right within a row.
- **Scope: foreground window + its popups by default**, with a key to widen.
- **Visible action legend** on the overlay. No hidden compound codes.
- **Auto-suspend when a click lands in a text field**, so digits reach the field.
- **Global digit and Escape capture is accepted**, guarded by a dead-man timer.
- **Voice selection**: saying "thirteen" while hints are up clicks 13.
- **Gaze ordering is deferred.** The Tobii is disabled most of the time. Revisit later —
  numbering by distance from gaze would make the intended target almost always single-
  digit, and the hardware is already wired up via `WarpToGaze`.
- **OCR / computer-vision fallback is parked**, to be decided when we hit an app where
  UIA returns nothing.
- **`docs/gui-conventions.md` numpad rules do NOT bind this system.** Explicit exemption
  — this is a bare click-through canvas, not one of the seven blessed GUI shapes, and its
  no-Enter prefix-free input is deliberately different from the `N`/`N.M`/Enter
  convention. Do not "fix" it to match.

---

## What shipped (2026-07-27)

| Piece | File |
|---|---|
| Engine — scan, filter, order, label, overlay, modal loop | `Helpers\ClickHints\ClickHintsEngine.ahk` |
| Own-process host | `Scripts\ClickHintsHost.ahk` |
| Thin launchers (in MAINFUNCTIONS) | `Helpers\ClickHintsLaunch.ahk` |
| Per-app filter profiles | `INIDATA\click_hints_profiles.json` |
| Voice — start the overlay | `caster\rules\click_hints_commands.py` |
| Voice — act while it's up (overlay-scoped) | `caster\rules\click_hints_active_commands.py` |
| Filter tuning harness (Python, reads `uia.txt` dumps) | `Diagnostics\hint_filter_proto.py` |

**Voice:** "make click" (content) · "make click window" (+ app chrome) · "make click all"
(every window). While it's up: say a number to click it, "right 13" / "double 13" /
"hover 13" / "middle 13", "click wider", "click again", "click stop".

**Keyboard:** type the label, no Enter. `00` widen · `01` right · `02` double ·
`03` hover · `04` middle · `05` rescan · Escape quit.

### Two label namespaces: numbers AND letters

Every hint carries both, rendered as **`15.AG`** — one box, monospace, letters uppercase.
Either activates it; the first character typed decides which namespace is in play, and
since digits and letters can never collide the two sets stay independent. `label_mode`
drops either (`"number"` gives narrower badges and is what voice uses).

Both are shaped around Jamie's **split keyboard** so a whole label is typed one-handed:

- **Digits are hand-grouped**, 1-5 left and 6-0 right. All 25 left pairs are issued
  first, then the 20 right pairs. Past 45, labels grow into same-hand **triples** rather
  than reaching for a two-handed mixed pair — three keystrokes on one hand beats two
  across both. Each expansion costs one label and yields five, net +4, and runs from the
  end of the ordering so the extra keystroke lands on the least-likely targets. **Nothing
  two-handed appears until 225 hints.** Verified offline: prefix-free, no duplicates, zero
  two-handed labels at n = 12/45/80/130/225.
- **Letters come from the left half only** — `asdfg` / `qwert` / `zxcvb`, home row first.
  15 × 15 = 225 two-character labels, all the same length so the set is prefix-free for
  free. Note the consequence: on a dense page numbers go to three digits while letters
  stay at two, so **letters become the faster path exactly when it matters**.

⚠️ **The badge font must stay monospace.** The badge is four controls at fixed
character-cell widths; with a proportional font the `.` is narrower than a digit and the
reserved cell leaves visible slack, rendering `15. AG`. Consolas advance is 0.5498em, so
the cell is `fs * 0.733`.

**Backspace is a single escalating undo** (Jamie's request, 2026-07-27): with digits in
the buffer it removes one and the dimmed badges come straight back; **on an empty buffer
it quits the whole overlay.**

**Partial input narrows the field, Vimium-style.** Type `2` and the `2x` labels stay gold
while everything else recedes to grey; *within* each surviving badge the digits already
typed are dimmed and the digits still to come stay solid. Backspace restores the field.

Each badge is therefore **two adjacent Text controls** sharing one background — consumed
prefix, then remainder.

> ⚠️ **Flicker took three separate fixes.** Jamie caught each one.
> 1. **Don't rebuild the Gui.** The first attempt destroyed and recreated the badge
>    window per keystroke — every badge flashes, reading as "the whole overlay
>    refreshed" rather than "the field narrowed".
> 2. **Don't invalidate the whole window either.** The second attempt mutated controls
>    in place but finished with one `WinRedraw(overlay.Hwnd)`, which repaints *every*
>    badge including unchanged ones — still a visible full-screen flash. Invalidate only
>    the controls that changed, via `GuiCtrl.Redraw()`.
> 3. **Double-buffer the window** — `WS_EX_COMPOSITED` (`E0x02000000`) alongside the
>    existing layered/transparent bits, so a repaint composites off-screen instead of
>    clear-then-draw. Verified this does not break rendering on a layered TransColor
>    window (it can conflict on some Windows versions — it doesn't here).
>
> Geometry is also only `Move()`d when the prefix split point actually shifts, so a
> badge that merely changes colour never triggers a window-region recalculation.

**The pop-in flash is a separate bug and took three goes.** Showing the window and calling
`WinSetTransColor` afterwards paints one fully opaque near-black frame — the "whole screen
flickers as the overlay appears" report. Fixes, in order of how much they mattered:

1. Both windows are built with `Hide`, styled, and only then shown via
   `ShowWindow(hwnd, SW_SHOWNA)`. This alone left it flashing ~60% of the time.
2. `WinSetTransColor` now runs **immediately after `Gui()`**, before a single control is
   added, so the background is keyed out before the window can ever paint.
3. **`WS_EX_COMPOSITED` was removed.** It had been added for double-buffering, but on a
   window this large it forces a DWM recomposite when the window appears — itself a
   full-screen flash. Per-control invalidation is what actually fixes keystroke flicker;
   the composited style was only ever belt-and-braces. *If per-keystroke flicker returns,
   this is the thing to put back — but shrink the window first.*
4. **The overlay is no longer full-screen.** It used to span the whole 3640×1920 virtual
   desktop. Badges only ever cover the target window, so pass 1 places them in screen
   coords and takes the union, and the window is sized to that bounding box. Far less
   surface to bring up.

**The status bar is click-through and translucent.** It sits over the bottom-left of the
screen where there are usually real hints; as a solid window it swallowed those clicks and
blocked the view. It now carries the same `WS_EX_TRANSPARENT` as the badge layer plus
`WinSetTransparent(170)`, so hints underneath it are both visible and clickable.

A digit that can't lead anywhere is rejected rather than poisoning the buffer.

**Badge placement is size-dependent** (`badge_position`, default `auto`). Centre on
targets at least `badge_center_min_w` × `badge_center_min_h` (200 × 56), corner on
anything smaller.

Neither alone works. Corner placement makes big targets ambiguous — on a Plex result card
the card's badge and the poster's badge end up squashed together in the same corner with
no way to tell which is which. But centring a badge on a one-word link buries it in the
text. The split costs two integer comparisons per hint, so it is free. `"center"` /
`"corner"` force one everywhere.

### Click history — `%TEMP%\click_hints_history.jsonl`

One JSON line per click: timestamp, action, label, control type, Name, AutomationId,
rect, click point, focusable flag, scope, exe, window title.

Immediate purpose is debugging — when a click lands somewhere unexpected, this says
exactly which element the filter handed over without needing to reproduce it live.

**Planned: fold this into the macro recorder** (Jamie, 2026-07-27 — explicitly *not* now).
A recorded click-hints session becomes a list of concrete, re-findable UIA selectors
rather than bare coordinates, which is the programmatic form a macro actually wants. The
record shape is deliberately kept close to `make record`'s recordings
(`INIDATA\UIARecordings\*.json`, see `references/uia-clicking.md`) so the two can merge,
and the intended UI is the same shape as `open dumps` — a Miller listing recent sessions
with actions on each. The longer-term goal is that acting through the recorder captures
as many elements as possible automatically, so macros can be written programmatically
instead of by coordinate.

Voice works by sending the digits as real keystrokes into the host's global hotkeys, so
voice and keyboard share one input path. It must use dragonfly `Key()`, not `Text()` —
`Text()` sends unicode packets that AHK hotkeys never see.

### `probe` — the diagnostics mode you'll actually use

```
AutoHotkey64.exe Scripts\ClickHintsHost.ahk probe <scope> <hwnd>
```

Scans, filters, labels, writes `%TEMP%\click_hints_probe.txt`, exits. **Draws nothing and
arms no hotkeys**, so it's safe to run any time — this is how to tune a profile without
the overlay seizing the keyboard.

### Measured live, after the perf fix

| Window | raw elements | hints (content) | scan |
|---|---:|---:|---:|
| Chrome — YouTube video page | 1274 | 87 (all two-digit) | **141 ms** |
| Chrome — same, window scope | 1274 | 122 (36 three-digit) | 168 ms |
| VS Code | 1418 | 46 | **203 ms** |
| Signal | 149 | 42 | **32 ms** |
| Kindle for PC | **21** | 2 | 16 ms |

Content scope holds under 90 (two-digit) on every real window tried. The live numbers
track the dump-corpus predictions closely.

### ⚠️ The performance trap (cost us 11-14×)

The obvious way to batch UIA property reads is wrong:

```ahk
els := root.FindElements({}, 5, 0, 0, cacheRequest)   ; DON'T
```

That overload loops `BuildUpdatedCache` **per element** (`UIA.ahk:3081`) — one COM
round-trip each. Use a single subtree cache instead:

```ahk
els := root.BuildUpdatedCache(cacheRequest).GetCachedChildren(5)   ; DO
```

One cross-process call for the whole tree, then every `.CachedX` read is in-process.
Measured on the 1274-element YouTube tab: **1594 ms → 141 ms**. VS Code 2875 → 203 ms.
This is what made the click-rescan loop viable at all; a two-second rescan after every
click would have killed the interaction.

### The Plex "big box" problem — and why it was a general bug (fixed 2026-07-27)

Jamie's `ListNavClickNth` profiles exist partly because hint-style systems kept surfacing
the little links *inside* a Plex show card and never the card itself, so nothing actually
opened the show. v1 reproduced that exact failure. The tree explains why:

```
[link]   (NO Name, NO AutomationId)  [332,310 883x113]   <- the box you must click
  [heading] "Steven Universe"        [440,333 120x25]
  [button]  "steelflicks"            [440,377 120x24]
  [button]  "Play"                   [1107,343 48x48]
  [button]  "More Actions"           [1155,343 48x48]
  [button]  "Select Steven Universe" [332,310  32x113]
```

The card **is** in the tree, as an unnamed `[link]`. It was being killed twice over:

1. `require_name` dropped it for having neither Name nor AutomationId.
2. `nesting: leaf` would have dropped it anyway for containing other candidates.

Both filters were too blunt, and both fixes are **generic, not Plex special-cases**:

- **`require_name` now spares keyboard-focusable elements.** An unnamed `<a>` box is a
  real target; unnamed *non*-focusable filler still goes.
- **`nesting: "smart"` is the new default** (see the correction under Finding 4). The card
  is focusable so it survives; Play and More Actions are focusable so they survive too —
  correct, because opening the show and playing it are different actions. A Signal chat
  bubble is *not* focusable, so it still yields to its Options button.
- **Nameless survivors borrow a label** from the heading inside them, so probe output and
  logs stay readable.

Regression-checked live: Chrome 58 hints, VS Code 49 — no explosion in count, and 10
nameless-but-focusable elements (the Plex-box class) now survive on a Chrome page.

**Not yet confirmed on Plex itself** — the window wasn't open when this was written. The
check is `probe 1 <plex-hwnd>`: the card should appear as a `[link] FOC` roughly 883×113
carrying the show's name.

### Kindle for PC exposes almost nothing

21 UIA elements total, of which exactly 2 are clickable (the page-turn arrows). Not a
filter failure — there is nothing else in the tree. **This is the concrete case that
decides the parked OCR-fallback question**, and it's an app Jamie uses constantly. The
existing `Lib\OCR.ahk` + the GrabText engine already number text on screen in this exact
window, so the fallback has a working precedent to borrow.

## Measured: what filtering actually yields

A prototype filter was run over **12 real `uia.txt` dumps** (9 Chrome, 2 Signal, 1 VS
Code) captured from ordinary use. Pipeline: control-type allowlist → drop degenerate
rects → drop rects >40% of the window → drop offscreen → dedup exact rects → dedup
near-identical rects (4px) → prefer leaves over wrappers → drop nameless *and* aidless.

| raw elements | hints (all) | hints (page only) | window |
|---:|---:|---:|---|
| 1156 | 171 | 131 | Spotify Search (Chrome) |
| 969 | 103 | 59 | Google Voice (Chrome) |
| 865 | 56 | 56 | Mokwheel product page (Chrome) |
| 676 | 151 | 114 | Spotify album (Chrome) |
| 535 | 99 | 57 | Google search results |
| 247 | 72 | 35 | Bandcamp checkout |
| 242 | 111 | 69 | Steven Universe (streaming) |
| 190 | 101 | 64 | INVINCIBLE (streaming) |
| 150 | 47 | 42 | Signal |
| 134 | 46 | 46 | VS Code |
| 103 | 31 | 26 | Signal |

**Median 100 hints at full scope; 58 with browser chrome excluded.** Over-90 cases fall
from **7 of 12** to **3 of 12**.

### Finding 1 — 90 labels is not enough at full scope

This contradicts the assumption that good filtering alone keeps three-digit badges rare.
It does not. Native apps are fine (Signal 31–47, VS Code 46), but Chromium apps land at
100–171 and those counts are *legitimate* — those pages really do have that many
clickable things. Spot-checked: Spotify Search's 169 hints are 100 buttons, 42 links, 10
check boxes, and the repeated 17×32 elements are per-track row controls. Not filter junk.

### Finding 2 — browser chrome is a flat ~40 hint tax

Every Chrome window spends **~37–44 hints** on the tab strip, toolbar, and extension
icons before the page contributes anything. Two dumps had 19 and 21 elements of exactly
34×34 — the extension button row.

**This is the fix for Finding 1: a scope ladder rather than paging.**

1. Page/content region of the foreground window — median **58**
2. \+ application chrome (tabs, toolbar, menu bar) — median **100**
3. \+ all visible windows / whole desktop

`0` is already reserved as a non-label digit, so **`0` widens one rung and re-scans**.
That preserves "no Enter", keeps two-digit labels in the common case, keeps browser
chrome genuinely reachable (a thing Vimium can never offer), and is discoverable from
the action legend.

### Finding 3 — offscreen rejection is the single biggest lever, and we can't see it yet

On Google Voice the allowlist cut 969→378, then offscreen rejection cut 378→142. That
one stage removes **62%** of surviving candidates — the scrolled-out conversation rows.

The prototype approximates it by testing whether an element's centre falls inside the
window rectangle. The real signal is UIA's **`IsOffscreen`** property, which handles
clipping by any scrolling ancestor, not just the window edge. `Diagnostics/UIADump.ahk`
does not capture it today. **Adding `IsOffscreen` is the highest-value change to the
element source.**

### Finding 4 — wrapper-vs-leaf cannot be resolved geometrically

Two cases with identical geometry and opposite correct answers:

- A Google Voice **chat bubble** (688×45) wrapping a 40×41 "Options" button. The bubble
  is not a target; the button is. → **keep the leaf**.
- A Spotify **album tile** wrapping a play button. The tile opens the album and is the
  thing you want; the play button is secondary. → **keep the wrapper**.

Size ratio does not separate them. What separates them is whether the wrapper is *itself
an independent target*.

> **⚠️ Correction to this doc's original recommendation.** It said to use
> `IsInvokePatternAvailable`. **Measured on live Chrome: nothing reports it — not one
> element on a real page.** Chromium exposes activation through
> `LegacyIAccessible.DoDefaultAction` instead, so an Invoke-based rule silently
> classifies every Chromium element as "not a target". **Use `IsKeyboardFocusable`**,
> which does populate and is arguably the better signal anyway: real interactive targets
> take keyboard focus, decorative containers don't.

This is what the shipped `nesting: "smart"` mode does — drop a wrapper only when it is
**not** keyboard focusable. A focusable wrapper survives alongside its children, because
they genuinely do different things.

⚠️ `Helpers/UIAActions.ahk` carries a hard rule: **never touch a ValuePattern property**
(`el.Value` / `el.Dump()`) — it builds a live `IUIAutomationValuePattern` that crashes in
`__Delete` on edit controls. Any pattern probe added here must be guarded and tested
against edit controls specifically.

### Finding 5 — do not round-trip through the text file

`uia.txt` is a lossy rendering. Element names contain embedded newlines, so records
cannot be split on line boundaries (several dumps have name fragments that parse as
bogus elements). The live system must work on in-process UIA element objects. This also
removes the `AutoHotkey64.exe` spawn + `RunWait`, which dominates the current 1–2 s dump
time.

---

## Per-context filter profiles

Anticipated from the start: Signal will want different rules than VS Code than Spotify.
The prototype's counts already show why — "prefer leaves" is correct in Signal and wrong
in Spotify.

**Built on day one**, as planned, so nothing needs retrofitting: `INIDATA/click_hints_profiles.json`,
keyed by lowercased exe, with a `default` block merged first and the exe block merged over
it. Tunable per app: `types`, `nesting` (leaf/wrapper/both), `min_size`,
`max_window_fraction`, `require_name`, `near_dedup_px`, `row_band_px`, `settle_ms`,
`badge_font_size`, `exclude_name`.

Deliberately a **standalone file rather than a `hints` block inside `INIDATA/Contexts/`**,
which is where ListNav's `selectors` live. Reasoning: this is filter *tuning data*, not a
context *definition*, and keeping it separate means a bad edit can't destabilise the
context/binding/Stream-Deck registry. If it later wants context inheritance (parent-chain
fallback, URL matching), migrating into Contexts is the obvious move.

Seeded with `spotify.exe → nesting: wrapper` and `signal.exe → nesting: leaf` to make the
knob's purpose legible from the file itself.

---

## Reuse map — what already exists

AHK root = `~/Desktop/Important/AutoHotkey/`.

| Need | Already built |
|---|---|
| Click-through numbered badge overlay | `_GTShowOverlay()` / `_GTBar()`, `Helpers/GrabText/GrabTextEngine.ahk` — `+E0x80020` (layered + transparent) with `WinSetTransColor("010101")`, gold `FFE24D` badges |
| Numpad modal capture loop | `GTShowGrab()` + `_GTNumKey` / `_GTNumEnter` / `_GTNumBack` / `_GTNumCancel`, same file — including a 2-minute auto-exit dead-man timer |
| Own-process host skeleton | `Scripts/GrabTextHost.ahk` — `#SingleInstance Force` means relaunching kills the previous overlay for free |
| Launcher that captures the pre-launch hwnd | `_GTHostRun()` in `Helpers/GrabTextLaunch.ahk` (hidden launcher does not steal focus) |
| UIA element enumeration with screen rects | `Diagnostics/UIADump.ahk` — `UIA.ElementFromHandle(hwnd).FindElements({}, 5)` |
| Click a screen point with modifiers / multi-click | `_LN_ClickScreen()` in `Helpers/ListNavFunctions.ahk` |
| Guarded UIA click | `UiaClickElement` / `_UiaClickElementGuarded`, `Helpers/UIAActions.ahk` |
| Voice mode gating | `window_exists(title=...)` in `caster/rules/window_context_helpers.py` — raw `FindWindowW`, cheap enough for a `function_context` predicate |
| Per-context profiles precedent | `INIDATA/Contexts/*.json` + `Helpers/ListNavFunctions.ahk` |

Descolada's **UIA-v2 is already vendored** at `UIA-v2-main/Lib/UIA.ahk` and is already
`#Include`d by `MAINFUNCTIONS.ahk` ahead of the Helpers — a new hint function gets `UIA`
for free with no include work.

Caster's bundled Legion grid is enabled and is the closest prior art already installed,
but it finds hints by **pixel analysis** (PIL `ImageGrab` in a separate tkinter process),
so it hits text and misses icon-only buttons. Not the foundation we want.

---

## How the shipped v1 resolved the open questions

| Question | Decision in v1 |
|---|---|
| Re-scan trigger | Auto-rescan after every click. Settle = poll the cheap foreground identity (hwnd + title) every 60 ms until stable for two consecutive polls (max ~480 ms), then a fixed `settle_ms` tail for in-page rendering. Deliberately avoids walking the tree just to decide whether to walk the tree. Jamie left this to me and expects iteration. |
| Which window after a click | Re-acquires the foreground window every scan; never reuses the launch hwnd. Scope 1/2 also sweep other visible top-level windows of the **same PID**, so menus, popups and dialogs get hinted. |
| Action encoding | `0` opens a control namespace: `00` widen · `01` right · `02` double · `03` hover · `04` middle · `05` rescan. All shown in the legend. Voice uses the **same prefix form** ("right thirteen"), per Jamie's preference, rather than a suffix. |
| Click method | Real mouse click at the rect centre. Jamie confirmed she doesn't use a mouse and doesn't care where the cursor lands, which removes the only argument for `Invoke()` — and `Invoke()` silently no-ops on Chromium/Electron anyway. `GetClickablePoint()` is the obvious refinement if centre-clicks ever hit an occluder. |
| DPI awareness | Host calls `SetThreadDpiAwarenessContext(-4)` (per-monitor v2) before anything else, so AHK's coordinates match UIA's physical pixels. |
| Badge collision | Nudge-down-on-overlap, up to 6 attempts. Badge width scales with label length so three-digit badges don't clip; height and width both derive from `badge_font_size`. |
| Multi-monitor | One layered click-through window spanning the **whole virtual screen**, not per-window client area. |

### Still genuinely open

1. **OCR fallback (Jamie's Q9, parked for discussion).** Kindle for PC is the forcing
   case — 21 UIA elements, 2 clickable. Options: automatic fallback when the hint count
   is below a threshold, or a separate explicit command. `Lib\OCR.ahk` and the GrabText
   engine already do exactly this numbering in that exact window.
2. **Confirm the Plex box fix on Plex.** Logic verified generically; the actual page
   wasn't open. `probe 1 <plex-hwnd>` settles it.
3. **Whether `smart` over-hints anywhere.** Keeping focusable wrappers *and* their
   focusable children is right for Plex cards but adds badges. If some app doubles up
   pointlessly, that app gets `nesting: "leaf"`.
4. **Settle timing.** May prove too eager on slow SPA route changes, or too slow on
   snappy native apps. `settle_ms` is per-app for exactly this reason.
5. **Badge legibility.** `badge_font_size` defaults to 9. Raise it in the `default`
   profile if it reads small.
6. **Redraw cost on dense pages.** The badge layer is rebuilt on every keystroke to do
   the dimming. Imperceptible at ~60 badges; if it lags at 150+, switch to mutating
   control colours in place instead of destroy-and-recreate.
7. **Whether global digit capture is annoying in practice.** The text-field auto-suspend
   and the 3-minute dead-man are the guards; real use decides if they're enough.

### The "it just shuts out for no reason" bug (fixed 2026-07-27)

Clicking certain links quit the entire overlay; others re-scanned fine. The cause was the
text-field auto-suspend: `_CHFocusedIsTextField` counted `document` as a text field, and
**after any browser navigation focus lands on the page's RootWebArea, which reports
LocalizedType `document`.** So every click that changed pages tripped the suspend — which
is exactly why it looked random: only *navigating* clicks did it.

Now only `edit` counts. `document`, `combo box` and `search` were all removed, and there
is a comment in the code saying never to add `document` back.

### The stale-scan bug on SPA navigation (fixed 2026-07-27)

After navigating in Plex, the sidebar and bottom bar were re-hinted correctly but **none
of the new page's elements ever appeared** — the system behaved as though the page had
already refreshed.

`_CHSettle` polls `hwnd + window title` and waits for two stable readings. **A
single-page app changes neither on navigation**, so it reported "settled" on the first
poll, the re-scan hit the old DOM, and the parts of the page that genuinely don't change
(chrome, sidebar) made the result look plausible.

Fixed by making the re-scan *content*-aware rather than window-aware. `_CHSignature`
fingerprints a hint set (count + each element's position and name — positions alone are
insufficient, since an SPA route swap can reuse the exact same layout). `_CHActivate`
captures the signature before clicking and passes it to `CHRescan`.

**Waiting for the signature to merely DIFFER was not enough** — that was the first attempt
and it still failed. A page mid-load re-renders its **chrome first**: Plex's left nav and
bottom bar come back before the content does, so "it differs now" fired while the page was
still loading and the scan froze on a half-built DOM.

The working test is **stability**: re-scan until two consecutive scans are identical *and*
different from the pre-click signature, up to 8 attempts at 120 ms. An `A_Index >= 4`
escape hatch covers a click that legitimately changes nothing (a toggle), which would
otherwise never satisfy "different".

`_CHSettle` was correspondingly gutted to a plain sleep — its hwnd+title poll was
worthless for SPAs and cost up to 480 ms doing it.

### Findings worth not relearning

- **`IsInvokePatternAvailable` is useless on Chromium** — reports for nothing. Use
  `IsKeyboardFocusable`. (Cost: one wrong rule, caught by probing before shipping it.)
- **Never batch UIA reads via `FindElements(..., cacheRequest)`** — per-element COM calls.
  `BuildUpdatedCache(cr).GetCachedChildren(5)` is 11-14× faster.
- **An unnamed element is not automatically filler.** If it takes keyboard focus it is a
  real target — this is the entire Plex-card bug.
- **Don't infer one tool's coverage from another tool's tree** (the Vimium error above).
- **`document` is not a text field.** It's what a browser focuses after every navigation.
- **Don't rebuild an overlay to restyle it.** Mutate controls in place; a rebuild flashes.
  And don't `WinRedraw` the window either — invalidate only the controls that changed.
- **A single-page app changes neither hwnd nor title on navigation.** Any "has the page
  settled?" check based on window identity is worthless there; fingerprint the content.
  And "the content differs now" is *still* not settled — chrome re-renders before content.
  Wait for two consecutive identical scans.
- **Read `ahk_event.log` first.** A `Format()` call left one `{}` unfilled, so every badge
  control's options string ended `Background c202020` with no colour and `Gui.Add` threw
  "Invalid option" — silently, because `OnError` returns -1. Five minutes of theorising
  versus one `grep -a ClickHints ahk_event.log`. This is Jamie's standing rule and it was
  right again.
- **Don't inject synthetic keystrokes to test while Jamie is at the keyboard.** A test
  `SendKeys` landed in her chat box. Use `preview` mode (draws the overlay for N seconds
  and registers NO hotkeys — safe any time), `probe` mode, or the `ClickHints/key` log.
- **In AHK v2, assigning to a name a function didn't declare `global` creates a LOCAL.**
  `CHRescan` populated `CH_ALPHA` without declaring it, so the real map stayed empty:
  letters rendered on every badge but no keystroke ever matched, the buffer reset on each
  press, and it looked like the hotkeys weren't firing at all. The `ClickHints/key` log
  line settled it in one read — `key=a buf=a ns=alpha hit=0` followed by `key=v buf=v`
  (buffer reset, not accumulating) points straight at an empty lookup table.

---

## Testing status

Verified by me: AHK validates, include closure resolves, JSON parses, both Caster rules
compile, `MAINFUN.bat ClickHintsStart` returns `OK` and brings up the host, the overlay
renders correctly at 2 hints and at 122 hints (reading order, three-digit badges,
collision nudging, legend all confirmed by screenshot), and scan timings were measured
live across five applications.

**Not yet verified — needs Jamie at the keyboard:** actually typing a number and having
the right thing get clicked; the click → settle → rescan loop across a real navigation;
Escape and the dead-man timer under real conditions; voice number recognition into the
overlay; and whether the text-field auto-suspend fires when it should.

The `probe` mode is the safe way to iterate on filtering without any of that.
