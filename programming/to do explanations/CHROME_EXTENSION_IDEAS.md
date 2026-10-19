---
tags: [todo, ahk, chrome-extension, dom, design-doc, macros]
related: ["[[GLOBAL_CLICK_HINTS]]", "[[CHROME_BRIDGE_EVENTS]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]", "[[context-and-listnav-profiles]]", "[[JOURNAL_QUESTION_LOOP]]", "[[SPOTIFY_SCRAPING_INTERNALS]]", "[[ANNAS_ARCHIVE_PIPELINE]]"]
status: 1-4 BUILT 2026-10-18 (tab facts/focus/URL events, video control). Generic EVENT LAYER + topics tabs / page.ready / media / download / input.focus BUILT 2026-10-18 ([[CHROME_BRIDGE_EVENTS]]). Rest waiting.
created: 2026-10-18
---

# Chrome extension: what else the AHK Macro Bridge could do

The bridge (`ChromeExtensions/macro-bridge`, built for DOM click hints) makes
any "do X in the page" feature cheap: one op in `content.js` (or `background.js`
for tab-level things) plus one `ChromeBridge_Call` from AHK. The host never
changes. Design and transport are in [[GLOBAL_CLICK_HINTS]], section "Chrome
extension — DOM hints".

Each idea replaces a workaround AHK uses today because it can only see Chrome
from the outside (UIA, keystrokes, the clipboard, sleeps).

## Being built (2026-10-18)

1. **Tab facts.** url / title / audible for the active tab or any tab, read
   from Chrome itself. Replaces the UIA address-bar read and the Ctrl+L
   fallback, which flashes and breaks fullscreen, in `ChromeCurrentUrl` (30
   files). It also replaces selecting the audible tab just to read its URL.
2. **Tab focus by url/title** in any window. Replaces the UIA scan in
   `FocusChromeTab` (14 files). It also covers the old to-do item "Go to a
   specific Chrome tab by name".
3. **URL change events** pushed to the context detector, instead of polling
   the URL. 39 of the 77 contexts match on URL.
4. **Video control without focus.** Play / pause / seek / speed / captions /
   volume on the audible tab's `<video>`. One implementation covers YouTube,
   Plex and any other site.

## Generic event layer (BUILT 2026-10-18)

Jamie wants everything the extension sees to be listenable generically and
programmatically. **Rule: new data from Chrome = a new event TOPIC on the
layer, never a bespoke push.** The layer has one event shape,
`{topic, data, at, tab}`, and the host:
- appends every event to `MacroData\chrome_events\<day>.jsonl` (durable,
  replayable; kept `chromebridge.event_keep_days`, 30);
- keeps the latest event per topic (`GET /state?topic=`, plus
  `state\<topic>.json`, read by `ChromeBridge_Latest`);
- pushes live to subscribers: `ChromeBridge_On(topic, fn)` in AHK (one window
  message + a subscriber file), `/events?since=` long-poll for Python, and
  in-host `consumers\*.py`.

Live topics:
- `tabs` (migrated; the context detector is the worked example);
- `page.ready` (load + SPA);
- `media`;
- `download` (feeds the download manager as source `chrome`);
- `input.focus` (click hints asks it before UIA).

Full design: [[CHROME_BRIDGE_EVENTS]].

**Adding a topic:** `emit('<topic>', data, tab)` in background.js, or
`emitEvent('<topic>', data)` in content.js. Then document its data shape in
[[CHROME_BRIDGE_EVENTS]]. Nothing downstream changes.

## Waiting: fix each one when its area is next touched

5. **Real page-ready waits.** "Ready" = an element exists / the URL settled /
   network idle, reported by the page. Replaces the sleeps and polling loops in
   `WaitForScrapeReady` (5 files: the Spotify scrapers), the Anna's Archive
   click-through, and the click-hints settle. **The signal now exists:** the
   `page.ready` topic (load + SPA, after a 400 ms DOM quiet). What is left is
   the CALLERS waiting on it, `ChromeBridge_On("page.ready")` or a
   `ChromeBridge_Latest` check, with their sleeps kept as the fallback.
6. **Clean text grabs.** Selection, article text, or a chat transcript read
   from the DOM, with no clipboard and no UIA scrape (the UIA chat scrape is
   about 83% sidebar). Candidates: `grab quote` (QuoteSources), `grab chat` /
   `GrabChatTranscript`, journal captures.
7. **ListNav profiles by CSS selector.** `jump N` / `next` on Google Voice,
   Instagram and the rest, from page selectors instead of hand-built UIA
   profiles ([[context-and-listnav-profiles]]).

## Speculative (become event topics when wanted)

- **Selection** (`selection`): what's selected + url + title, for grab quote / journal with no clipboard.

8. **Google Voice in-page watcher.** A MutationObserver on an open GV tab gives
   instant toasts, including group texts that never send an email. The mail
   watcher stays as the backup.
9. **Fill / type into fields by selector,** without focusing the field and
   without the clipboard. Useful for recorded-macro replay (`selectorclick`
   already exists).
10. **Per-tab time log.** Which page and how long, adding page detail to the
    context log.
