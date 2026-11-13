---
tags: [programming, macro, stream-deck, todo]
created: 2026-11-11
status: design, not built
related: ["[[STREAMDECK_WORKFLOW_OVERHAUL]]", "[[KEY_ACTION_BINDING_FEATURE_QUEUE]]", "[[SYSTEMS_STACK_THIS]]", "[[MILLER_GUI_PERFORMANCE]]"]
---

# Macro wizard on the feed kit

**The ask (Jamie, 2026-11-09):** "The macro wizard is very slow and clunky... making new
Stream Deck pages in general is just a pain. Assigning them to context and everything is a
lot of going really deep into Millers and toggling settings... there has to be a way to make
this more automatic, accessible, and intuitive."

**The idea:** rebuild the wizard as a feed-kit system. One Python file describes it. It shows
up as a Chrome page and as a Miller, with the same rows and the same numbers. Making a whole
new deck page becomes a few picks from inside the wizard.

Lives on the Stream system (item t-72d43369). The Spotify / Music / Ambient pages from the same
item shipped 2026-11-11, along with `streamdeck.py usage` and the PAGE RECIPE in the streamdeck skill.

## 0. DECIDED with Jamie (2026-11-11) — build from this, it overrides the sections below

**Her pain, in her words:** "I often have to activate a function first so that it'll show up at
the top, rather than trusting that there will be a nice Spotify section with ranked commands of
what I often use right there." (Today's order: global recents, then all ~2,800 functions. Where
she pressed the key changes nothing.) And: "I don't recognize any of the icons, and none of them
are stuff I would have picked" — Claude-built deck pages were wiped back for this reason.

**It's a local web PAGE, sections of click buttons**, not a Miller. Sections, top to bottom:
1. **This key** — what it does now / did before.
2. **Here** — what she runs in this place (and its parents), ranked by frequency + recency,
   voice + deck + keys all counted.
3. **Recent** — last ~20, anywhere.
4. **For this app** — functions that belong to the place even if never run (from its scoped
   voice rules / `caster_scope`, the context's `dump.code` helper files, function tags/prefixes).
5. **Everything** — search, last resort.

**Pick several, then place one at a time:** she ticks e.g. six functions. Then the page pops
each one in turn, "Where do you want this button?", and she **taps the physical deck key** — an
empty or a full one; a full one is overwritten. (So the wizard has to listen for the next deck
press while placing, the same press interception the wizard uses today.) One deck restart at
the end, not per button.

**Icons:** show the **icons she already uses** on her other pages first, then a few candidates,
**plus full access to the whole icon search**. Never auto-pick silently. (Still open: whether an
un-iconed key shows its text label by default.)

**Data first:** the dispatch log rotates after ~2 weeks, so "Here" needs a small persistent
per-place running count (rolled up daily from `ahk_event.jsonl` + `INIDATA/context_log`;
`Scripts/StreamDeck/sdlib/usage.py` already does the join). Build that first.

**Already built this session, reuse it:** the Add Stream page (`Scripts/StreamDeck/stream_setup*.py`,
`stream_setup_page.html`, :8303) is the same shape (sections of buttons, kit jump numbers, a live
8×4 deck picture with her real icons via `/icon?ref=`). The wizard page should share its look
and the `deck_grid` / `icon_file` helpers. `streamdeck.py usage <context>` is the ranking's seed.

**Process:** build in a fresh chat with this doc as the brief.

**BUILT 2026-11-11 (deck keys only).** Per-place usage store (`sdlib/usage_store.py` →
`INIDATA/usage_by_place.json`), the page on :8304 (`Scripts/macro_wizard/`), tap-to-place
via `Helpers/MacroWizardPage.ahk` + the dispatcher hook, and one write verb
`streamdeck.py place-keys`. A blank key opens the page by default (setting
`macrowizard.deck_opens`, "old" brings the Miller back; the page also has "Old wizard for
this key"). Tested end to end: two keys placed on Spotify by a simulated tap and a click,
mirrored to Deck B, then blanked again. Not done yet: keyboard / pedal / numpad keys,
arguments picked from history, new pages from the wizard (§3), a daily rollup trigger
that doesn't depend on the page server running. Details in the streamdeck skill.

## 1. Why it is slow and clunky now (measured)

From `ahk_event.log` / `ahk_event.jsonl`, 2026-10-15 to 2026-11-11 (25 wizard sessions):

| Step | Time | Where |
|---|---|---|
| Key press → new process starts | 0.33 s | `_SetMacroWizard_RunMiller` starts GuiHost every time |
| Load the function list | 1.1–1.4 s, every open | `_VceLoadData`: AHK parses two big JSON files (490 KB in 0.65 s, 420 KB in 0.3 s) |
| Draw the Miller | 0.8–1.6 s | `_MwRootNodes` builds 2,801 function rows up front |
| First search | 1.5–2.7 s | the search index is built on the first keystroke |
| Open to bound (pedal/keyboard) | median 24 s, slowest 1 in 10 at 31 s | scroll/search 2,801 rows, then the scope picker (`_VWPickScope`) |
| Deck key: icon grid | 0.5–2 s to show, 12–54 s to choose | `_Smw_RunSdIconPicker`, every time |
| Deck key: save | about 4 s, plus a deck restart | `_Smw_AssignSdSlot` → `streamdeck.py replace-at --restart` |

- Press to window averages 1.7 s, up to 3.8 s. The first search adds another 1.5–2.7 s.
- **About half the sessions end with nothing bound** (25 presses, 13 bindings). The wizard
  opens, she can't find the thing fast enough, and closes it.
- Every deck binding restarts the Stream Deck app. Ten buttons means ten restarts.
- A new page is not part of the wizard at all: `open stream` → profile → context link →
  groups → apply, or ask Claude to run the PAGE RECIPE.
- Arguments (like `ListNavClickNth 3`) are typed in (`_MwEditOneArg`).
- The code is big: `MacroWizardMiller.ahk` (1,330 lines) + `SetMacroWizardFunctions.ahk`
  (2,670 lines), which still holds the old numpad picker (`_SetMacroWizard_RunPicker`,
  looks unused, check before removing) and the icon grid.

## 2. The feed-kit version

### Files

| File | What it is |
|---|---|
| `Scripts/macro_wizard/wizard_library.py` | **The spec.** Every row, pick-list and action, defined once (`feed_spec.py` contract). |
| `Scripts/macro_wizard/wizard_do.py` | **The router.** Owns no file. Takes a short list of allowed verbs and passes each to `bindings.py`, `streamdeck.py` or `context.py`. It is the spec's one `WRITER`. |
| `INIDATA/wizard_press.json` | **The press ticket.** AHK writes it on each wizard press: device, slot, layers, context chain, time. The spec only reads it; its time is part of `stamp()`, so the page redraws when she presses a new key. |
| `INIDATA/wizard_queue.json` | Deck changes waiting to go out together (written by `wizard_do.py queue-*`). |
| `Helpers/MacroWizardMenu.ahk` | Made by `new_feed_system.py MacroWizard`: `OpenMacroWizard` / `…Miller` / `…Page`, via `Gui/MillerFeed.ahk`. |
| Page | Free from `kit/feed_page.js` + `feed_routes.py` (`INIDATA/feed_systems.json`). |

The function list loads **once** in the page server and stays in memory. The Miller loads it in
Python (about 0.3 s instead of 1.1–1.4 s in AHK).

### The levels (every row has a number; nothing is typed)

1. **This key** — what it does now and what it did before. Pick: `remove` · `go back to this` ·
   `change place` · `change mode layers`.
2. **Suggested for this key** — up to 8 things she runs *in this place* with no key yet, with
   counts ("ClaudeDeskPin — said 5 times here"). One pick binds it.
3. **Pick a function** — her recent ones first, then everything by group, plus search.
4. **This place's deck page** — which Stream Deck page this place gets, or make a new one
   (section 3). One row, not a tree to dig through.
5. **Groups on this page** (deck keys only) — the ledger checklist, tick on or off.
6. **Tools** — modes, "why didn't that key work", add a place, make a key-action.

**Binding takes two picks.** Pick a function; the first row under it is "Put it here —
<place>", then "everywhere" and "the parent place". The scope picker becomes a short list,
not a separate window.

**Arguments are picked, not typed.** A function with arguments shows the values she has
actually used (from the dispatch log, e.g. `ListNavClickNth 1…6`).

**Deck keys:**
- The icon is picked automatically (top `icon` match). "Change icon" opens a numbered grid on
  the page itself, not a separate AHK window.
- The change goes into the waiting list. A row at the top says "3 deck changes waiting —
  apply (restarts the deck once)". Apply runs one `batch-add`, one restart, then clears it.
- Key Logic buttons: short / mid / long are three rows under "This key".

**Context link** goes through `wizard_do.py context-link get|set|clear` (straight to
`streamdeck.py context-link`), not `_SdLedContextNode`, so the ledger Miller and its tests
stay untouched.

### Which tool does each write (nothing new writes a file of its own)

| Action | Goes to |
|---|---|
| bind / remove / go back / move | `bindings.py add\|remove\|move` (same as `WriteBinding` today) |
| deck key, page, groups, context link | `streamdeck.py batch-add\|remove-at\|ledger …\|context-link\|new-profile\|history revert` |
| new place | `context.py add` |
| spoken phrase | `voice_wizard.py` (as today) |

## 3. A whole new page from the wizard

Her main pain, so it gets its own path: row 4 → **Make a new page for <place>**.

1. **Which place?** Defaults to where she pressed the key. If that window has no place yet:
   "make a place from this window" (`context.py add`).
2. **Start from?** The parent page; for a website, the clean `chrome`. Two or three numbered choices.
3. **What goes on it?** From `streamdeck.py usage <place> --days 14`, as a ticked list:
   - Things she runs here with **no key yet** come pre-ticked, up to 10.
   - Keys she **already presses** on the inherited page are marked "keeps its key" (muscle
     memory): their group stays on, members she never presses get unticked. (This is the
     lesson from the Spotify page, which first shipped without the ListNav keys she used.)
   - A number ticks or unticks. "Add a function" opens the normal list.
4. **Look at it.** The page draws the 8×4 grid with the new keys. The mic key (0,3) is locked.
   "Move this key" offers free spots by number.
5. **Approve.** `wizard_do.py build-page` runs `new-profile "<Name>" --parent <node>
   --context <token>`, then `batch-add`, then one restart. If `batch-add` fails it offers
   `delete-profile` right there.
6. **Undo.** "Undo this page" runs `streamdeck.py history revert <sha>`: it's all one commit.

Needs one new **read-only** verb: `streamdeck.py page-draft <place> --json`. It takes the usage
numbers, removes what is already on a key, adds the inherited keys she presses, and returns a
suggested grid using the ledger's `FILL_ORDER`. The rules live in `sdlib/`, so Claude's PAGE
RECIPE and the wizard draw the same grid.

## 4. What stays in AHK, what moves to Python

**Stays in AHK** (it has to catch real key presses):
- `SetMacroWizard` on/off, device filters, hold-Esc, the flashing reminder.
- Catching the press on each device and `SetMacroWizard_HandleIntercept`.
- **New:** the launcher writes `wizard_press.json`, then opens the wizard; if the page is
  already open it only brings it to the front, and the page redraws for the new key.
- The captured layers (`_Bind_CaptureLayersForSave`) ride in the ticket.

**Moves to Python** (the spec), then deleted from AHK: everything in `MacroWizardMiller.ahk`
(rows, ranking `_MwRankCurrentRows`, scope choices, Key Logic faces, `_MwParseArgDocs`,
`_MwPropagate*`), and from `SetMacroWizardFunctions.ahk` the deck assign / go back / remove
helpers, the icon grid and the old numpad picker.

## 5. Build order

Each step works on its own. The old wizard stays reachable ("old wizard") until step 8.

| # | Step | Size |
|---|---|---|
| 1 | Scaffold `MacroWizard` (`new_feed_system.py`, store → `wizard_do.py`). Page only: "This key" + "Pick a function" from a hand-written ticket. Reads only. | S |
| 2 | `wizard_do.py` bind / remove / go back for keyboard, pedal, numpad, remote. Scope rows under each function. Tests against `bindings.py check`. | M |
| 3 | Press ticket + launcher change in the always-on scripts. Setting `macro_wizard.open_with` (old / page / miller), default **old**. | M |
| 4 | Deck keys: waiting list, one-restart apply, icon grid on the page, Key Logic faces. | M–L |
| 5 | "Suggested for this key" row. | S |
| 6 | `streamdeck.py page-draft` + the new-page path (section 3). | L |
| 7 | The Miller from the same spec (`MillerFeed`). Arguments from history. | M |
| 8 | Default to the page. After two weeks with no "old wizard" use, delete the old code. | S |
| 9 | Groups row + Tools row. | S |

### Risks

- **Always-on scripts** (Footpedals, Q0MaxNumpad, Parent) carry the launcher: reload them right
  after, keep the change to a few lines, test each device by hand. A bad include breaks pedals.
- **`MAINFUN.bat RunGuiTests sd_ledger_leaf`** — the wizard root mounts `_SdLedWizardNode` and
  `_SdLedContextNode` today. Any step that touches `StreamDeckLedgerMenu.ahk` (unmounting at
  step 8) must run it (~90 s). A builder error shows as an empty level with nothing logged.
- **Page server** holds the spec in memory; restart it after spec edits.
- **Deck restart** happens while she is using it: apply only when she picks it.
- **Wizard mode catches pedals and numpad.** While the page is open those presses pick the
  *next key to set*, not page rows; page rows are picked by voice number or deck keys (question 2).
- **Usage covers about 2 weeks** (the log rotates), so suggestions for rare places are thin.

## 6. Open questions for Jamie

1. When you press a key in wizard mode, should the **page** or the **Miller** open by default?
2. Should one page stay open and **follow each key you press**, or close after each binding?
3. Should a new binding default to "this place only" in one pick, asking about scope only when you want something else?
4. When should waiting deck changes go out: when you pick "apply", when you close the page, or after a quiet moment?
5. Icons: take the best match automatically and change it later, or always ask?
6. Suggestions: how far back (14 or 30 days), and how many uses before something is suggested (5?)?
7. A new page goes on both decks (A and the B twin) by default. Any exceptions?
8. Which monitor should the wizard page open on?
9. How long should "old wizard" stay before it is deleted?
10. Anything in Tools you never use and want gone?
