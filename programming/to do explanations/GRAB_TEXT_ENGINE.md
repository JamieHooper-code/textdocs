---
tags: [programming, autohotkey, caster, voice, ocr, grab, quotes, design-doc]
created: 2026-07-22
status: v1-built-awaiting-test
related: ["[[QUOTES_SYSTEM]]", "[[COMPLETION_LOG]]", "[[ALWAYS_ON_CONTEXT_DETECTOR]]"]
---

# Grab Text Engine

Generalize the polished Kindle "show grab" flow — number the text on screen,
pick a unit by number, nudge the selection, capture it — so it works in **any
window**, sends to a **destination** (clipboard by default), and grows
**per-context providers** for apps/sites that need special handling.

This is the "global grab" Jamie asked for: *the same commands we have on Kindle,
enabled everywhere.*

---

## The core idea, in one line

**Trigger is global. Target is the foreground window. Capture is uniform
(OCR-for-overlay + drag-for-text). What varies per context is: how to bound the
text, how to read provenance, whether to annotate, and where the grab goes.**

The mistake to avoid is "run the Kindle OCR pipeline everywhere as-is." The
whole reason Kindle uses OCR is that its Qt surface exposes **no accessible
text**. Most other surfaces *do* — but for a **number-and-pick overlay** we need
word **positions**, and OCR is the one universal way to get positioned words on
any surface. So OCR stays the position source; exact text comes from a real
drag-select + Ctrl+C (falling back to the OCR text only if the copy comes back
empty).

---

## Where this sits vs. the grab features that already exist

Three "grab" systems now coexist. They do **different jobs** — this is not a
rewrite of either existing one.

| System | Job | Mechanism | Files |
|---|---|---|---|
| **Kindle grab** | highlight → quote, with book provenance + on-page highlight | OCR overlay + drag + UIA highlight swatch + `parse-kindle-clip` | `Helpers/KindleGrab.ahk`, `Scripts/KindleGrabHost.ahk`, `rules/kindle_commands.py` |
| **Chrome info grab** | pull **structured fields** (phone/email/address) off a page | UIA Name-walk + `Scripts/grabbers/grab.py` | `Helpers/ChromeInfoScraper.ahk`, `Helpers/ChromeGrabMenu.ahk`, `rules/page_grabber_commands.py` |
| **Grab Text Engine** (this) | select **arbitrary running text by number**, send anywhere | OCR overlay + drag capture + destination router | `Helpers/GrabText/GrabTextEngine.ahk`, `Scripts/GrabTextHost.ahk`, `rules/grab_text_commands.py` |

**Same VUI as Kindle, now global.** The number/number-range phrases *moved* from
`kindle_commands.py` into the global rule: `make grab` / `show grab` numbers the
units, then `grab 23` / `grab 23 to 25` grabs by voice. On Kindle these
**delegate straight back** to `KindleShowSentences` / `KindleGrabRange` (rich
quote + highlight), so Kindle is byte-for-byte identical; everywhere else they
run the generic engine. Kindle keeps its OCR-only extras (`grab reset`, `make
preview`, `switch preview`, `grab music`, `grab <phrase>`). Chrome's word-based
`grab data/phone/URL/tab` never collide with the numeric `grab <n>`.

---

## Architecture

```
Voice ("grab show" / "grab quote" / …)   [global rule]
      ↓
GrabText launcher (MAINFUN hot path — captures foreground hwnd, spawns host, NO OCR loads here)
      ↓
Scripts/GrabTextHost.ahk   [own process; #SingleInstance Force; OCR lib loads here]
      ↓
GrabTextEngine.ahk:
   OCR the target window → positioned words
   segment into numbered units (sentence | paragraph)
   overlay: click-through numbered badges
   picker: type N (or N.M range) → preview → arrows nudge → Enter commits
   capture: drag-select the span + Ctrl+C (fallback: joined OCR words)
   route → DESTINATION (clipboard | quote | todo)
```

Same own-process-host pattern as Kindle grab: the launcher on the MAINFUN voice
hot path is thin and never loads the ~100 KB OCR lib; the host is a separate
`#SingleInstance Force` process, so a new grab kills any live overlay for free.

---

## The provider contract (the "per-context code" Jamie wanted)

A **provider** answers five questions about a context. Everything else is shared
engine. v1 ships a single `generic` provider that answers them uniformly; the
seams are already isolated so a real per-app provider overrides only what's
special.

| Seam | Question | `generic` (v1) | Kindle (future fold-in) | Chrome (future) |
|---|---|---|---|---|
| **bodyWords** | which OCR words are "the text"? | all words | drop toolbar + page-turn gutters (7 % / 95 %) | drop browser chrome |
| **capture** | how to get exact text? | drag + Ctrl+C, fallback OCR text | drag + Ctrl+C | drag + Ctrl+C (or DOM range) |
| **provenance** | source metadata? | window title | author / ISBN / Kindle location (`parse-kindle-clip`) | page title + URL |
| **annotate** | highlight the span? | no-op | UIA "`<color> highlight`" swatch | userscript (maybe) |
| **defaultDest** | where does an un-qualified grab go? | clipboard | quote | clipboard |

**v1 realizes providers as clearly-sectioned branches, not a plugin registry.**
That's deliberate: a switch is a robust, readable first cut and each branch *is*
the per-context code. A later pass can promote them to a `Map`-of-func-refs
registry once the contract has proven stable — noted, not needed yet.

---

## Destinations

Chosen at launch (baked into the spawned host as an arg), read at commit:

- **`clipboard`** (default) — `A_Clipboard := text`, tooltip. The 90 % case:
  "grab this off a random site."
- **`quote`** — reuses the Quotes add-form (`_QAdd*` globals +
  `_QSuggestAndShowAdd`, the exact path Kindle grab uses). Provenance fills
  source title from the window title. See [[QUOTES_SYSTEM]].
- **`todo`** — **provisional.** v1 appends to a safe inbox file
  (`INIDATA/grabbed_inbox.txt`), NOT the real Quest Log, so testing can't
  pollute the live todo list. Wiring it to the real todo/projects store is a
  deliberate follow-up (Jamie picks the target).

New destination = one branch in `_GTRouteToDest`. "grab quote" / "grab todo"
are just launch phrases that pre-set the destination; the number-pick/adjust
interaction is identical regardless of sink.

---

## Voice surface (v1)

Global rule `GrabTextRule` (`rules/grab_text_commands.py`, `RuleDetails` with no
`executable` → active everywhere):

| Phrase | Action |
|---|---|
| `make grab` / `show grab` | number the sentences in the front window |
| `make grab block` / `show block` | number the paragraphs |
| `make grab lines` / `show lines` | number every visual line (structured cards, tables, lists, code) |
| `make grab quote` | number sentences; the grab goes to the **quote store** |
| `make grab to do` | number sentences; the grab goes to the **todo inbox** |
| `grab 23` | grab numbered unit 23 (immediately — preview is OFF by default) |
| `grab 23 to 25` | grab the span 23..25 |
| `make preview` | reopen the LAST grab as an adjustable cyan preview; arrows nudge, Enter re-commits |

Destination is chosen by *which* `make grab` you say (clipboard default) and
persisted to `INIDATA/grabtext_dest.txt`, so the following `grab N` routes there.
**On Kindle every one of these delegates to the native Kindle flow** (via
`_GTUseKindle` in `GrabTextLaunch.ahk`), so Kindle stays a highlighted quote —
unchanged. `todo`/explicit non-Kindle-default dests fall through to the engine
even from a Kindle window.

Voice number entry (`grab N`) runs as its own short-lived host process that reads
the cached unit rects from `make grab` (`grabtext_units.tsv`). **Preview defaults
OFF** — `grab N` grabs and commits immediately; if the sentence boundary was a
hair off, `make preview` reopens that last grab (`grabtext_last.txt`) in the cyan
adjust state to nudge (← → end, ↑ ↓ start) and re-commit to the same destination.
The keyboard picker in the overlay still works too. A flag file
`grabtext_preview.txt` holding `1` forces preview ON for every grab if ever
wanted.

**Segmentation modes + the blob guard:** sentence mode breaks on `.!?` plus
layout gaps; paragraph mode makes bigger chunks; **line mode** makes one unit per
visual line. Structured cards (a Google info panel: Address / Phone / Menu /
Hours stacked as short label lines) have no sentence punctuation and no big gaps,
so sentence mode used to merge them into one giant unit — geometry alone cannot
tell a complete-but-long line from a mid-wrap line. Two fixes: line mode (pick
any line deterministically) and a **blob guard** — a sentence unit spanning ≥ 4
visual lines with zero terminal punctuation is split into per-line units. Real
prose is untouched (a multi-line sentence always ends in `.!?`, so it never trips
the guard).

**Whole-word capture:** the drag anchors sit just outside the first/last word
(left of the first, right of the last), not at word centers — a center-to-center
drag clips the end words in half on character-precise surfaces (Notepad++, Chrome,
editors). Kindle tolerates center-drag; the generic engine cannot, so it anchors
on the edges.

---

## What v1 deliberately does NOT do (and why)

- **Does not touch Kindle grab internals.** The global phrases *delegate* to the
  existing Kindle launchers on a Kindle window (`_GTUseKindle`), so `KindleGrab.ahk`
  is unchanged and Kindle behaves exactly as before. Folding Kindle in as a true
  in-engine provider (retiring the duplicated interaction code) needs a live
  Kindle to test against — **do that with Jamie at the keyboard**, not blind.
- **No URL provenance yet.** Getting the Chrome URL means including `LinkManager`
  (heavier include closure) — deferred to keep the host load rock-solid on the
  first blind build. Enhancement is ~2 lines: include it, call
  `ChromeCurrentUrl()` in `_GTProvenance` when foreground is Chrome.
- **No DOM-native web capture.** The universal OCR + drag path already works in
  Chrome. A true DOM/UIA text provider (exact text, no OCR latency) is a future
  provider, not a blocker.
- **No per-context body-word tuning.** `generic` numbers every OCR word
  including toolbars. Fine for v1; per-app `bodyWords` overrides refine it.

---

## Convergence plan (the roadmap Jamie's vision implies)

1. **v1 (built):** generic engine + clipboard/quote/todo, global voice, Kindle
   untouched. ← *you are here; test this.*
2. **Kindle → provider.** Lift `KindleGrab.ahk`'s Kindle-specific seams
   (bodyWords gutters, highlight swatch, `parse-kindle-clip` provenance, quote
   default) into a `kindle` provider branch; repoint `KindleGrabHost` at the
   engine. Behavior-identical refactor, tested live. Retires ~400 lines of
   duplicated interaction code.
3. **Chrome → provider.** URL provenance (include `LinkManager`), `web/<domain>`
   quote grouping, optional DOM-native capture.
4. **Registry promotion.** Once ≥3 providers exist and the contract is stable,
   convert the switch branches to a `Map`-of-func-refs registry keyed off the
   [[ALWAYS_ON_CONTEXT_DETECTOR]] context chain, so adding a provider is one
   registration.

---

## Files

| File | Role |
|---|---|
| `Helpers/GrabText/GrabTextEngine.ahk` | the engine — overlay, picker, adjust, segmentation, capture, destinations, provenance. Loaded only by the host. |
| `Scripts/GrabTextHost.ahk` | own-process host (`#SingleInstance Force`); include closure mirrors the proven `KindleGrabHost` set + the engine. |
| `Helpers/GrabTextLaunch.ahk` | thin launchers (`GrabTextShow` etc.), `#Include`d by `MAINFUNCTIONS.ahk`; capture foreground hwnd + spawn the host. |
| `rules/grab_text_commands.py` | global `GrabTextRule` voice mappings. |
| `INIDATA/grabtext_preview.txt` | preview-mode flag (default ON). |
| `INIDATA/grabbed_inbox.txt` | provisional todo-destination inbox. |

Debug: `grep "GrabText/" ahk_event.log`.
