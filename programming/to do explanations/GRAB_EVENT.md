---
tags: [programming, design-doc, events, instagram, voice, chrome-extension, ocr, local-llm]
created: 2026-10-30
updated: 2026-10-30
status: BUILT 2026-10-30 (posts + image stories; video stories skipped on purpose)
related: ["[[GAINESVILLE_EVENTS]]", "[[GLOBAL_CLICK_HINTS]]", "[[DOCUMENT_TEXT]]", "[[QUOTES_SYSTEM]]", "[[JOURNAL_SOURCES]]"]
---

# "grab event" — turn an Instagram post or story into an event

Jamie (2026-10-30): on Instagram, looking at a post or a story, say **"grab event"**.
If it's a story that links a post, read the post. Take the caption (the right
sidebar); read the date, time and place; read the flyer itself; save the image.

Instagram is where most Gainesville flyers live and the one place the events
system can't scrape (and she doesn't want it to, see [[GAINESVILLE_EVENTS]]). This
is the manual, one-at-a-time way in: she's already looking at the flyer.

## How to use it

| Say (in Chrome) | Does |
|---|---|
| **grab event** | Grabs the post/story, marks it **Keep**, opens its editor |
| **grab event going** | Same, marked **Going** (so it also goes on the published calendar feed) |

The events page opens on the event's **editor** (`http://127.0.0.1:8300/#edit=<id>`)
within a couple of seconds, showing the flyer and a draft. It says *Reading the
flyer…* while Gemma 4 reads it (about a minute on the first read of the day,
~6 s once the model is warm), then fills the form in. Fields the flyer and
caption don't actually say are outlined in yellow with "check this". **Enter
saves** (Ctrl+Enter from the description box). The editor also has Going /
Keep, *Read flyer again*, and Delete; "What it read" shows the caption, the
model's reading of the flyer and Windows OCR. A grabbed event's card on the
page has an **Edit** button to get back to it.

## Decisions (Jamie, 2026-10-30)

- Default mark **Keep**; "grab event going" for Going.
- After the grab, **open the event in the events page** to confirm or change it
  (replaced the planned Question-window confirm).
- A story that reshares a post: read the post **in a new background tab** beside
  the story, so the story she's watching is never navigated away from. The tab is
  left open for her to look at.
- **Video stories: skipped** ("that story is a video; nothing to read").
- Not decided yet: Partiful links in a caption going to the Partiful reader.

## The pieces

```
"grab event [going]"  (chrome_commands.py, Chrome-scoped, next to "grab book")
   ↓
GrabEvent(mark) / GrabEventGoing   Helpers/EventsFunctions.ahk  (runs the script hidden)
   ↓
Scripts/events/grab_event.py grab [--going] --title "<Chrome window>"
 PHASE 1 (~2 s)
   ├─ bridge op ig.read on her tab (content.js)
   │    story with a post link → ext.readtab: post in a background tab, ig.read there
   ├─ images → downloaded → "events" screenshot bucket (screenshots.py add)
   ├─ draft event: source "grabbed", uid = post url (re-grab = same event)
   └─ MAINFUN OpenEventEditor <id>  → page at /#edit=<id>
 PHASE 2 (~6–60 s)
   ├─ Windows OCR on the images (winsdk)
   ├─ local_llm task event_grab: Gemma 4 26b with the images + caption + OCR
   ├─ checks: date/time/end must be WRITTEN somewhere (see below)
   └─ fills the event in, unless she already saved her own edits
```

| File | Role |
|---|---|
| `ChromeExtensions/macro-bridge/content.js` | op **`ig.read`**: post or story → `{kind, author, caption, posted, location, images, alts, video, post_url}` |
| `ChromeExtensions/macro-bridge/background.js` | op **`ext.readtab {url, op, keepOpen}`**: open a url in a background tab beside hers, run a content op there |
| `Scripts/events/grab_event.py` | the two phases; `read` (debug: print what ig.read sees), `redo <id>` (read the flyer again) |
| `Scripts/local_llm/tasks.json` → `event_grab` | Gemma 4 vision prompt; `local_llm.generate(..., images=[b64])` is new |
| `Scripts/events/events_store.py` | `add_manual`, `get_event`, `update_event`, `delete_event`, `known_address` |
| `Scripts/events/events_server.py` | `/api/grab`, `/api/grab/image`, `/api/grab/save`, `/api/grab/delete`, `/api/grab/redo` |
| `Scripts/events/events_page.html` | the editor view (`#edit=<id>`), the Edit button on grabbed cards |
| `INIDATA/events/sources.json` | row `grabbed` (kind `manual`: no reader, never synced, never flagged gone) |
| `Helpers/EventsFunctions.ahk` | `GrabEvent`, `GrabEventGoing`, `OpenEventEditor(id)` |
| `~/MacroData/events/grabbed/` | one sidecar per grab (`<id>.json`: caption, OCR, model answer, image paths, status, edited) + `grab.log` |
| `Scripts/codebase_tools/tests/test_grab_event.py` | the date/time/end checks |

## How Instagram is read (measured on the live site 2026-10-30)

Class names are scrambled, so `ig.read` goes by structure only:

- A post's **first `<time>`** is the caption's own (no link, or "Edited"). Comments'
  times link to `/p/<id>/c/<n>/`; the **post's** time links to `/<handle>/p/<id>/`
  and gives `posted` and the account.
- The **caption** is found by climbing from that first `<time>` to the first box
  holding a span that isn't the handle / "Edited" / "•". A "more" button is
  clicked first. On a direct page load `og:description` is the fallback.
- **Images**: the smallest box around the post's time with a ≥280px image, the
  largest `srcset` entry; a carousel is stepped through with its Next button.
- A **story** is one big `<img>` (or `<video>`), a Pause button (clicked, so it
  doesn't move on while it's read), a `<time>`, and maybe an `a[href*="/p/"]`
  ("View post"). Opening someone's story marks it seen; ig.read only reads the
  one she is already on.
- Instagram's CDN image urls download with a plain GET.

## Doors and show (2026-10-30)

Events have a `doors` column (`HH:MM`, '' if not said); `start` is the show.
Gemma returns both ("DOORS 7PM x WRESTLING 8PM" → doors 19:00, time 20:00); a
doors time is kept only if it's earlier than the show and actually written. A
lone "doors 7pm" is just the start. The editor has Date | Doors | Show | Ends;
a card says "doors 7pm" in its meta line; the published calendar entry starts
at **doors** (when she'd arrive) and its description says "Doors 7pm, show 8pm".
The description box grows to fit all its text.

## Trusting a date (the failure that matters)

The model's answer is checked against what was **read**, not what the model says
it saw: caption + alt text + Windows OCR.

- **Date** trusted when written there (Nov 14 / November 14th / 11/14 / 14 Nov),
  or as a weekday within a week of the post ("this Friday"), or "tonight" /
  "tomorrow" relative to the **post's** date. The model's own `flyer_text` counts
  only if OCR independently found the same day number (stylized months OCR badly:
  "SATURDAY NOV. 14" came out as just "14").
- **Time** trusted when "7pm" / "7:30" / "19:00" appears (flyer text allowed).
- **End time** kept only for a written range ("7-10pm", "until 11:30", "til
  midnight"). Gemma read "DOORS 7PM x WRESTLING 8PM" as 7 to 8 even when told
  not to, so this is enforced in code.

Anything untrusted is listed in the sidecar's `unsure` and outlined in the editor.

## First real grab

fagathering's lube wrestling post (2026-10-28): Gemma read **Lube Wrestling!**,
Sat Nov 14, doors 7pm, Siembra Farm, 2033 SE 23rd Pl, $15–30, with a short
"bring a towel, carpool" description. ~60 s cold, ~6 s warm. It's on the page
as a Keep.

## Later

- Partiful link in a caption → hand it to the Partiful reader instead.
- A Stream Deck key for "grab event".
- If stories with "View post" need a click to reveal the link, ig.read finds
  nothing and falls back to reading the story image (not yet seen on a real story).

## Next: watch a few accounts automatically (agreed 2026-11-02, TABLED)

Jamie approved the idea and parked it for later. When it's picked up:

- **What:** twice a day, open each watched account's profile in a background tab
  of her signed-in Chrome (`ext.readtab`), read the newest ~6 post links, skip
  ones already seen, and run only NEW posts through the same reader as "grab
  event" (ig.read + Gemma flyer reading). No new scraping machinery.
- **Accounts to start with:** Casual Tuesday Ride, Tongue-Tied, Queer the Fest,
  Florida Abolitionist Gathering (`fagathering`), the CMC. Claude looks up the
  exact handles (the `civicmediacenter` profile only showed 2013-2015 posts on
  2026-10-30, so it may not be their live account).
- **Treated like any other listing:** its own source (e.g. `instagram`), scored by
  the normal keywords/ratings, NOT auto-marked Keep. "grab event" stays the
  manual, marked path.
- **Not getting banned:** a few dozen page views a day through a real logged-in
  browser, spaced out with random gaps, only while Chrome is open. Stop
  immediately and tell her (tooltip, plus the source shows FAIL) the moment a
  challenge / "suspicious activity" / login page appears; never retry through it.
  Instagram's terms forbid automated collection, so the risk is not zero; at
  this volume the realistic worst case is a temporary action block.
- **Why it matters less now:** gainesvilleshows.com (fixed 2026-10-30) already
  covers The Bull, The Atlantic, Lightnin' Salvage and Signal, the venues that
  were "Instagram only". The watcher is mainly for groups that only post there.
