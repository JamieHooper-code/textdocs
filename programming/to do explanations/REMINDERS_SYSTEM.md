---
tags: [programming, design-doc, reminders, journal, people, ifs, local-viewer, ahk, caster]
created: 2026-09-16
updated: 2026-09-17
status: v1 built 2026-09-17 — reading-room panel and new tab panel still to come
related: ["[[JOURNAL_SOURCES]]", "[[JOURNAL_SYSTEM]]", "[[PEOPLE_SYSTEM]]", "[[READING_ROOM_BOOKS]]", "[[NEW_TAB_PAGE]]", "[[QUOTES_SYSTEM]]", "[[SETTINGS_SYSTEM]]"]
---

# Reminders — and IFS parts as people

Jamie says **"remind this"** and something she wants to come back to is kept —
most of all *"check back in with this part"*, written while journalling about
her. Reminders live in **sections** (Programming, IFS, Moon, and any she makes)
on a plain page of text with buttons, which opens beside the daily IFS practice
on the touchscreen. Each one knows where it came from, which parts it is about,
how old it is and every time she checked in.

Designed 2026-09-16, v1 built 2026-09-17.

---

## Decisions

| Question | Decision |
|---|---|
| New system, or a list/stack variant? | **Its own small system.** Lists are markdown checkboxes with no dates; the projects store (where stacks live) has no check-in history. |
| Sections | **Data, and she makes them.** Seeded Programming / IFS / Moon; "Make section" on the page adds more. Rename, reorder, remove-when-empty. |
| Due dates / "late" | **None.** A reminder is "remember to check in with her", never "3 days late". `due` exists on the record for dated reminders later; nothing sets it and nothing shows lateness. |
| Checking in | **Two buttons.** *Check in* just stamps it — no note. *Check in with journaling* opens the writing box, and saving the entry is the check-in. |
| Voice | **`remind this`**, smart the way `journal this` is. Plus `open reminders` / `open reminders <section>`. |
| Viewer | **A Chrome page, not a Miller** — a list of text with buttons, not something to arrow through. |
| Reading room | A panel **later**, copied in once the standalone page has settled. |
| New tab page | Later, and **IFS reminders stay off it**. |
| Parts | **Records in the person store** (`roles: ["part"]`) with a **tagline** — most are Greek figures: *Artemis · male gaze part*, *Nate · inner child*. |
| Part auto-linking | **Like any person.** A mis-link now and then is not a big deal (same as the Wild case in [[PEOPLE_SYSTEM]]). |
| The word `reminders` | The quote store's group behind "Quotes to Remember" was **renamed `remember`** (42 quotes), so the word has one meaning. |
| Storage | `E:\Media\catalog\reminder.json`, next to the journal, git-tracked like it. |
| Clipboard fallback | **None.** A stale clipboard silently becoming an IFS reminder is the failure to avoid. |

---

## Files

```
Scripts\reminders\reminders.py          engine + CLI — the ONLY writer of reminder.json
Scripts\reminders\reminders_server.py   the page's server, :8294 (serve | state | stop | restart)
Scripts\reminders\reminders_page.html   the page (read per request; reloads itself on edit)
Helpers\ReminderFunctions.ahk           RemindThis, OpenReminders, ReminderJournal,
                                        ReminderCompanionForPractice, ShowRemindContext
Helpers\JournalCapture.ahk              _JournalBoxOpened / _JournalBoxClosed (the box's side)
Scripts\MediaCatalog\people.py          add_part / find_part / parts / part_label / is_part
Scripts\MediaCatalog\people_sync.py     create_part (THE place a part is made), `parts`, `part-add`
INIDATA\Settings\schema\reminders.json  companion_on_practice, companion_monitor
INIDATA\VoiceChoices\generic_commands.json   the three voice rows (materialized, no reboot)
Scripts\codebase_tools\tests\test_reminders.py
```

---

## The record

```json
{
  "id": "reminder:20260917_check_back_in_with_her",
  "text": "Check back in with her",
  "category": "ifs",
  "created": "2026-09-17T09:52:00-04:00",
  "status": "active",
  "bumped_at": null,
  "due": null,
  "links": {
    "part_ids": ["person:artemis"],
    "journal_id": "journal:2026-09-17-…",
    "quote_id": "", "book_id": "", "url": "",
    "source_kind": "", "source_title": "", "context": "typing_box"
  },
  "pending_journal": null,
  "visits": [{"at": "2026-09-18T08:40:00-04:00", "journal_id": "journal:2026-09-18-…"}],
  "archived_at": null
}
```

Sections are rows in the store's `categories` list:

```json
{"key": "ifs", "label": "IFS", "spoken": ["internal", "ifs", "parts"],
 "journal_types": ["ifs"], "uses_parts": true, "contexts": []}
```

- `spoken` — what `open reminders <word>` answers to. Renaming a section makes
  the new name sayable and drops the old one; the **key** (and URL) never changes.
- `journal_types` — the journal form a journaled check-in files under.
- `uses_parts` — the edit form offers parts; a reminder made with parts linked
  lands here when nothing else decides.
- `contexts` — context tokens that mean "made here, belongs here" (Programming
  is seeded with `code`, `claude_desktop`). Edit via `reminders.py category-edit
  --contexts` for now; not on the page yet.

### Ordering: attention, never lateness

`attention_key`: **bumped** ("Top") first, newest bump first — until she checks
in, which is what the bump asked for. Then **never checked in**, oldest first.
Then everything else, **least recently checked in** first. A part she has not sat
with in a while rises on its own; nothing is ever flagged.

On screen: *Artemis · male gaze part · started 7 days ago · checked in 3 times (2
in the journal), last yesterday*.

---

## `remind this`

Resolved in `reminders.py resolve` (one spawn), in this order:

1. **Text** — the selection (`GetSelectedText`). None → a one-line box titled
   with the section. No clipboard fallback. (VS Code copies the whole line when
   nothing is selected.)
2. **Inside a journal box** (`typing_box` in front AND the note's pid matches the
   foreground window's process): the box's forms decide the section (IFS entry →
   IFS). A saved entry being edited links `journal_id` and any **parts** linked to
   it; an unsaved one stores the box's **token** (see below).
3. **Anywhere "journal this" knows the source** (reading room, Kindle, video) —
   the source links, and the forms that source implies (an IFS exercise → IFS).
4. **Parts linked** → the first section with `uses_parts`.
5. **Context chain** → a section whose `contexts` has a token.
6. Nothing → a numpad pick-list of sections; typing a new name makes it.

Tooltip: *Reminder added · IFS · Artemis · male gaze part*.

**Diagnostic:** `MAINFUN.bat ShowRemindContext` from any window shows what
`remind this` would decide there, without saving.

### The unsaved journal box

An entry being written has no id, and the box runs in the GuiHost process. So
`AddJournalEntry` / `EditJournalEntry` write `%TEMP%\journal_open_box.json`
(token, pid, forms, source, entry id) when the box opens. `remind this` saves the
reminder at once with `pending_journal: <token>` and drops a flag file; when the
box closes, `_JournalBoxClosed` sees the flag and runs `attach-journal --token
--entry <new id>` (empty on cancel — the reminder stays, only the token goes).
The flag means a normal save with no reminder in it costs nothing.

---

## The page

`http://127.0.0.1:8294/` (all) · `/reminders/<section>` (one — the URL is the
voice command; unknown → 404). Local web app on `Scripts\viewers\local_viewer.py`.

- **Each reminder:** text, a meta line, then *Check in* · *Check in with
  journaling* · (*Entry*) · *Top* · *Edit* · *Archive*. *Check in* and *Archive*
  show an **Undo** bar for 10s — it is a touchscreen, stray taps happen.
- **Edit:** text, section, and (when the section is about parts) part chips with
  ×, plus *Add part*: a name that matches a part links it; a new name + tagline
  creates the part in the person store.
- **Section ▸ Section:** rename, about parts yes/no, journal form, move up/down,
  remove (only when empty, two taps).
- **Archived (n):** *Bring back* · *Delete forever* (two taps; only archived
  reminders can be deleted).
- **Entry** opens the journal on the reminder's newest journaled check-in (else
  its origin entry) via `OpenJournalAtEntry` — the id comes from the reminder,
  never from the request.
- Refreshes every 20s and on focus, but never while she is editing or typing.
- Every control is a real `<button>`/`<a>` (link hints) and ≥44px (touch).
- **Writes are POST from the page's own origin** (`local_viewer.origin_allowed`);
  refusals are logged to `ahk_event.log` as `Reminders/server`.
- **Stale-code trap:** the server holds `reminders_server.py` AND `reminders.py`
  in memory. After editing either: `py Scripts\reminders\reminders_server.py
  restart`. The HTML needs no restart.

### Beside the daily practice

`exercise internal` → `OpenExerciseReader` → `ReminderCompanionForPractice`:
the practice's source resolves to a section (Daily IFS Meditation → IFS), and if
that section has active reminders, `OpenRemindersCompanion` opens
`/reminders/ifs` in **its own Chrome window, maximized on the rightmost monitor**,
then gives focus back to the reading room.

- New primitive `OpenChromeWindowOnMonitor(url, titlePart, monitor)` in
  `OpeningAndClosingFunctions.ahk`: reuses a window on THIS desktop whose title
  starts with `titlePart` (navigating it), else opens a new one; never drags her
  to another virtual desktop. `SnapRightmostMain` joined `SnapLeftmostMain`
  (both now `_SnapMaximizedOnMonitor`), with `GetRightmostMonitorIndex`.
- Settings ▸ Reminders: `companion_on_practice` (on), `companion_monitor`
  (rightmost / leftmost).
- A window centred on the touchscreen keeps it awake, which is wanted here.
- Not wired to `meditate IFS` lockouts — a lockout minimises everything first, so
  that would need the lockout profile's `open` hook.

---

## Journal

"Check in with journaling" → `ReminderJournal(id)` → `reminders.py
journal-source` → `AddJournalEntryForSource(src, <section's journal form>)`:

- **One part** → source kind **`part`** (`person_id`), numbered in the PART's
  series: *IFS: Artemis #3:*. A part outlives any one reminder about her — the
  same reason a letter's series is the person. The entry is linked to the part
  (`people`, manual), so her page in the people hub lists it.
- **Otherwise** → source kind **`reminder`** (`reminder_id`), numbered per
  reminder.
- The box header: *Reminder: …* / *Part: Artemis · male gaze part* / *From: …*.
- `reminder_id` rides along on both, and **saving the entry records the visit**
  (`_JournalBoxClosed` → `reminders.py visit --journal-id`). Idempotent per entry.

Journal-side changes: `reminder_id` in `SOURCE_FIELDS` + `--source-reminder-id`
on `add` / `set-source` / `source-next`; `part` and `reminder` in `SEQ_KINDS` and
`SUBJECT_KINDS`; `source_key` checks `reminder_id`; `_JSourceArgs` passes it;
the journal Miller's By-source labels now name Letters / Conversations / Parts /
Reminders instead of raw kinds.

---

## Parts in the person store

A part is `{"roles": ["part"], "tagline": "male gaze part", …}`. What came free:
`journal letter <part>` (a new part is added to `letter_people.json` and the
journal rule's reload marker is bumped), journal links, a people-hub page.

**Made in exactly one place:** `people_sync.create_part` (used by the page and
`people_sync.py part-add`). `people.add_part` never returns a HUMAN with the same
name — a part called Nate gets `person:nate_part`, not the person Nate.

**Kept out of** (done):
- `read <person>` — `clog _recommender_ids` skips parts.
- `clog person-list` — parts left out by default (it feeds the recommender
  menus); `--with-parts` for `JournalPersonInfo`, which resolves letter names.
- `clog rec-rank` — the recommender picker.
- `people_sync.py report` — counted, but never "probably the same person" or
  "probably not a person".
- People hub — a **Parts** branch; "Everyone" leaves them out.

**Not done (fine for now):** the name-fallback writers (mailwatch, GV import,
`add_voice_choice.py`) could in principle attach a phone to a part by name —
unlikely with Greek names. Journal "By person" still lists parts beside people.
`tags:` markup resolves people before tags, so a part aliased like a tag word
would take it. No `merge` guard against part-into-human.

---

## Voice

| Phrase | Does |
|---|---|
| `remind this` | the above |
| `open reminders` | the page, every section |
| `open reminders <section>` | one section — any name or spoken word (`internal` → IFS) |

Generic-store rows (live without a reboot). Don't use "check" for anything here —
it already means three things (the quote store's `checkin` type, the journal's
`check <type>` sessions, the new tab's planned "check N").

---

## Testing

- `py -m pytest Scripts\codebase_tools\tests\test_reminders.py` — sections,
  attention order, idempotent/undoable visits, archive-before-delete, every
  `resolve` path, journal sources, parts vs humans, journal numbering.
- Visual check: a throwaway server on another port with a temp store and fake
  parts (monkeypatch `R.STORE_PATH`, `R.parts_map`, `_psync.create_part`, set
  `app.port`) — never seed fake reminders or parts into the real stores.
- `MAINFUN.bat ShowRemindContext` for the AHK→Python plumbing.

---

## Later

- **Reading-room panel** — the same list inside the reading room. The Origin rule
  means the reading room (:8289) cannot POST to :8294; proxy through
  `read_server.py` or allow its origin.
- **New tab panel** for non-IFS sections, held to that page's rule: *somatic and
  inspiring, never judgmental or managerial*.
- **Dated reminders** using `due` — where the Macros TODO "natural language
  reminders" would land. No date parser exists yet.
- Section `contexts` on the page; page-scoped voice by row number.
- Moon section ↔ the journal's `checkin-moon_*` forms could pair up.
