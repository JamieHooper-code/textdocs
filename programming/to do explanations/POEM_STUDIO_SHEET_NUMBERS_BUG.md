---
tags: [bug, poem-studio, voice, dragon, kit]
---
# Known bug: spoken 1 / 2 / 3 don't work in the Poem Studio caption pop-over

Logged 2026-11-03. Related: [[POEM_PRINT_AND_DESIGN]].

## Likely cause, fix in (2026-11-03, untested by voice)
Jamie spotted it: lone "one" to "six" clicked the **Slides** buttons labelled "1" to "6",
also with a pop-over open (they stay in the page behind it). 7 and up worked. The
Dragon Web Extension clicks a button by its text, so a bare-number button beats
Caster's number rule. Fix: studio.html `btn()` now gives any bare-number button the
name "N slides option" and draws the digit with CSS (`.numlabel`), so the button has
no "2" text. **Rule for every local page: never a button whose whole label is a number.**

## Symptom
On the studio page (`127.0.0.1:8296/studio/<poem>?shape=instagram`), spoken numbers
work. Open a caption pop-over (Preview, Hashtags, Content warning; `Scripts/poems/studio/sheet.js`)
and a lone "one" / "two" / "three" does nothing, or clicks something unexpected.
Two-word numbers ("ninety one") still work inside it.

## What the logs show
- `Scripts/dictation_logger/dictation_log_<date>.jsonl`: the lone numbers come back as
  Dragon's digit words `"1"` / `"2"` with `rule: ""`, meaning no Caster rule took them.
  "ninety 1" in the same pop-over is matched by `local pages rules`.
- `ahk_event.log`: no `Keyboard/RawMonitor digit=` line for them, so Dragon did not
  type them either. Once (15:49:18) the caption saved right after a lone "1", so
  something clicked a control.
- Suspect: the **Dragon Web Extension** (installed in Chrome) taking short words with
  its own click-by-name commands while the pop-over is up.

## Tried (all still in place, none fixed it)
1. Page numbers parked while a pop-over is open (only the pop-over is numbered).
2. Digit words 0-9 added to the Caster number grammar
   (`caster/rules/number_words.py`): fixed "ninety 1", not the lone digit.
3. Pop-over no longer `role="dialog" aria-modal`; drawn numbers kept out of button
   names (`content: attr(data-jump-n) / ""` in studio.html and option_grid.js).

## Next ideas
- Turn the Dragon Web Extension off for 127.0.0.1 (or entirely) and retest. If that
  fixes it, the extension is confirmed.
- Compare with the Layout/Background picker (option_grid.js), which is also a
  full-screen overlay. Do lone numbers work there?
- Check Natlink's own log for which grammar took the lone "2".
