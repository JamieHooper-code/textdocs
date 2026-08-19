---
tags: [programming, journal, caster, ahk, tagging, media-system, writing]
created: 2026-07-29
related: ["[[QUOTES_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[COMPLETION_LOG]]", "[[EXPORT_SYSTEM]]"]
---

# Journal System

A tagged store for Jamie's written journal — 98 entries, 2020-09-26 to
2026-07-29, ~85k words, imported from the Google Doc she had been keeping since
2020. Built as the **second parallel text system** after [[QUOTES_SYSTEM]] and,
like it, a thin layer over the media catalog core: same atomic writes, same
locks, same ONE tag vocabulary. Poems are the planned third sibling.

See also [[QUOTES_SYSTEM]] (the system this is modelled on) and [[MEDIA_SYSTEM]]
(the catalog whose infrastructure both reuse).

## The mental model — four axes

- **tags** — WHAT it's about. The shared vocabulary in `INIDATA\media_tags.json`
  with umbrella rollup, opted in via `applies_to: ["journal"]`. This carries all
  thematic search, and because it is the *same* vocabulary the books and quotes
  use, an entry about shame and a book quote about shame find each other.
- **entry_types** — the FORM. Multi-valued, a dedicated field rather than tags so
  it stays deterministic and never pollutes the LLM tag vocabulary. Same design
  as the quote store's `entry_types`.
- **people** — WHO is in it. Links to `person.json` via
  `MediaCatalog/people.py` — the very same store the media catalog's
  "recommended by" dimension uses, so the friend who recommended a book and the
  friend in an entry are one record.
- **segments** — the entry's INTERNAL structure, and the one real departure from
  the quote model. See below.

## Segments and the three depths — why an entry is not one blob

A journal entry is an ordered list of **segments**, each carrying a `role` and a
`depth`:

| role | depth | what it is |
|---|---|---|
| `body` | core | her own journaling |
| `commentary` | core | her follow-up thoughts on the material below |
| `summary` | summary | a condensation of a long chat |
| `quoted` | summary | a verbatim message to or from someone (`speaker`) |
| `prompt` | summary | what she wrote TO an AI (`model`, `url`) |
| `ai` | full | an AI's reply / a full transcript |

`render --depth X` emits every segment at depth X **or above it**, which is what
makes generated documents modular. **Within one entry the segments are then
reordered for reading** — her writing first, summaries collapsed at the bottom,
transcript last — regardless of stored order; storage keeps chronology so
`chat-log` still reads correctly. See [[EXPORT_SYSTEM]] § *Her writing on top*.

Jamie asked for exactly three views:

```
--depth core      the original entry by itself
--depth summary   the entry plus the summary (and her side of any chat)
--depth full      the entry with all the other stuff beneath it
```

**Why this is load-bearing:** she is moving toward saving whole ChatGPT
conversations alongside entries. Without a depth axis, one 40-turn transcript
would drown a year of journaling in any document generated from the store. With
it, the core document stays the thing she actually reads and she burrows down
only when she wants the source.

### Two invariants that protect her writing

1. **`ensure_core` — every entry has something at core depth.** An entry that is
   nothing but a message to ChatGPT plus its reply has no `body`; without this
   it would vanish entirely from `render --depth core`. The promotion prefers
   `prompt` (her words) over anything else. Called from `refresh_derived`, so
   every write path gets it.
2. **When the importer is unsure whose voice a passage is, it assumes hers and
   files it at core.** A misfiled AI paragraph makes the core document verbose;
   a misfiled paragraph of *hers* makes it disappear. Those costs are not
   symmetric, so the tie always breaks toward her.

## Storage

| Thing | Path |
|---|---|
| Journal store | `E:\Media\catalog\journal.json` — `{"items":[...]}` |
| Shared tag vocab | `INIDATA\media_tags.json` (`applies_to: ["journal"]`) |
| New-tag review queue | `INIDATA\journal_tag_review.json` |
| People | `E:\Media\catalog\person.json` (via `MediaCatalog/people.py`) |
| Engine + CLI | `Scripts\journal\journal.py` |
| Importer | `Scripts\journal\journal_import.py` |
| One-time seeds | `Scripts\journal\seed_people.py`, `seed_people_round2.py`, `seed_journal_tags.py`, `fix_people_20260729.py` |
| Viewer (AHK) | `Helpers\JournalMenu.ahk` + `Scripts\JournalViewer.ahk` |
| Capture / write (AHK) | `Helpers\JournalCapture.ahk` |
| Chat transcript parser | `Scripts\journal\chat_parse.py` (testable: `--file <dump> --report`) |
| Inline markup parser | `Scripts\journal\markup.py` |
| Standing AI prompt | the send named `journal thoughts` (`INIDATA\sends.json`) |
| Voice rules | `caster\rules\journal_commands.py` (`JournalRule`) + `journal_markup_commands.py` (`JournalMarkupRule`, `function_context`-scoped to the writing box + Google Docs) |
| LLM tasks | `Scripts\local_llm\tasks.json` → `journal_title`, `journal_tag`, `journal_summary` (`journal_type` exists but is unused) |
| Generated documents | `E:\Media\Journal\` (setting `journal.export_dir`) |

`"journal"` is in `media_catalog.EXTRA_APPLIES_TYPES`, **not** `MEDIA_TYPES`, so
entries never surface in `clog media-query` while still sharing the taxonomy.

## Entry schema

```json
{
  "id": "journal:2026-07-29-aggressive_therapies_for_chronic_pain",
  "type": "journal",
  "date": "2026-07-29",
  "time": "",
  "title": "aggressive therapies for chronic pain",
  "title_source": "manual|llm|",
  "entry_types": ["ai-conversation"],
  "types_source": "auto|manual",     // manual => `retype` leaves it alone
  "importance": 7,                   // 0-10, hers alone; null when unset
  "segments": [
    {"role":"prompt","depth":"core","text":"…","model":"ChatGPT","url":"https://chatgpt.com/share/…"},
    {"role":"ai","depth":"full","text":"…","model":"ChatGPT"}
  ],
  "people": [{"id":"person:cedar","src":"auto"}],
  "tags":   [{"tag":"chronic pain","src":"auto"}],
  "links":  [{"kind":"chatgpt","url":"…"}],
  "source": {"kind":"google-doc","ref":"Journal.txt:0","weekday":""},
  "status": "active",
  "added":  "2026-07-29T…",
  "modified":"2026-07-29T…",
  "words":  488
}
```

`links` and `words` are DERIVED by `refresh_derived` — never hand-edit them.

### `entry_types` vocabulary and why classification is NOT an LLM job

Seeded `classic, ifs, meditation, ai-conversation, message, therapy, creative,
daily-log, dream`; grows via `entry_types_vocab()` which unions seeds + stored +
actually-used, so a type created at runtime is never lost from a picker.
Distribution: **classic 62, message 12, daily-log 10, ifs 5, ai-conversation 4,
creative 4, meditation 3, therapy 2**.

`classify_entry` in `journal.py` is **deterministic**, and that is a considered
decision, not laziness. The local model was tried for this (task `journal_type`,
which still exists) and was materially worse across two prompt revisions: it put
`meditation` on an entry about starting work again, `ifs` on entries containing
no parts work, and *dropped* `message` from an entry built around a quoted
message. Rules can be inspected and measured; a 7B classifier here could not.

The governing insight, from reading the corpus: **a form requires the entry to BE
that kind of document, not to mention it.** "I meditated" appears in 8 entries;
only 2 are records *of* a sit — the rest are one line inside a day's recounting.
So the discriminators are the **title** and the **opening ~500 characters**,
which is where she says what an entry is, rather than any mention anywhere in the
body. (Both Jamie and I initially assumed `meditation: 1` was too low. It was
nearly right. Widening it to a body-wide match would have produced 8 wrong
labels.)

Two types are **structural facts** rather than judgment calls and are derived,
never guessed: `ai-conversation` from the presence of an `ai`/`prompt` segment or
a chat link, `daily-log` from the date preceding `DAILY_LOG_BEFORE`.

`journal.py retype` re-runs the identical logic over the whole store; the
importer calls the same function, so the two cannot drift. They can still report
slightly different counts on the same corpus, which is expected rather than a
bug: 51 titles were LLM-generated *after* import, and title-based rules can only
fire once a title exists.

## PROVENANCE IS A CORRECTNESS RULE (read before touching the scanners)

Both `tags` and `people` are lists of `{value, src}` where `src` is
`auto | manual | import`. **A rescan rewrites only the rows it owns (`auto`) and
never touches a link made by hand.**

This is what makes the retroactive backfill safe, which was an explicit
requirement: *"if I end up adding someone in the future, it will check the old
data and add them and tag them for that as well."* The loop is

```
journal.py people-propose            # recurring capitalised names nobody owns
journal.py person-add <Name> --aliases …
journal.py people-scan --commit      # re-links EVERY entry, backfilling the new person
```

`people-scan` recomputes the auto set from scratch every run, so it is idempotent
and safe forever. Getting this backwards is exactly how a routine `--retag` once
deleted every Anna's Archive tag (see [[ANNAS_ARCHIVE_PIPELINE]]).

### Name matching — three tiers

`_name_patterns` builds the matchers for one person. Names in this corpus range
from "Charli" to "10", so one rule cannot cover them:

1. **`match_patterns` on the record wins outright.** Some names cannot have a
   matcher derived from them at all. A person called **T** would match every
   "T-shirt"; **10** would match every "10 minutes" and "10/10 effort"; **five**
   is both a person and the number, and matching the bare word produced 12 hits
   of which *zero* were her. These get hand-written CONTEXT patterns keyed on how
   they actually appear — inside comma-separated name lists (`, T,`), after
   with/and beside another person (`with 10 and Rigo`), or followed by a verb
   (`five and I are hanging out`, `because five is not really poly`).
2. **`case_sensitive` on the record overrides the length heuristic, in BOTH
   directions.** Jamie writes **wild** and **Mel** in lower case, so the
   ≤4-char default guess missed 9 of Wild's 10 entries; **Ally** is also an
   ordinary word and only counts when capitalised. Neither fact is inferable
   from the name, so it is stored, not guessed.
3. **Otherwise the heuristic**: short (≤4 chars) or all-lowercase names match
   case-sensitively; longer distinctive names match case-insensitively, so
   `charli` and `Charli` both resolve.

**Aliases are not a merge tool for similar names.** The first seed filed *Nata*
as an alias of *Natalia* — they are two different people, and the mistake
silently merged two humans into one record and mislinked 4 entries. Splitting
them and rescanning corrected it (`+44 new, -4 stale`), which is the backfill
guarantee doing its job. When unsure whether two spellings are one person, make
two records: merging later is one command, un-merging costs a manual audit.

**Aliases carry real weight here** — several people appear under more than one
spelling and would otherwise read as separate humans: Charli/Charlie,
Jessie/Jesse, Natalia/Nata, Marrow/mero, Briar/Brandi (she is unsure of her
therapist's name herself, writing "Brandi/Briar(?)").

`people-propose` filters candidates by a **sentence-initial ratio** — a
capitalised word that *always* opens a sentence is "Maybe" or "Honestly", not a
name. That positional test removed far more noise than the stop-list did, and is
why the stop-list only needs to cover names that are also common words.

Deliberately NOT seeded, because a wrong person is worse than a missing one
(a missing one costs one command and is then backfilled): places (Springs,
Gainesville), media mistaken for names (Eldon/Elden Ring, Lord, Mistborn), and
names that are also ordinary words (Ally, Mel, Mary, Bell, Wild, Five). The
corpus also contains people called **T**, **Jam** and **10**, which is a good
reminder that the proposer can only ever be a suggestion.

## Import — two header eras, and the coverage guard

`journal_import.py` reads a plain-text export of the Doc (File → Download →
Plain text). The RTF export also exists but Google splits heading text across RTF
runs, making it unusable.

Two header formats, both of which must be handled:

```
2023-2026   July 29, 2026 -  aggressive therapies for chronic pain
            January 23, 2026 Breakup          <- NO separator before the title
2020-2021   Monday December 21st 2020 5:40 p.m.
```

Each cost a parse pass during development. **Requiring a `-` between the year
and the title silently swallowed 7 entries into their predecessors**; ignoring
the ordinal form dropped the whole 2020-21 era. The parse reported a confident
88, then 91, when the true number was 98 — Jamie caught it.

**The guard against a repeat is coverage, not a regex.** `journal_import.py
verify` asserts that every non-blank line of the source lands inside exactly one
entry. If a header form is ever missed again, its lines show up as orphans and
the count is non-zero; `import` refuses to run with a parse gap unless forced,
and `--expect N` refuses unless exactly N entries parse. Always import with
`--expect`.

```bash
py journal_import.py verify
py journal_import.py import --expect 98 --review        # dry run
py journal_import.py import --expect 98 --commit
```

Idempotent: entries are deduped on date + normalized opening text, so the Doc can
stay the source of truth and be re-imported after edits.

### Segmentation

Two phases, then a merge. Phase 1 gives every unit a provisional role; phase 2
smooths isolated misclassifications; then consecutive same-role units merge back
into one segment.

- **Units are LINES, not paragraphs.** The Doc export is inconsistent about blank
  lines, and splitting on paragraphs welded a share URL, the message above it and
  the AI reply below it into one indivisible block — making the whole
  prompt/AI boundary undetectable.
- **Voice scoring** distinguishes the AI from her: AI condensations in this
  corpus are overwhelmingly second-person ("your instinct", "try asking"), her
  journaling overwhelmingly first-person. Scored per unit, continuously — an
  earlier state machine stopped scoring once her voice resumed and never
  re-detected AI prose later in the same entry.
- **A fully-quoted line carries no evidence** about who is quoting it, so it
  inherits the surrounding voice rather than scoring. `"If I can just find the
  right amount of discipline"` reads first-person but is ChatGPT quoting a
  thought back at her.
- **Transcript dialects**: `You said:` / `ChatGPT said:` (copied share page) and
  `Prompt:` / `Response:` (typed by hand). Both are matched **before** the
  generic `Name:` speaker rule, which otherwise reads "Prompt" and "Response" as
  the names of two people and files a whole exchange as quoted messages.

Round-trip verified: 351 of 356,636 normalized characters differ, and a word-level
diff confirms all of it is label prefixes (`talking to chat gpt`), share URLs
(now structured fields) and `______` separators. No prose was lost.

## Writing into the journal

`Helpers\JournalCapture.ahk` owns everything that WRITES; `JournalMenu.ahk`
browses. Three entry points:

- **`journal <form>`** → `AddJournalEntry(types)`. The SAME box, with the form
  named up front: `journal dream`, `journal classic`, `journal internal`. The
  title becomes `Journal entry (dream) — Sunday, August 16, 2026` (the words
  "journal entry" stay, so the markup rule's scope is unaffected).

  **This is a correctness feature, not a shortcut.** `new_entry` stores an
  explicit type with `types_source: "manual"`, which `enrich` and `retype` both
  refuse to touch — so the entry cannot be silently reclassified by the enrich
  pass that runs seconds after she saves. Title, tags and people are still
  derived automatically; only the FORM is hers. Jamie asked for these commands
  because she is *"really scared of my journal entries getting misclassified and
  forgotten"*, and this is the mechanism that answers it. Verified: a `dream`
  entry survives `enrich` as `dream/manual` while still being auto-titled
  ("flooded library dream") and auto-tagged.

  The spoken vocabulary is a REGISTRY, not a literal list. `entry_types_meta` in
  the store holds extra `spoken` aliases and an optional label; the form's own
  name is always speakable and is deliberately NOT stored (storing it would
  leave a stale alias behind if the form were renamed). Currently 16 phrases
  reach 9 forms — `ifs` answers to `ifs`, `internal`, `parts work` and
  `internal family systems`.

  Managed in the Miller under **Forms**: per form, *Write one now* · *Spoken
  names* · *Entries of this form*, plus **New form…**. Adding a kind of
  journalling costs a few keystrokes and no code, which was the requirement:
  *"I can very easily add the voice command and add the type and everything. It
  will slot into the backend perfectly."*

  Caster reads `~/.claude/context/journal_types_dump.json`, rewritten by
  `journal.py` on every registry edit (`type-add` / `type-meta` / `types-cache`).
  Same contract as sends, same silent failure if skipped. Choice keys are
  sanitised — `ai-conversation` → `ai conversation` — because punctuation in a
  key raises `GrammarError` and kills the whole rule. `journal <journal_type>`
  is declared BELOW the `add journal` / `open journal` literals so those are
  never shadowed.

- **`add journal`** → `AddJournalEntry`. A multiline box (Ctrl+Enter saves), then
  `journal.py enrich` derives the title, form, tags and people in one call and
  the Miller opens so anything wrong is one keystroke away. Nothing is asked of
  her at write time — the whole point is that writing costs only writing.

  The box is titled **`Journal entry — <weekday, month day, year>`**, like a page
  in a paper journal, and that title is also what the markup voice rule scopes
  itself to. It is **`resizable: true`**, which is what makes Win+Left /
  Win+Right work — snapping needs a sizing border, and a fixed-size Gui simply
  ignores those keys.

  **End and PgDn do NOT cancel it.** They are end-of-line and page-down in a
  textarea, and binding them to cancel threw away the whole draft the first time
  she reached for End out of habit. Fixed in the shared template for every
  multiline caller, not just this one — see
  [gui-conventions.md](../../../AutoHotkey/docs/gui-conventions.md)
  § *The multiline exception*.

  **The box's features are GENERIC, not journal-specific.** `draft_key`,
  `line_numbers`, `resizable` and the multiline key fixes all live on
  `_SingleFieldSkipableInputGui` / `GuiPrimitives.ahk`, because the plan is to
  write in these boxes routinely and eventually retire the Dragon dictation pad.
  The journal is just the first caller. Contract + gotchas:
  [gui-conventions.md](../../../AutoHotkey/docs/gui-conventions.md) § *Composing
  prose in a box*.

  - **Nothing typed can vanish** — `draft_key: "journal_entry"` autosaves to
    `INIDATA\Drafts\`; cancel also copies to the clipboard. The draft is cleared
    **only after the entry is really in the store**, so a failed save leaves the
    text recoverable and the next `add journal` offers to restore it.
  - **Line numbers + "go 70"** — a gutter numbering logical lines (a wrapped
    paragraph is one number), and `GuiGoToLine` jumps the caret, cross-process.

  **It runs in its OWN PROCESS via `_RunInGuiHost` (2026-07-29).** MAINFUNCTIONS is
  `#SingleInstance Force`, so the next MAINFUN call — a voice command, a Stream
  Deck button — replaces the process and takes the open box with it, destroying a
  half-written entry. gui-conventions.md exempts "short-lived modal GUIs that
  block and return a value", and this is one; **the exemption is about the length
  of the session, not the template.** She sits here for minutes. It is also no
  longer +AlwaysOnTop: that turned out to be a write-only option on this template
  (see § *Flags go through `_GuiApplyStandardSetup`*).
- **`grab chat`** → `GrabChatToJournal`. Capture the ChatGPT/Claude conversation
  in the front Chrome tab as a NEW entry.
- **`grab chat here`** → `GrabChatToLastEntry`. Append it to the most recent
  entry instead — the "I wrote an entry, then talked to ChatGPT about it" flow,
  where the conversation belongs *underneath* the writing it came from.

## The chat grabber is now GENERIC and TWO-SOURCE — `Helpers\ChatGrab.ahk`

`GrabChatMarkdown(opts)` is the engine, and it knows nothing about journalling — the
journal calls it, and a poem store or a bare "save this chat" gets it free. Voice
`grab conversation` → `GrabConversation()` puts the whole chat on the clipboard and
in `page-grabs\`.

### Neither source is complete on its own

Measured on one real 8-message conversation (2026-07-29):

| | Obsidian Web Clipper | UIA page walk |
|---|---|---|
| turns captured | 7 | 5 |
| her opening question | ✓ | **✗** |
| a **collapsed** message ("Show more") | **✗** | ✓ |
| headings / lists / bold / tables | intact | all flattened |
| useful text | **97%** of the file | 45–67% |

The clipper reads the DOM and has a ChatGPT-aware template, so it is **primary** —
it keeps the markdown, which is the entire point. But ChatGPT collapses long user
messages behind "Show more" and collapsed text is not in the DOM, so the clipper
loses them *and does not know*: its frontmatter claimed "7 messages". The UIA tree
does expose that text, so the scrape is a **cross-check that can contribute turns**,
not a fallback.

Three defences, cheapest first:

1. **`ChatExpandCollapsed()`** — UIA-Invoke every "Show more" before anything reads
   the page, bounded to 6 rounds since expanding one can reveal another. Fixes the
   loss at source, which is what makes one grab enough.
2. **Merge** — `chat_parse.py --merge-with`. Primary wins on text and order;
   turns only the secondary has are spliced in where their neighbours imply, matched
   on punctuation-stripped text (the two sources never agree byte for byte) at
   ≥0.6 similarity, with containment counting as a match. Verified on the real pair:
   7 primary + 5 secondary → **8 turns, 1 recovered**, in correct order.
3. **Scroll passes** — ChatGPT only renders near the viewport, so `_ChatScrapePasses`
   sends `^Home` then walks down with `PgDn`, concatenating each scrape into one
   file. `dedupe_near()` then keeps the LONGEST version of each turn, so a
   half-rendered copy from an early pass loses to the complete one.

The counts and any recovery go in the tooltip on the **success** path too
(`8 turns (4 you / 4 ChatGPT), 1 recovered by cross-check`), so a short capture is
visible without going to look for it.

`_GrabPage_ClipAndWait()` + `_GrabPage_CloseObsidianHolding()` were extracted out of
`GrabTab` so both share one implementation of the clipper handshake.

**`grab chat debug` keeps timestamped dumps** in `page-grabs\chat-debug\` — the UIA
capture, the clipper markdown, and a report, all dated. Overwriting them meant two
captures could never be compared, which is exactly what had to happen to discover
the two sources lose different messages.

Verified end to end through `add --chat`: 8 segments, correct roles and order, and
headings / bullets / bold / tables all present in the store. Depths: core 32 ·
summary 3,442 · full 14,404.

### The dependency this added, and the four hosts it broke

The draft-restore prompt made `_SingleFieldSkipableInputGui` call
`_ConfirmationModalGui`. Four own-process hosts included the former and not the
latter (`GrabTextHost`, `JournalViewer`, `KindleGrabHost`, `SettingsViewer`) — a
latent runtime crash that `validate` cannot see, because AHK resolves an
out-of-scope call dynamically. **`py ~/.claude/helpers/ahk.py deps <host>` is the
check for this**, and it is worth running after adding ANY cross-file call to a
shared template.

### The original UIA-only capture (superseded)

`GrabChatTranscript()` knows nothing about journalling. It returns
`{ok, url, model, text, raw}` so a future poem store, or a bare "save this chat"
command, can call it unchanged — Jamie asked for that explicitly.

**Do NOT use Ctrl+A / Ctrl+C.** The first version did, on my untested
assumption, and it failed every time. On ChatGPT focus normally sits in the
composer textarea, so Ctrl+A selects that (usually empty) field rather than the
conversation; with focus elsewhere it often does nothing at all.

It now reuses **`_CIS_ScrapePageText`** (`ChromeInfoScraper.ahk`), which walks the
page's UIA tree and joins every element Name. Read-only: no selection, no
clipboard, nothing for her to undo, and scoped to the page Document so browser
chrome never leaks in.

`_JChatClean` is **permissive on purpose**. A UIA walk yields one element Name
per line with no guaranteed turn markers, and the set of names is whatever the
site renders this week. So it drops what it positively recognises as chrome
(whole-line matches only, so a message containing the word "Share" survives) plus
consecutive duplicate lines (UIA repeats a Name up the ancestor chain), and keeps
everything else. It does NOT require a `You said:` marker before keeping
anything — that stricter rule returns empty the moment the marker is absent,
which is the worst failure mode: silent and total.

If the markers ARE present they survive untouched and `journal_import.segment`
splits the turns. If not, the conversation lands as one `ai` segment at full
depth — verbose, but nothing lost and nothing polluting the core document.

**`grab chat debug`** is the iteration tool. Every capture writes the raw scrape
to `%TEMP%\journal_chat_raw.txt` with a header (time, model, url, char count);
the debug command appends what the cleaner made of it and opens the file. This
is a page whose DOM we do not control, so when a capture comes back wrong the
evidence has to already be on disk rather than needing a repro.

### VERIFIED against a real page, and what it showed

Jamie ran `grab chat debug` on a live ChatGPT conversation (16,303 chars,
711 lines). Parsing now lives in **`Scripts\journal\chat_parse.py`** — Python, not
AHK, because it is pure text logic over a format we do not control and therefore
has to be testable: `chat_parse.py --file <dump> --report` runs it over any saved
scrape. That is how the real structure was found. AHK does the one part only it
can (walk the UIA tree) and hands the blob over.

What the real dump revealed:

- **There are NO `You said:` / `ChatGPT said:` markers** anywhere in the UIA
  output. Any parser keyed on them returns nothing.
- **~590 of 711 lines are the SIDEBAR** — chat history at three lines per past
  conversation (`<title>`, `Pin <title>`, `Open conversation options for
  <title>`) plus nav and the profile menu. Scoping UIA to the page Document does
  NOT exclude it, because ChatGPT is one SPA document.
- **The conversation is delimited by three reliable markers**:
  a timestamp line (`Today 12:55 PM`) opens a user turn, `Your message actions`
  ends it, `Response actions` ends the assistant's reply.
- A first pass that only dropped a junk LIST kept 13,081 of 16,303 chars — i.e.
  nearly all the sidebar. **Structure, not a denylist, is what separates
  conversation from chrome.**

### Roles are assigned RETROACTIVELY, at the end marker (2026-07-29, second capture)

The first version predicted a role forward from a timestamp line. Jamie's second
real capture showed two ways that silently loses whole messages — and the
`messages_on_page` check caught it: **5 messages on the page, only 3 read out**.

1. **A timestamp is not emitted for every message.** The dump had a
   `Your message actions` with no timestamp before it, so the parser was still
   waiting for one and dropped that message entirely. It was the message asking
   for a summary.
2. **The conversation does not begin at the first timestamp.** 58 lines of a
   complete assistant reply sat *above* it and were discarded.

So the structure now keys on the **end markers** — `Your message actions` /
`Response actions`. Whatever is buffered when one arrives IS that kind of message.
They are per-message and reliable, which is exactly why counting them detects loss
in the first place. Timestamps are now ignored entirely.

`find_start()` handles the sidebar instead: every past conversation contributes an
"Open conversation options for &lt;title&gt;" row, so the LAST such row ends the
history list; the profile menu and model selector are skipped after it. A trailing
message with no end marker (the newest reply, possibly still streaming) is flushed
as `ai` rather than dropped.

Same dump, after: **5 turns of 5, 67% of raw kept** (was 3 turns, 45%). The
remaining warning is correct and useful — turn 1 is an assistant reply, so an
earlier message of hers really was virtualised off the page, and she is told to
scroll up and re-run.

The turns are emitted as `You said:` / `ChatGPT said:` blocks, which
`journal_import.segment` then splits into `prompt`/`ai` segments.

Two bugs that fix found, both worth knowing:
1. The timestamp regex was case-sensitive and the text is `Today`, not `today`.
2. A bare `You said:` on its own line matched the segmenter's label pattern but
   left no text, so the role never carried to the lines beneath and the whole
   message fell through to `body`. Labels now set a **sticky** `turn_role` that
   following lines inherit until the next marker.

`grab chat debug` writes its report to a **separate** file
(`journal_chat_report.txt`). Appending it to the raw dump — which the first
version did — contaminates the parser's input on the next run and made every turn
appear twice.

**Still unproven:** Claude has no equivalent dump yet, so it falls to
`parse_permissive` (everything not obvious chrome, as one `ai` turn).

`chat_segments()` forces every captured segment to `prompt` or `ai` even when the
voice score says otherwise: text copied off a chat page is chat by definition,
and a `body` segment here would smuggle transcript into the core document. It is
shared by `add --chat` and `append-chat` so a conversation is stored identically
whether it OPENS an entry or lands UNDER one.

### The first live capture, and the four bugs it found (2026-07-29)

The parser was right and the capture still lost everything — twice, for two
unrelated reasons. Worth reading before touching this code, because three of the
four are general.

**0. The transcript never reached Python.** `cmd.exe` truncates a command line at
the first newline, keeping line one and discarding every argument after it, so
`--text "You said:\nhello…" --model ChatGPT` arrived as `--text "You said:"`. The
segmenter got a bare label, produced zero segments, and the new guard correctly
reported *"nothing was saved"*.

**The same bug meant a multi-paragraph `add journal` entry silently saved only its
opening line** — invisible until now only because the sole entry written through
that path was one line of test text. Every text-bearing command now takes
`--text-file`, and AHK's `_JShellText` writes the value to a file. Full writeup:
[design-decisions.md](../../../AutoHotkey/docs/design-decisions.md) § *Multi-line
text NEVER travels on the command line*.

**1. `chat_parse.py` died on a `→` and AHK stored the traceback.** A redirected
pipe on Windows is cp1252, not UTF-8. The full story and the standing rule are in
[design-decisions.md](../../../AutoHotkey/docs/design-decisions.md) § *Every
Python helper AHK shells MUST force UTF-8 stdout* — it applies to every script in
`Scripts/`, not just this one.

**2. Nothing verified the write.** `AttachChatToEntry` shelled `append-chat` and
tooltipped "attached ChatGPT chat" unconditionally, so a capture that stored
nothing reported success. Now `append-chat` prints `error\t<why>` and exits
non-zero when it appends zero segments, refuses text that opens with a Python
traceback, and `_JAppendChat` in AHK parses the segment count and shouts
`journal: NOTHING was saved` when it is zero. **A write that stores nothing must
never tooltip a success.**

**3. `grab chat` flooded the core document.** It routed the transcript through
plain `add`, which stores its text as ONE `body` segment at **core** depth — so a
captured chat became her own writing at the shallowest depth, the exact thing the
depth system exists to prevent. `add --chat` fixes it: same segmenter as
`append-chat`, then `ensure_core` promotes the opening turn so a chat-only entry
still renders. Measured on the real 4-turn capture: core 98 chars, summary 319,
full 5,988. Before the fix, core was all 5,988.

### Telling her when a capture is short

Jamie asked to be told when a capture is probably incomplete — *"just to know
that I didn't lose anything."* `chat_parse.py --meta <path>` writes a JSON
coverage summary that AHK reads back: `turns`, `user`, `assistant`,
`messages_on_page`, `saw_footer`, `complete`, `warnings`.

**The counts go in the tooltip on every success** (`attached ChatGPT chat — 4
turns (2 you / 2 ChatGPT)`), so a capture that grabbed less than she expected is
visible without going to look. If any warning fires, a modal appears **before the
write** with a "Save it anyway" / "Cancel" choice — the fix for a short capture is
to scroll to the top of the conversation and re-run, which is only useful if she
is asked first.

The strongest signal is **`messages_on_page` vs `turns`**: ChatGPT emits one
`Your message actions` / `Response actions` affordance per rendered message, so
counting those markers gives an independent expected turn count, and a mismatch
means the *parser* dropped something — silent and fixable, so worth shouting
about. Verified: a dump with 6 markers but 4 extractable turns reports
`The page has 6 messages on it but only 4 were read out`.

**What cannot be detected:** virtualisation. A message ChatGPT has unloaded takes
its markers with it and leaves nothing behind in the tree, so
"first turn is an AI reply" almost never fires for real truncation. That is
precisely why the counts are reported unconditionally rather than only on
suspicion, and why a long conversation gets an advisory warning on size alone.

## The `summary` send — a reply that counts as the summary

Jamie's flow: talk the entry through with an AI, then fire the **`summary`** send —
*"Please summarize the most important lessons learned from this journal entry.
Please include any juicy quotes or big lessons that stuck out to me in our
conversation, but please make it decently succinct so that it is more easily
readable than scrolling through the whole conversation."* — and let that answer
stand in for the whole conversation.

When `grab chat` sees a reply to that prompt, it files the reply as the entry's
**`summary` segment at `summary` depth** instead of burying it at `full` with the
rest of the transcript. That is what makes the middle depth real: *the entry plus
its takeaway*, without the conversation.

- The prompt lives in the **send store** (`sends.py`, group `journal`, body in
  `INIDATA\ClaudePrompts\summary.txt`), so she can reword it in "open send"
  without a code change — same rule as `journal thoughts`.
- **Two things are needed to make a new send speakable**, and missing either looks
  identical (Dragon just types the words):
  1. `sends.py cache` — the Caster rule reads
     `~/.claude/context/sends_dump.json`, NOT the store. Adding a send does not
     refresh it.
  2. A firing rule **for the context you are speaking in**. `send <choice>` lived
     only in `claude_commands.py`, scoped to VS Code / Claude desktop / Dragon's
     dictation box — so it was never live in ChatGPT-in-Chrome. Hence
     `rules/chatgpt_send_commands.py` (`CONTEXT_TOKEN = "chatgpt"`,
     `executable="chrome"`). 53 of 57 sends are `claude`-only, so that rule adds
     just the two `*` journal prompts.
- **AHK reads the send, the engine does not.** `_JSummaryPromptFile` resolves it and
  passes `--summary-prompt-file`; journal.py stays ignorant of the send system,
  the same division as `AskAIAboutEntry`. No send, no detection — nothing breaks.
- Matched on **words**, at ≥80% of the prompt's vocabulary, on the **LAST**
  occurrence: she may ask for a summary more than once in a long conversation, and
  the final one is the one she kept.
- **The prompt itself is demoted to `full`.** 284 characters of boilerplate at
  summary depth is precisely the noise the summary render exists to avoid.

Measured on a real four-turn chat: core 46 chars · summary 101 · full 466.

### When matching FAILS — `merge-chat` (2026-08-16)

`chat-match` works on the designed flow, and fails on the common one. ChatGPT
collapses a long pasted message behind "Show more" and the collapsed text is not
in the DOM, so a capture of "here is my entry, what do you think" frequently
does **not contain the entry at all**. Verified on a real Aug 2 capture: segment
1 is a 16-word framing line while the model replies *"I read the whole thing
carefully."* There is nothing for the matcher to match on, so `grab chat`
creates a standalone entry and the pairing — obvious to Jamie — is invisible to
the store.

`journal.py merge-chat <chat-id> --into <entry-id>` files it after the fact:
segments are appended with their roles, the chat entry is SOFT-deleted and
stamped `merged_into`. Two details matter:

- **Promoted depths are reset.** A chat-only entry has no `body`, so
  `ensure_core` promoted one of its turns to core. Under an entry that *has*
  writing, that promotion would put "What you think about this journal entry"
  into the document at the same depth as her journaling.
- **It refuses an entry with `body` segments** unless `--force`. That is
  writing, not a capture, and folding it into another entry would lose a real
  entry.

`merge-candidates --id X` feeds the Miller's **🔗 File this under an entry** row
(offered only on a chat-only entry). It ranks by **date proximity, not
similarity** — deliberately, because similarity is measured against the model's
paraphrase and sits around 0.3 even for a certain match, while a conversation is
nearly always about something written the same day. The overlap is shown as a
human tie-breaker.

Two real pairs were reunited this way. The Aug 15 one was confirmed by reading:
the reply discusses "who will be there?", "what if I'm misgendered?", "what if I
hurt my elbows?" — straight out of the Hermes entry. The Aug 2 one by counting:
the model names Sophie 6 times and age gaps twice, and the entry mentions Sophie
exactly 6 times and age gap exactly twice.

## Matching a chat back to the entry it came from

"Ask an AI about this" pastes `<standing prompt>\n---\n<the entry, rendered>` into
ChatGPT, so the conversation's **first user message literally contains the
entry**. When she later says `grab chat`, that conversation belongs *under* the
entry it came from, not in a new one. `journal.py chat-match` decides which, and
`GrabChatToJournal` attaches there instead of creating an entry.

**Matching is on WORDS, never characters.** The two sides are never
byte-identical: the paste carries the markdown render (`# 2026-07-29 — title`,
`**Me:**`, `> ` quote prefixes), ChatGPT re-wraps and smart-quotes the text, and
she may have edited a sentence before sending.

Two verdicts, deliberately treated differently:

- **`contained`** — the entry's words appear as a contiguous run inside the
  message. Unambiguous, so it attaches with **no question asked**, which is what
  she asked for. This is what the designed flow always produces.
- **`overlap`** — a fuzzy score. This one **asks**, because guessing wrong buries a
  conversation under an unrelated entry, and an entry is not a cheap thing to
  corrupt.

### Why overlap scores on DISTINCTIVE words only

Scored on raw words, `|entry ∩ message| / |entry|` has a nasty failure mode: a
**short** entry made of ordinary language scores 0.72–0.85 against almost any long
message, because nearly all of its few words are ones everybody uses. Measured
across all 99 entries, two short ones (`just took an imitrex…`, `have had a ton of
headaches…`) fuzzy-matched practically everything.

So `common_words()` derives a stoplist **from her own corpus** — any word in ≥25%
of entries carries no signal — and the score uses only what is left. No list to
maintain, and it adapts as she writes more. Currently drops 231 words.

Calibration, measured rather than guessed:

| | |
|---|---|
| Distinctive words per entry | min 4 · **median 139** · max 540 |
| Entries below the `MIN_DISTINCT_WORDS = 12` gate | **2** — exactly the two pathological ones |
| Self-match sweep, all 99 entries | **0 wrong, 0 ambiguous, 0 unexplained misses** |
| Real entries still matched uniquely after a 12% reword | **88/88** |
| False positives on unrelated messages | 0 of 3 controls |

`ai-conversation` entries are excluded as candidates: a captured conversation is
never the *source* of another conversation.

## Reading the chat attached to an entry

Three separate rows, because they are three separate intentions — and the chat
rows only appear when there *is* a conversation attached (`chat-info` reports
`turns`, so the Miller can ask before offering):

| Row | What it does |
|---|---|
| **📖 Read** | the entry at `full` depth — her writing, chat beneath it |
| **✏️ Edit the text** | `EditJournalEntry` — reopen her writing in the entry box |
| **🔗 Open the chat in Chrome** | reopen the *live* conversation from the stored url |
| **💬 View the chat log** | `chat-log` — the conversation **alone**, her writing not in the way |

### Attaching a chat must not overwrite the entry's type

`append-chat` used to set `entry_types = classify_entry(e)` wholesale, which on a
real classic entry returns just `['ai-conversation']`. Two things wrong with that:
the entry drops out of *"just the classic ones"* — the export filter she asked for
by name — and it **silently discarded a type she had set by hand**, even though
`cmd_retype` has honoured `types_source == "manual"` from the start. Now the
manual case only appends `ai-conversation`, and the auto case merges rather than
replaces. Same provenance rule as tags and people links.

### Handing an entry to an AI

**`AskAIAboutEntry(id)`** builds `<standing prompt>\n\n---\n\n<entry>` and pastes
it into the chat in front, leaving it unsent so she can read it first.

The standing prompt is **the send named `journal thoughts`**, not a string
literal — a prompt she will want to reword belongs in the send system where
"open send" can reword it without a code change. Current body: *"This is a
journal entry that I just wrote. What do you think about it? Mostly from a
therapy / self-learning perspective."* The literal in `JournalCapture.ahk` is
only a fallback for a missing send.

### THE DISPATCHER RULE (a real trap)

Any viewer action that touches **Chrome, the clipboard, UIA or the context
chain** must go through `_JournalMainfun(fn, args*)`, which runs it in a fresh
MAINFUNCTIONS process. It must NOT be called directly from the viewer.

Why this is not optional: the viewer is a light own-process host with a
hand-curated `#Include` closure, and `ahk_include_closure.py` **excludes
`Scripts/`** from its checks (`SKIP_PATH_PARTS`). So a viewer action calling
`ChromeCurrentUrl()` passes `validate`, passes the closure check, passes the
Miller convention check — and then throws at runtime with nothing having warned
you. The first cut of the entry-actions menu did exactly this. Dispatching gets
the full closure for free and keeps viewer startup fast.

## Inline markup — tagging by writing it into the entry

`Scripts\journal\markup.py`. She tags entries by writing the metadata into the
text itself, at the top or bottom (or both), and the same notation works whether
she typed it into the journal box or into the Google Doc that later gets
re-imported:

```
rate: 7
type: insight meditation
tags: Charli grief relationships

I had a difficult and lonely weekend …
```

`apply_markup` runs on **every write path** (`add`, `enrich`, and the importer),
so there is one notation and nothing to keep in sync. The lines are consumed —
stripped from the body — and the values applied as `manual` provenance, because
they are her explicit instruction and neither the auto-tagger nor `retype` may
overwrite them. Keywords are flexible (`rate`/`rating`/`importance`,
`tag`/`tags`/`themes`, `type`/`kind`/`form`) and the colon is optional, since
dictation drops it.

**Word boundaries are the hard part.** Values are space-separated when dictated,
but many real tags are themselves multi-word (`chronic pain`, `inner critic`,
`parts work`). So `tags: chronic pain grief` is ambiguous. `split_values` does a
**greedy longest match** against the live vocabulary — longest phrase that is a
real tag wins, then continue — which resolves
`tags: inner critic shame parts work` to exactly three tags. Commas, when
present, win outright and need no guessing.

**People and tags share one line, deliberately.** She thinks of "what this entry
is about" as one list, mixing a person with two themes. Each value resolves
against the person store FIRST and the tag vocabulary second, so nothing makes
her remember which axis a word belongs to.

Anything resolving to neither is returned as **unknown** — never silently created
(a typo would quietly grow the vocabulary that books and quotes also draw on) and
never silently dropped. `add`/`enrich` print `unknown-tag <v>` / `unknown-type
<v>` rows; `_JournalConfirmUnknowns` offers each one via
`_ConfirmationModalGui`, and answering yes both creates it and applies it. The
importer prints the same values as a summary.

**A gotcha worth remembering:** `apply_markup` must run BEFORE the entry id is
minted. The id is slugged from the opening words, so markup at the top of an entry
otherwise produces `journal:2026-07-29-rate_8_type_insight_tags_cedar`.

### Speaking the markup

`caster\rules\journal_markup_commands.py` (`JournalMarkupRule`):

```
"rate 8"                         -> types   rate: 8
"tag Charli grief relationships" -> types   tags: Charli grief relationships
"type insight meditation"        -> types   type: insight meditation
```

These **insert text**, they do not call a function — the metadata belongs in the
entry, and `markup.py` reads it on save.

Values are `Dictation`, not a `Choice`. Jamie asked for "a dictation list of
possibilities but also accept just any normal input", and a Dragonfly element
cannot be both; free dictation accepts anything and the greedy matcher resolves
it against the real vocabulary afterwards. Dictation's capitalisation is harmless
because the matcher normalises case.

**Scope is `function_context=_journal_writing`** — the writing box (matched on the
words `journal entry` in the window title) or Google Docs. Deliberately narrow:
`type <anything>` claims a large slice of the command namespace and must never be
live globally.

**Every one of them starts with `Key("end")` and a newline**, so the markup always
lands on a line of its own. `markup.py` only recognises a metadata line anchored
at the start of a line, so speaking one mid-sentence used to produce
`...a hard day.tags: grief`, which parses as prose — the tag was **silently
lost**, the worst outcome available. `end` first because the caret is normally
inside the line she just dictated.

**Saying several in a row is fine and always was.** `extract()` accumulates across
*all* lines, so `tag Charli` then `tag mindfulness` gives one entry with the
person `charli` and the tag `mindfulness` — two lines are not a contest between
two values. Verified.

`markup.py` also **un-glues** a keyword jammed onto the end of a prose line, so a
manually typed `I had a hard day.tags: grief` is recovered too. The guard is the
**missing space**: every keyword is also an ordinary English word, so matching
them anywhere inside a line would maul real sentences — but prose always puts a
space before the word (`I talked about my type: A personality`) and this artifact
never does. Verified against three prose decoys, none of which split.

## Editing from the Miller

Drilling an entry gives: **Read** (Notepad++) · **Importance** · **Type** ·
**Tags** · **Ask an AI about this** · **Attach the chat in front** ·
**Summarize its chat** · **Re-derive type and tags** · **Delete this entry**.
Enter on the entry row still reads it (row action `1`), so the common case costs
no extra keystroke.

**Deleting always asks, and the default button is Keep.** The modal names the
entry (title + date) rather than saying "this entry" — the row it fired from is
off screen once the confirmation has focus, and "are you sure?" with nothing to
check against is how the wrong thing gets deleted. It is a **soft** delete
(`status=deleted`, which `query` filters out); the words stay in `journal.json`.
Nothing in the Miller can hard-delete — that needs `journal.py remove <id>
--hard` at a prompt, deliberately, because an entry is years of writing that
exists nowhere else once the Doc stops being the source of truth.

Writing also lives at the bottom of the browse root — **New entry — type it**
and **New entry — from the chat in front** — so an entry can be started without
leaving the Miller.

`add journal` / `grab chat` open the viewer **on the new entry**, not at the
generic root: `OpenJournalAtEntry(id)` passes the id to the viewer, whose root
becomes that entry's action page plus a `◀ Browse the whole journal` row. It
closes any open viewer first, because `LaunchMillerViewer` reuses an existing
window and ignores its args — correct here rather than a workaround, since she
just wrote the entry and asked to be taken to it.

**Tags** is an in-line toggle list, per the standing rule that a value from a
known set is picked and never typed. Ordering is the same idea the **book**
tagger uses (Jamie asked for parity): checked tags first, then tags that
**co-occur** with them across the corpus — `mc.rank_related_tags`, whose
sqrt-frequency damper rewards tags that are *disproportionately* associated
rather than merely common — then everything else by usage. Recommended rows are
marked ★. Without this the 142-tag vocabulary is an alphabetical wall.

**Type** is a multi-select over the form vocabulary. Toggling sets
`types_source: "manual"`, and **`retype` then skips that entry** — the same
provenance rule that protects tags and people. A classifier that silently
overwrote her corrections would make the editor pointless. `retype --force`
overrides.

**Importance** is a 0-10 single-select — her own axis for "how worth re-reading
is this", never derived and never guessed by a model. It shows on entry rows as
★N. The scale anchors (`not worth re-reading` / `worth a look` / `essential`)
live in the row NAME rather than the detail column, because a detail wide enough
to hold them gets pushed off the right edge of the pane at some widths.

People editing is deliberately not there yet: Jamie asked for tags first.

### One-row lookups

The Miller needs a single entry's metadata in a few places (its header row, the
current importance). `viewer --mode entries --id X --excerpt 0` returns exactly
one row with no excerpt. Before that existed, those paths pulled **every** entry
*with* 1600-character excerpts — ~150 KB of JSON to read one title — which was
slow enough to visibly truncate the render mid-fill.

## Summaries — what makes `--depth summary` real

`journal.py summarize` condenses an entry's `ai` segments into one `summary`
segment (task `journal_summary`, local). Without it the middle depth only adds
her side of a chat and the three export levels collapse into two useful ones.

The prompt preserves memorable formulations verbatim in quotes and forbids adding
advice, and a summary that comes out **longer than what it condensed** is
rejected as a failure rather than stored. `set_summary` inserts the segment
directly above the first `ai` segment so it reads in the right order.

## Local-LLM assists (free; never the paid Claude gateway)

Tasks `journal_title`, `journal_tag`, `journal_type` in
`Scripts\local_llm\tasks.json` (qwen2.5:7b-instruct).

**Titling.** 52 of 98 entries were untitled (the older ones). `journal_title`
deliberately does NOT use the shared `title_line` postprocessor — it Title Cases,
which fights her house style (`message to Raven about veganism and food and
shame`, not `Message To Raven About Veganism`). Instead:

- the prompt makes lower case a hard rule with worked examples;
- `clean_title` strips label prefixes, snake_case and over-long tokens (a
  run-together blob like `amiglovesanddonuts` passes a word *count* check);
- `_grounded` requires at least one content word of the title to actually occur
  in the entry, which catches well-formed but invented titles;
- `recase_people` restores capitals on names using the person store — the one
  casing error the model makes that can be fixed exactly;
- `fallback_title` derives from the entry's opening clause if two attempts fail.

**Tagging.** Same three guards as the quote tagger: an existing-vocab-only
prompt, a junk filter (digits / >2 words / >24 chars), and a **frequency gate** —
a proposed NEW tag stays dormant in `journal_tag_review.json` until it recurs on
3+ entries, so one-offs never nag. Only existing-vocab tags are applied; genuine
proposals need `review-approve`.

The tagger is fed `text_for_llm`, which prefers her own words (`core`) and only
falls back to deeper layers if there are none — tagging an entry by ChatGPT's
prose would drift the vocabulary toward whatever the model happened to say.

**Model backend is LOCAL only.** `Scripts\llm_gateway.py` (Claude) is DEPRECATED
for automation — gateway `claude -p` calls now cost real money. For a sharper
pass, ask the interactive Claude Code assistant in-session ($0 on Max).

## Tag vocabulary

`seed_journal_tags.py` opted 116 existing tags into `journal` and created 17 the
journal needed: `parts work, inner critic, burnout, boundaries, jealousy,
abandonment, attachment, breakup, dating, food, veganism, sleep, medical,
substances, writing, transition, work`. A further 9 were promoted from the review
queue after the first tagging pass (`communication, meditation, sexuality,
depression, intimacy, gender, loneliness, joy, connection`). 142 tags now apply
to journal entries.

NOT opted in: fiction genres, music genres, specific works, and the social
groupings that belong to people — those describe media she consumes, and letting
them into the journal picker would bury the tags that actually apply.

**Safety note:** `media_tags.json` has been silently clobbered once before (a
corrupt write left it loading as an empty taxonomy, which killed umbrella rollup
across the entire book/media system with no error, because `load_taxonomy`
swallows a parse error into an empty dict). Every writer here backs up first,
writes to a temp file, re-parses it, and refuses to shrink the tag count.

## UI

**`open journal`** → `OpenJournal` (Miller viewer, own process, per the enforced
convention). Sections: Recent · By year · By type · By theme · By person ·
Search · Generate document · ⚙ Settings.

> **Generate document is now the SHARED export system** — see [[EXPORT_SYSTEM]].
> The old hardcoded depth × type grid was replaced (2026-08-16) by named presets
> stored as data in `INIDATA\export_presets.json`, mounted via
> `_ExportsAsNode("journal")` from `Helpers\ExportsMenu.ahk` — the same node the
> quote store mounts. The grid could express neither "everything except the
> ChatGPT ones" nor "only the ★10 ones", which were the two documents she
> actually wanted. Documents render into the Obsidian vault and the default one
> regenerates itself after every write to the store.

- Focusing an entry fills the right pane from `preview_text`. The engine wants a
  plain **string**, not a callback, so the excerpt rides in on the same TSV row
  as the rest of the list — one shell-out per column instead of ~100.
- Enter on an entry opens the full render in Notepad++ (the preview pane is for
  glancing; that is for reading).
- **Generate document** is the modular export: pick a depth, then a scope (all,
  or one entry type), and it writes to `journal.export_dir` and opens the file.

Settings (declared, per the standing rule that every Miller exports its
tunables): `journal.recent_count`, `journal.preview_depth`, `journal.export_dir`.

## CLI quick reference (`py Scripts\journal\journal.py …`)

```
add --date … [--title --text --types --tags --time]
add --date … --text-file <path> --chat [--model --url]    # a CAPTURED CHAT: segment
                                #   into prompt/ai turns instead of one core body blob
#  --text-file REPLACES --text for ANYTHING MULTI-LINE (add / add-segment /
#  append-chat / chat-match). cmd.exe truncates a command line at the first
#  newline. Never pass a paragraph as --text.
chat-match --text-file <first user message> [--min 0.72 --limit 3]
                                # which entry did this conversation come from?
                                # TSV: id, score, kind(contained|overlap), date, title
chat-info --id X                # turns / url / model of the attached conversation
chat-log  --id X                # the conversation ALONE, readable
get-body  --id X                # her own writing, RAW (round-trips into the edit box)
set-body  --id X --text-file <path>
                                # replace her writing ONLY. Chat / summary /
                                # commentary and hand-set types are left alone;
                                # inline markup IS applied; types are NOT re-derived
                                # (she is editing by hand -- same rule as retype)
add-segment <id> --role body|commentary|summary|quoted|prompt|ai --text … [--depth --url --model --speaker]
list [--type --tag --person --year --from --to --text --untitled --untagged]
show <id> [--depth core|summary|full]
render [--filters] --depth … [--out FILE] [--title …]      # the document generator
types | set-types <id> --types … | set-title | set-tags | tags | remove <id> [--hard]
retype [--commit]               # re-run the deterministic form classifier
enrich --id X                   # classify + tag + people + title, ONE entry
toggle-tag <id> --tag X | tag-vocab [--id X]     # what the Miller's toggle list uses
summarize [--id X] [--force] [--commit]          # condense an entry's AI turns
append-chat --id X --text … [--model --url]      # attach a captured conversation
people-scan [--commit]          # RETROACTIVE re-link; safe to re-run forever
people-propose [--min N]        # recurring names not yet in the person store
person-add <name> [--aliases --tags] | person-link <id> --person X
person-merge <name> --into <name> [--commit]     # undo a duplicate
title-missing [--limit N] [--commit]   # LLM titles
tag-batch [--limit N] [--commit]       # LLM tags
review-list [--min N] | review-approve <tag> --parents a,b [--apply] [--quote] | review-reject
viewer --mode entries|years|types|tags|people|entry   # TSV feed for the Miller
stats
```

`remove` is a SOFT delete by default (`status=deleted`, which `query` filters
out) — the words stay on disk. `--hard` actually drops the entry.

## Open threads

- **The Doc is still the historical source**, but no longer the only way in:
  `add journal` writes directly to the store. Re-import stays idempotent, so both
  paths coexist while she decides.
- **Pulling an AI's reply back in** works via `grab chat here` / "Attach the chat
  in front", which re-copies the whole conversation. A cheaper incremental
  capture (only the turns added since last time) is unbuilt and probably
  unnecessary.
- **People editing in the Miller** — deliberately deferred; tags came first.
  `person-merge` exists on the CLI but has no UI row yet.
- **`Rigo`/`Rego` and `Ally`/`Allie`** are separate records pending
  confirmation that they are separate humans. If not: `person-merge Rego --into
  Rigo --commit` then `people-scan --commit`.
- **Chat capture is clipboard-based** and untested against a very long
  conversation (thousands of turns) or a page that lazily unloads early turns.
  If a capture comes back short, that is the likely cause.
- **Poems** — the planned third parallel text system, with its own tag axis.
  `GrabChatTranscript` was written generic specifically so it can serve that too.
