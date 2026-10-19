---
tags: [programming, journal, caster, ahk, chatgpt, writing, media-system]
created: 2026-09-03
related: ["[[QUESTIONS_SYSTEM]]", "[[NEW_TAB_PAGE]]", "[[JOURNAL_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[EXPORT_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[JOURNAL_SOURCES]]"]
---

# The journal question loop

Jamie writes an entry, hands it to ChatGPT ("ask an AI about this"), and it asks
her questions back. This is the loop that closes: the questions come into the
entry they were asked about, she answers them there, and the answers go back to
the conversation that asked. Round after round, in the same entry.

Built 2026-09-03, on top of [[JOURNAL_SYSTEM]]. Nothing here is a new capture
mechanism — it rides the conversation reader that `grab chat` already uses.

## The shape

```
add journal              she writes the entry
   ↓
ask an AI about this     AskAIAboutEntry -> the standing prompt + the entry
   ↓                     (the prompt asks ChatGPT to NUMBER its questions
   ↓                      and repeat the numbered list at the end)
grab questions           the newest reply's questions ->
   ↓                       · the clipboard, always
   ↓                       · appended to the entry as "round N"
   ↓                       · the writing box, open on that entry
she answers them         typed under each question, in her own entry
   ↓
make answers             the newest round + her answers -> back into the chat
   ↓
   └─────────────────► grab questions again = round N+1
```

## Three bugs this started as

All three were real and all three are fixed. Worth keeping, because two of them
were invisible and the third had been lying about its cause for months.

### "journal: entry is empty" — a shared temp file, not an empty entry

`_JShell` captured Python's stdout through **one fixed path**,
`%TEMP%\journal_viewer.txt`. Several processes share the journal stack at once:
the writing box's GuiHost, the Miller's own viewer process, and every MAINFUN
dispatch. When two overlap:

```
process A: cmd /c py … > tmp    opens tmp with `>`, TRUNCATING it to 0 bytes
process B: FileDelete(tmp)      fails silently — A holds the handle
process B: cmd /c py … > tmp    cannot open it either; cmd exits ~instantly
process B: FileRead(tmp)        reads A's zero bytes -> ""
```

and B reported that as **"the entry is empty."** The tell in `ahk_event.log` is
the timing: eleven consecutive `AskAIAboutEntry` calls returned in **50–60 ms**
each, where a real `py journal.py viewer` takes 300–500 ms. It cleared up on its
own after about a minute — when the GuiHost that was still running `enrich` and
`generate journal` for the entry she had just written finally finished. That is
why it looked like the store was slow to catch up.

Fixed by `TempCapturePath` / `ShellCapture` in `Helpers\CommonFunctions.ahk`: a
capture path unique per process and per call, plus **the exit code**, so "the
shell-out failed" and "the command found nothing" stop being the same empty
string. Applied across `JournalMenu`, `JournalCapture`, `ChatGrab` and
`ChatGptApi` — 15 sites. The debug artifacts (`chat_grab_raw.txt` and friends)
deliberately keep their fixed names; `grab chat debug` opens them.

**`ExportsMenu.ahk` was the sixteenth, found later (2026-09-04).** `_EXShell`
still hand-rolled the same pattern into a shared `%TEMP%\exports_viewer.txt`, and
it sits on the after-save path: `journal.after_save` fires a detached
`generate journal` on every write, so opening the exports menu in that window put
two of them on the one file. It now goes through `ShellCapture`, with
`_EXShellChecked` for the caller that has to tell the two failures apart —
`_ExportPresetPath`, whose empty answer reached Jamie as **"no document named
'journal'"**: a confident, wrong sentence about a document she reads every day.

The lesson generalises: **a fixed `%TEMP%` filename is the bug.** Anything
hand-rolling `cmd /c … > tmp` is this defect waiting for a second process.

### The prompt sending itself — a trailing newline

`SendToChatGPT` typed the prompt with `SendText`, which sends each character as a
unicode keystroke. **Chrome reports U+000A as `key: "Enter"`**, so a prompt
carrying a newline pressed Enter inside the composer and sent what had been typed
so far. The send store puts a trailing newline on every body it stores, so the
standing prompt went as a message of its own and the entry was then pasted into
the empty composer behind it.

Confirmed from the conversation it produced: turn 0 was the 333-character prompt
alone, turn 2 the entry — two messages where there should have been one.

Fixed in `_ChatGPTTypeMultiline`: trailing whitespace dropped, internal line
breaks sent as `Shift+Enter`, which is the newline a chat composer actually
means. Every caller was exposed, not just the journal — Kindle vocab, YouTube
subtitles ×2 and EBookReader all go through the same function.

### `first_prompt` never arrived, so no chat ever attached to its entry

`_JChatSourceEntry` decides which entry a captured conversation came from by
feeding its opening message to `chat-match`. It read `res["first_prompt"]` —
a key **`GrabChatTranscript` never set**, and which `ChatGptApiFetch` did not
copy out of its meta either. Guarded with `.Has()`, so it silently returned ""
every time and every captured chat became a new orphan entry.

Fixed end to end, and `chatgpt_api.py`'s cap raised from 400 to 4000 characters
to match `chat_parse.py` — `contained` needs the whole entry to still be inside
the message, and entries run to thousands of characters.

## Where the questions come from

**Not OCR, and not the grab-text overlay** — both were on the table and neither
is needed. `ChatGrab.ahk` already reaches ChatGPT's own backend
(`ChatGptApi.ahk`) and gets the conversation as role-tagged JSON, so "the newest
reply" is a field rather than a guess about what is on screen. No scrolling, no
clipboard, and it works with the tab in the background.

Claude and Gemini have no equivalent endpoint, so they fall back to the rendered
`You said:` / `<model> said:` transcript and its **last** assistant block. The
general split of that dialect is unsafe (a reply quotes her back using those
literal words, which is why the journal ingests turns and not text) but the last
block is safe: everything after the final assistant marker is that reply, however
many markers it quotes inside itself.

## What counts as a question

`Scripts\journal\chat_questions.py`. Pure text logic over markdown nobody here
controls, so it is testable against a saved capture — the same reasoning as
`chat_parse.py`, and the only reason these rules are measured rather than
guessed.

> **The prompt asks for restraint, and has to.** The formatting demand ("number
> every question, then repeat the full list") reads as a quota on its own: Jamie
> got a round of noticeably worse questions and diagnosed it exactly — *"I think
> because it's trying to milk out questions based on our prompt"* (2026-09-04).
> The wording now says quality over quantity, that an irrelevant question should
> simply be left out, and that somewhere between **1 and 8** is about right.
> Extraction is unaffected: a shorter recap is still a recap. (It did
> collide with the old append-the-leftovers rule two days later — see
> below.)
>
> The prompt lives in **two** places — the send body and a fallback string
> literal in `AskAIAboutEntry`, used only when the send is missing or emptied. A
> drifted fallback silently sends a prompt she rewrote, so
> `test_journal_standing_prompt.py` now parses the literal out of the AHK and
> fails when the two disagree.

1. **The numbered recap, and nothing else.** The standing prompt asks ChatGPT to
   number its questions and repeat the numbered list at the end. When that list
   is there it IS the answer — the model telling us what it considers a
   question, which no heuristic of ours beats. Taken as the LAST run of
   consecutive `1. 2. 3.` items, most of which end in "?". Last, not first: a
   reply often contains an earlier numbered list that is not the recap.
2. **The loose scan alone**, when there is no recap. The tooltip says so
   (`5 questions (not numbered — scanned)`), because what landed in the entry is
   then this system's guess rather than the model's own list.

### The merge that used to sit between those two, and why it is gone

Rule 2 used to be *"anything question-shaped the recap did not cover, appended
after it"* — to catch a reply that sets one more question apart in its own
paragraph, "often bolded even", which is a thing Jamie asked for.

Measured against a real round (2026-09-06, *T1/League and Dysregulation*) it was
**wrong five times out of five**. The reply asked five numbered questions; the
round landed in her journal with ten. The five it invented were:

| what it added | what it actually was |
|---|---|
| *I'm activated; how do I make myself less activated?* | one half of a quoted contrast between two framings |
| *I'm activated; what does this activation want to do?* | the other half |
| *But why should "getting ready to see someone you like" necessarily culminate in calmness?* | a rhetorical aside mid-paragraph |
| *Can my system move flexibly among different states…* | a reframed **goal**, written as a question |
| *Is this about to end? Is this about to turn around? …* | five short sentences describing what her nervous system does during a close game |

None was addressed to her.

**Every syntactic filter that would have caught those also kills the case the
merge existed for.** The keeper in the test fixture —

```
And the one I'd sit with for a while:

> **If accommodating my body didn't mean surrendering, what would gentleness feel like?**
```

— is blockquoted and first-person, exactly like two of the five. Bold, italics,
blockquote, a colon lead-in, second person: the genuine keeper and the false
positives share all of them. The difference is *meaning*, not shape, so the
shape-based rule went.

**The job moved to the prompt**, which is the mechanism designed for it: the
final list "is the only part I read back, so it has to hold every question you
want me to answer, including any you set apart as especially worth sitting
with." Guessing alongside an instruction that explicit is how you get ten
questions from a reply that asked five.

This also resolved a conflict introduced two days earlier. The prompt had just
been reworded to make the recap **curated** ("if a question does not feel
relevant, do not feel the need to include it") while the extractor was still
appending everything left out — the two instructions pulled in opposite
directions, and the extractor won.

`extra` is still counted and returned, so a reply that quietly stops numbering
is visible. It is no longer acted on. The real reply is kept whole as
`tests/fixtures/reply_2026_09_06_league.md` rather than reduced to a snippet,
because the shapes that fooled it are the point.

### THE MARKS COME OFF BEFORE THE NUMBER IS LOOKED FOR

Both spellings are common and only one of them used to parse:

```
1. **What is the manager afraid of?**      number outside the bold   OK
**1. What is the manager afraid of?**      number INSIDE the bold    missed
```

The second starts with `*`, so `_LIST_RE` tried it as a **bullet**, wanted a
space after it, found another `*` and failed. The line came back unnumbered, the
run never formed, and a reply that asked five numbered questions was read as
having asked none. Round 2 on 2026-09-10 landed in the journal as one unrelated
sentence scraped by the fallback scan, and the tooltip said
`(not numbered — scanned)` — accurately, because it genuinely could not see them.

`_scan_lines` now runs `strip_markdown` over the line *first*, then matches.
Safe for real bullets: `* item` keeps its `* ` (a lone asterisk with no closing
partner is not emphasis) and `- **bold**` becomes `- bold`, which still matches.
The real reply is kept whole as `fixtures/reply_2026_09_10_bold_numbers.md`.

### A FRAGMENT IS NOT A QUESTION, and the capital is the test

Measured on one real conversation (6 turns, 29,950 chars). A naive scan of the
last reply returned **17 "questions" for a turn that asked 10**, because a reply
routinely breaks one question into a bulleted list of candidate answers:

> **What exactly is pleasurable about overriding the body?**
> - the feeling of being powerful?
> - freedom from limitation?
> - escaping helplessness?

Those six are pieces of the question above them, and taking them as questions
would have put six fragments of one thought into the journal as six separate
prompts. They are also, without exception, **lowercase** — because each continues
a sentence rather than starting one. Requiring a leading capital dropped all six
and cost nothing real.

That rule is what makes step 2 safe. Without it, the same merge would add the
fragments back.

## Rounds

A round is one assistant turn, appended to the entry's **body**:

```
---

## ChatGPT questions — round 1

1. What is the hyper-vigilant manager afraid would happen if it stopped…

2. When did I learn that my body couldn't be trusted to tell me what it needed?
```

**The body, not a segment of its own.** `EditJournalEntry` shows only the body
segment, and answering inside the entry is the whole point. A blank line after
each question is deliberate: it is where the answer goes, and a list with no room
in it invites nothing.

`next_round_number` counts the rounds already there. `newest_round` slices out
the last one **plus everything written under it** — that is what `make answers`
pastes back, and taking only the newest is what stops it re-asking questions she
answered days ago.

**A re-grab is not a new round.** "grab questions" is a thing she says twice —
once because she meant to, once because the first did not visibly do anything —
and the duplicate check makes the second a no-op (exit 2, "round 1 is already in
this entry") rather than a doubled entry.

## Which entry a conversation belongs to

Four ways, best first, because filing a round under an unrelated entry corrupts
something expensive:

1. **The conversation id**, once one round has landed (`journal.py find-chat`).
   Exact, and what makes every round after the first free. Stored on the entry as
   `chat_conversation_id` by `set-chat-url`.
2. **`chat-match` returning `contained`** — her entry's words verbatim inside the
   chat's opening message. Unambiguous, so it attaches silently.
3. **The last-ask record.** `AskAIAboutEntry` writes down which entry it just
   sent (`INIDATA\journal_last_ask.json`), and a grab within
   `journal.ask_recall_hours` offers that entry. **This exists because ChatGPT
   sometimes turns a long paste into an ATTACHED FILE rather than inline text** —
   observed live — and when it does, the opening message is just the standing
   prompt and (2) has nothing of hers to match on.
4. **`chat-match` returning `overlap`** — a fuzzy score, so it asks.

(3) and (4) ask. Only (1) and (2) are certain enough to write without a word.

**No match at all = the clipboard and a tooltip saying so, and nothing opened.**
Jamie was explicit about that: an unmatched grab must not go opening a journal
at her.

**The clipboard is never skipped**, on any path. It is the one outcome that
cannot fail, so it happens before any write — if the match is wrong or the append
fails, she still has the questions.

### `grab chat` uses the same four now — it used to use two

The resolver above is `JournalEntryForChat` in `Helpers\JournalCapture.ahk`, and
**both** `grab chat` and `grab questions` go through it. That is new as of
2026-09-10.

`grab chat` previously had only tiers (2) and (4) — no conversation id, no
last-ask record. When the opening message missed, it silently made a NEW entry
and Jamie had to delete it and re-do the attach from the Miller. The event log
for 2026-09-08…10 shows `GrabChatToJournal` followed within twenty seconds by a
manual `AttachChatToEntry` on the entry it should have found itself, four times:

```
16:53:20  GrabChatToJournal()                              ← new entry
16:53:30  deleted "(untitled)"                             ← she removes it
16:53:36  AttachChatToEntry("…buttplug_exile_teen_dream…") ← the way that works
```

The attached-file problem noted in (3) is not occasional — for a long entry it
is the norm, and it is exactly why tier 1 has to exist. Measured on the
2026-09-10 conversation:

```
first user message   940 chars — all of it the standing prompt, no entry words
the entry            7395 words
chat-match           returned nothing at all
find-chat --url      resolved the entry exactly
```

The link was already stored. `grab chat` just never asked for it.

### The link is written from every direction

`set-chat-url` used to be called only by `grab questions`, so the hardlink
existed only if she happened to grab questions first. It now happens in
`_JAppendChat` — the one place an attach is performed, which covers `grab chat`,
`grab chat here` **and** `attach the chat in front` — plus in
`_JournalCreateAndOpen` when a chat is saved as its own entry, so a second
`grab chat` on that conversation attaches to it instead of making a rival copy.

Best-effort and silent: the chat is already saved by the time it runs, so a
failure costs the next match its shortcut, not her writing.

Pinned by `test_journal_chat_resolution.py` — both commands delegating, no
private `chat-match`/`find-chat` left in either wrapper, and tier 1 ahead of
every dialog. Source-shape tests rather than behaviour tests, because what
regressed was the two commands drifting into two implementations.

## Voice commands

| Phrase | Does |
|---|---|
| `grab questions` | newest reply's questions → clipboard + the entry it came from + the writing box |
| `grab questions here` | same, into the NEWEST entry, no matching — the escape hatch |
| `see questions` | clipboard only, nothing written |
| `grab answers` (also `make answers`) | every answered, not-yet-sent question + answer → back into the entry's own conversation, staged not sent |
| `journal prompt` | today's unanswered question, opened in the writing box |
| `add prompt` | the selection / the reply's questions / an entry's rounds → the quote store as prompts |
| `star question <n>` (also `star <n>`, `mark question <n>`) | question n's affinity +1 (shown as stars on its line). In the writing box or on the entry's chat |
| `unstar question <n>` (also `unstar <n>`) | affinity −1 (through zero = buried) |
| `open special questions` (also `show special questions`) | the journal opened on every question at affinity 1+ |

All four live in `journal_commands.py` with the `grab chat` family — same engine,
same store, same global scope. `make answers` rather than `send answers` because
`send <choice>` is three separate Choice rules and a literal sharing that prefix
is asking for trouble.

Nothing is ever submitted. Same rule as `AskAIAboutEntry`: the message is staged
and she presses Enter, so a mis-grab is one Ctrl+A away from being discarded
rather than something already sent.

### Where a grab leaves her — the bottom of the box

`journal.questions_open_editor` opens the writing box on the entry so she can
answer the round straight away, and the round is at the **end**. The box used to
open with the whole entry SELECTED, because that is a multiline Edit's default
on focus — so a 7000-word entry sat highlighted with the caret at the top, and
the next character typed would have replaced all of it.

`caret_at_end` fixes it, with one correction: it sent `{End}`, which is
end-of-**line** and lands at the end of line one. It sends `^{End}` now — end of
the text, which is also where the questions are. Every caller of the shared box
gets this; for a single-line prefill the two are identical.

## Settings

- `journal.ask_recall_hours` (12) — how long after "ask an AI about this" a
  grabbed round still assumes it belongs to that entry.
- `journal.ask_auto_read` (on) — grab read by itself once the reply is finished.
- `journal.ask_reply_wait_minutes` (8) — how long that waits for the reply.
- `journal.questions_open_editor` (on) — land in the writing box after grabbing.
- `questions.affinity_weight` (1.5) and `questions.rest_days` (7) — how
  starred questions are favoured by the daily prompt. See [[QUESTIONS_SYSTEM]].

## Files

| Layer | File |
|---|---|
| Extraction (pure, testable) | `Scripts\journal\chat_questions.py` |
| Store commands | `Scripts\journal\journal.py` — `append-body`, `set-chat-url`, `find-chat` |
| AHK | `Helpers\JournalQuestions.ahk` |
| Shared shell-out helpers | `Helpers\CommonFunctions.ahk` — `TempCapturePath`, `ShellCapture` |
| The standing prompt | the send named `journal thoughts` ("open send") |
| Tests | `Scripts\codebase_tools\tests\test_journal_standing_prompt.py` |
| Voice | `rules\journal_commands.py` |
| Prompts (AHK) | `Helpers\PromptCapture.ahk` |
| Prompts (store) | `quotes.py add-prompts`, entry type `prompt` |
| Tests | `Scripts\codebase_tools\tests\test_chat_questions.py` |
| Tests | `Helpers\Tests\test_prompt_capture.ahk` (pure) |

## Prompts — `add prompt`

Built the same day. Many of these questions are good standalone journalling
prompts, flash-card shaped, and worth keeping apart from the entry they came out
of.

**A prompt is a quote**, not a new store. [[QUOTES_SYSTEM]] already has a FORM
axis (`entry_types`), so `prompt` is a fifth value beside
`quote / poem / exercise / meditation` — and the shared tag vocabulary, the LLM
auto-tagger, the umbrella hierarchy, facets, favourites and the quotes viewer all
work on prompts from the moment the type exists. No new engine, no new viewer, no
new registry. That is the entire argument for a type over a sibling system.

`AddPrompt()` in `Helpers\PromptCapture.ahk` is **contextual, in this order**:

1. **A selection**, wherever she is — Obsidian on the journal document, a chat, a
   book. First, because it is the one case where she has said exactly what she
   means, and overriding an explicit selection would be the worst failure here.
2. **An AI chat in front** → every question in the **whole conversation**,
   through the same `chat_questions.py` the round loop uses (`--all-turns`). Not
   a second extractor: one place decides what a question is.

   Whole conversation, deliberately, where `grab questions` takes only the newest
   reply. The two commands want different things: that one is appending a *round*
   to an entry, so sweeping the chat would re-append everything she has already
   answered; this one is harvesting keepers, and a good question asked four
   replies ago is worth exactly as much as one asked just now.
3. **A journal entry** → every question in every round already stored in it,
   with the entry resolved exactly as `make answers` resolves it.

A highlighted paragraph is ONE prompt; a numbered or bulleted block is one per
line. The `---` rule and the `## ChatGPT questions` heading are furniture and are
dropped, so selecting a whole round does the right thing. Reading rounds back out
of an entry takes **only the numbered lines** — her own writing sits above the
first round and her answers sit between the questions, and filing either as a
prompt would put her own words in the store as something an AI asked her.

Group is `prompts/<slug of where it came from>` — the conversation's own title,
or the entry's — which is the "organise by the context they came from" half;
auto-tagging supplies the theme half.

### Deduping across replies is the whole difficulty of the sweep

A reply's numbered recap repeats, in the first person, questions its own prose
asked in the second ("If YOUR body were a collaborator" → "If MY body were a
collaborator"), and later replies restate earlier ones again. So the sweep cannot
be a loop with a set.

`_covered` compares **content words only** — pronouns and auxiliaries dropped —
against the **smaller** of the two sets, at 0.8. Every part of that was forced by
a real pair:

| Pair | Must be | Why the naive version failed |
|---|---|---|
| "If **your** body were a collaborator…" / "If **my** body…" | same | differs only in pronouns; exact matching doubles every reply |
| "…to my **brain**?" / "…to my **body**?" | different | 7 of 8 words shared (0.875); a plain 0.6 overlap silently dropped one, and the word they differ by is the whole question |
| "…if I felt the discomfort?" / "…**without acting on it**?" | same | measured against the *longer* one it scores 0.71 — hence "smaller" |
| "What am I afraid of?" / "What am I afraid **will happen if I stop**?" | different | one content word is no evidence; containment would let "afraid" swallow every question about fear |

First phrasing wins: the question as first asked is the one in context. Measured
on the live conversation — 4 replies, 41 raw questions → 39 kept.

**Duplicates are expected, not adjudicated.** She will run this on a reply she
has already added. `quotes.py add-prompts` skips those silently and reports the
count, rather than running the `recapture_verdict` path that exists for a
re-grabbed book passage. One lock, one save, one Python start for the batch — a
reply asks ten to fifteen questions and that many process spawns is felt.

Tests: `Helpers\Tests\test_prompt_capture.ahk` (pure) pins the split rules,
because every row becomes a separate quote and both failure modes are silent —
one paragraph arriving as eight prompts, or eight questions arriving as one wall
of text. Both look plausible in the tooltip.


## Answers as data, the daily prompt, and linking at the source (2026-10-13)

Jamie asked for three things in one go, and they share one idea: **the questions
and her answers are data now, read back out of the entry bodies.**

### The question bank — `Scripts\journal\journal_questions.py`

Her answers were never stored as answers. She types them under each question in
the writing box: sometimes `A: …`, sometimes just paragraphs, sometimes nothing.
The body stays the source of truth (she edits it like any other entry), and the
parser reads it back into question / answer pairs **on every read**. Nothing is
stored, so nothing can drift. That's the same reasoning as the check-in answers
(`checkin_refresh`), and it's how "convert the Q&A to the check-in system" was
done: the same *derived* model, not the same text format.

Rules, measured against all 15 entries with rounds (160 questions, 13 answered):

| Shape | Read as |
|---|---|
| `A:` / `Answer:` at the start | dropped, it's furniture |
| paragraphs under a question | its answer, however many |
| a numbered line in her answer (`1. meditate`) | still her answer. A question needs the NEXT number AND a `?` |
| `### Afterward` | ends the answer, labels the questions after it (`section`) |
| text after `---` | her reply to the whole round, NOT the last question's answer |
| `CHATGPT QUESTIONS:` (hand-pasted, pre-rounds) | a round; takes the next free round number |

A question's id is `<entry id>#r<round>q<n>`. Its **importance is its entry's**
(`importance`, set by `rate: N`), so a question from a 10/10 entry is a 10/10
question. `journal.py questions [--unanswered] [--entry X] [--json]` is the
backend anything else can read.

**Answered** = her writing under it is non-empty, OR an entry written from it as
a journal prompt (below) has writing in it.

### "grab answers"

Replaces what "make answers" did (that phrase is kept and now does the same):

- **Only answered questions, each sent once.** `answers_sent` on the entry holds
  `{question key: hash of the answer}`. Saying it again sends only what's new or
  rewritten (the hash ignores whitespace). Unanswered questions are never sent.
- **The whole question and the whole answer**, `**Round 2, question 3: …**` then
  the answer. The chat may be days old, and "#3" alone would make it scroll back.
- **From the writing box, unsaved.** If the edit box is in front on an entry
  (proved the way "remind this" proves it: the box's state file + the foreground
  process), its live text is read from the Edit control. Answer, then send, with
  no save in between.
- **Into THAT conversation.** `chat_conversation_id` → focus its tab (or open
  it), and wait until the front ChatGPT window's address really shows that id
  before typing anything. Not linked and no ChatGPT in front = clipboard only.
- Staged, never submitted. `answers-to-send` / `answers-mark-sent` in journal.py;
  marked sent only after the paste lands.

### Linking the chat when it's created

`AskAIAboutEntry` now ends with `JQWatchForChatLink(id, prompt)`. A new chat has
no id until the first message is sent; the moment she presses Enter, ChatGPT
moves the tab to `chatgpt.com/c/<id>`. The watcher polls that window's address
bar by UIA (never Ctrl+L) every 1.5s for up to `journal.ask_link_wait_seconds`
(540, under the 10-minute reaper), then calls `set-chat-url`.

**A candidate is verified, not assumed.** She can switch tabs before sending, so
the conversation's first message is fetched (`ChatGptApiFetch`) and has to start
with the standing prompt. With no API, an id is accepted only if no entry owns it
yet. Rejected ids are skipped, not re-checked. `Journal/asklink` lines in
`ahk_event.log` say what it did. A second "ask an AI" replaces the watcher
(`MainfunClaim("askai")`).

**The link has to be VISIBLE, not just stored** (fixed the same evening). The
first live run linked correctly, and the log said so, but `set-chat-url` only wrote
`chat_conversation_id`, which shows nowhere. The link a reader sees (`links`)
and the `ai-conversation` form had only ever come from a captured chat's
segments, so the entry looked unlinked. `set-chat-url` now also stores
`chat_url` (which `collect_links` includes, so it survives every later write)
and ADDS `ai-conversation` to the forms, keeping a manual form like `ifs`.
The chat's TEXT still arrives only with "grab chat".

Everything later (grab questions, grab chat, grab answers) resolves by this id,
so the "which entry was this?" guessing is gone for any chat started this way.

### The automatic grab read once the reply is finished (2026-10-18)

Jamie: after "ask an AI about this", once ChatGPT has finished, do a grab read
on its own. So once the link lands, the same dispatch goes on to
`_JQAutoGrabRead(id, cid)` (JournalQuestions.ahk):

- **It waits for FINISHED, not started.** `JQWaitForFinishedReply(cid, maxMs)`
  polls `ChatGptApiFetch` every 3s and returns only when the last message is the
  assistant's, its status is `finished_successfully`, and the whole transcript is
  identical on two polls in a row. The same test `ReaderChatReplyWait` uses.
  Grabbing early stores half a reply, and append-chat (which matches turns on
  text) would later add the full one as a SECOND turn. Reusable for anything
  else that must not grab early.
- **It reads the API, not the screen.** She may switch tabs or desktops while
  it writes, and `GrabChatTranscript` reads whatever chat is in front.
  `JQApiCapture` puts the API result into GrabChatTranscript's shape.
- **Then it does exactly what "grab read" does to an entry's own chat.**
  `_JGrabChatIntoEntry(src, res, read)` (JournalCapture.ahk) was split out of
  `GrabChatToJournal` so both share it: attach the chat, add every question round
  (no prompt, see below), open the entry, open the chat in the reading room.
- `MainfunKeepAlive` keeps the reaper off the wait. Never finished =
  `Journal/autoread` "gave up" in `ahk_event.log` and a tooltip saying to "grab
  read" by hand. A second "ask an AI" still replaces it (`MainfunClaim("askai")`).
- Settings: `journal.ask_auto_read` (on) turns it off; `journal.ask_reply_wait_minutes`
  (8) is how long to wait after the link.
- Verified as a dry run, no message sent: the wait returned "" on a bogus
  conversation and the finished result in ~4s on a real finished one; the grab into
  a scratch entry attached 2 turns, added the 7 questions as round 1 and opened
  the reader on the reply. The scratch entry was then removed.

### The daily journal prompt (new tab)

[[NEW_TAB_PAGE]] has a **Journal Prompt** panel (`journal_prompt` source):

- `journal.py question-daily`: one **unanswered** question a day, a weighted draw
  (`1 + max(0, importance − 5)`, so unrated = 1 and a 10 = 6), preferring ones
  never offered before. It's the same question all day, including after she
  answers it, so the row can show `answered ✓`. State lives in
  `INIDATA\journal_prompt_daily.json` (which question each day + swaps); the draw
  is seeded by the date, so losing the file still gives the same question.
- **Click** → `OpenJournalPrompt <id>` → the writing box opens with
  `Prompt: <question>` / `From: <entry> (<date>)` at the top. The entry carries
  source `{kind: "question", question_id}`, and **saving it IS answering it**:
  there's nothing to tick. Those two header lines are not counted as the answer.
- **↻ a different question** is a panel *write* (`op: reroll`), so the panel
  redraws from the server's re-read straight away. A swapped-out question is
  remembered and not offered again that day.
- Voice: **"journal prompt"** opens today's.

Tests: `Scripts\codebase_tools\tests\test_journal_questions.py`.

### Question / answer pairs, walked with "go next" (2026-10-13)

A round used to land as a list of questions with one blank line between them,
and she had to move between them by hand. Every question now gets an answer
blank, the check-in convention:

```
1. What am I afraid of?

A: 

2. ...
```

**"go next" (`GuiGoToNextBlank`) is the navigation**, the same command she
uses in check-ins. It stops at a line ending in a bare colon, which a question
(ending in `?`) never is, so it lands right after `A: ` under the next
UNANSWERED question and skips answered ones (their line no longer ends in a
colon). It wraps at the bottom. The parser drops the `A:`; an `A:` with nothing
after it is unanswered; writing on the lines under the blank also counts.

"grab questions" opens the edit box with the caret on the new round's **first**
blank, not at the end of the entry: `EditJournalEntry(id, roundNo)` passes a
`caret_after` regex to the shared box (`SingleFieldSkipableInputGui`'s new
option; it matches against the control's CRLF text so the offset is right).

**"go next" also stops on a `Label? ` line** (`GuiIsUnansweredBlank`, GuiPrimitives.ahk).
Check-ins render a QUESTION-shaped label with no colon (`?:` reads as a typo), which had
silently made every such blank unreachable; the Weekly check-in is nothing but those. The
trailing space is the signal: an AI question line (`1. ...?`) and her own sentences ending
in `?` have none, so they never become stops. Pinned in `test_journal_checkin.ahk`.

`append-body` now trims only trailing NEWLINES, so the last blank keeps its
space. Rounds grabbed before this were given their blanks the same day by
`journal.py questions-add-blanks` (dry run unless `--commit`; additive, idempotent,
refuses any entry whose questions/answers would read back differently): 155
blanks in 16 entries, backup `journal.json.bak_before_answer_blanks_20261013`.
Re-run it any time. Pinned in
`test_journal_questions.py` (including the go-next regex).

## Special questions and numbering across the whole chat (2026-10-14)

She wanted to say "star question 3" — on the ChatGPT page or in the writing box —
and have that question kept aside to come back to, answered or not.

**The number problem came first.** Each reply used to number its questions from
1, so after two rounds there were two question 1s and "star question 1" meant
nothing. Fixed at both ends:

- **The standing prompt** now asks the AI to keep numbering across the whole
  conversation (both copies: the send body and the fallback literal, pinned
  together by `test_journal_standing_prompt.py`).
- **"grab answers"** appends one line to every message it pastes: *if you ask
  more questions, number them starting at N*. The prompt only goes out with
  the first message, and models drift back to "1." within a few replies.
  `--no-numbering-note` leaves it off.
- **The entry keeps the chat's numbers.** `numbered_recap_at` finds a recap
  starting at any number; `render_round(..., start)` writes it with those
  numbers; `parse_section` accepts any number for a section's FIRST question
  (every one after it must still be next-in-line, which is what keeps her own
  lists out). No recap → the round continues from the entry's highest number.
- `render_answers` names the round only when numbers actually repeat
  (`numbers_repeat`), since "Question 7" is unambiguous now.

**Older chats** (each round restarted at 1): `find_by_number` takes the NEWEST
round with that number. She said not to worry about these beyond that.

**Affinity, not a yes/no star.** Within the hour this became the unified
question store: see [[QUESTIONS_SYSTEM]]. "star question 7" is +1 and "unstar" is −1 on the
shared affinity scale, held in `E:\Media\catalog\question.json`. The stars on
the question's line are a MIRROR of the score (one per point, up to 5), written
by `set_stars`, which touches that one line, keeps a CRLF line's carriage return,
and round-trips byte for byte. `journal.py star-question --n N --by ±1` (or
`--qid`) either edits the stored entry under the lock (`--id`), or scores from
a text file (`--body-file/--out-file`, plus `--id`) and leaves the journal alone.

**Where the star goes** (`_JQStar`, JournalQuestions.ahk):
1. Writing box in front → its live text, via `_JQStarInBox`: the differing span
   is replaced with EM_SETSEL + EM_REPLACESEL (undoable, the draft autosave
   sees it), and her selection and scroll are restored.
2. Else the conversation's entry (`_JQAnswersEntry`: the chat URL, then the
   last-asked record). If that entry is open in a writing box BEHIND, the box
   again (`_JQBoxOpenOn`, from the box's state file), because a store write
   would be lost the moment the box saved.
3. Else the stored entry.

A writing box on a NEW, unsaved entry has no record behind it yet, so starring
there says "save the entry first".

**Phrases not used:** "special question <n>" — `special <phrase>` is the grab-store
command and would save "question 3" as a quote. "mark special <n>" — the writing
box's `mark <marking>` already has `special`.

**The list** — "★ Special questions" in the journal Miller (root row, or
`OpenSpecialQuestions` / "open special questions"). `journal.py
special-questions` (TSV) lists every question at affinity 1+, highest first,
with its stars. Each question previews with everything she has written to it.
Drill in for:
- **Write about it again** (`OpenJournalPrompt`): a NEW entry linked to the
  question, which she chose over writing under the original. The earlier answers
  are quoted as `> ` lines, which `prompt_answer` skips.
- **Open the entry it came from.**
- **Star it again** / **Unstar it** (`--qid`, `via: list`).

**The daily prompt** weights by affinity (× 1 + affinity × `questions.affinity_weight`)
and never offers a buried question. A starred question stays in the pool once
answered, is never used up, and rests `questions.rest_days` after each showing.
On the new tab it shows its stars and always opens to be answered again. Details
in [[QUESTIONS_SYSTEM]].

Tests: `Scripts\codebase_tools\tests\test_journal_special_questions.py`.

## Rounds she never grabbed + reading a chat (2026-10-15)

**The gap:** "grab questions" takes only the NEWEST reply. Send a follow-up before saying it and the earlier reply's numbered questions are buried in the transcript, never a round.

- `chat_questions.py --pending` (`pending_rounds` / `render_pending`) checks EVERY reply. A reply counts as grabbed when half its numbered questions are already in the entry's rounds. Only numbered lists count, so a loose "Would you like me to...?" is never offered. Rounds get unique numbers in the entry (a reply nobody grabbed usually restarts at 1, which would make "star 1" ambiguous).
- **"grab chat" / "grab read" / "grab questions" ADD them, no prompt** (changed 2026-10-18; Jamie: "stop asking about questions, just add them"). It was a confirm modal, and the answer was always yes. They are written before the Miller opens, so it lands on an entry that already has them, and the tooltip says how many ("1 round of questions added (7)", folded into the grab's own tooltip via `JQPendingLabel` and `JQAddPendingRounds(id, pend, quiet)`). Code: `JQPendingRounds` / `JQAddPendingRounds` in `JournalQuestions.ahk`; `JQAskPendingRounds` is gone.
- **Miller → entry → "❓ Add questions you never grabbed"** (`JQAddMissedRounds`, `--pending --entry <id>`) does it from the SAVED chat, no ChatGPT tab needed.
- **Adding a round is all there is to "saving" a question.** The question store is reconciled from the bodies. Before this it only synced when something READ it, so new rounds had no record, tags or special-list presence until then. Now `journal.py` spawns a detached `questions.py sync-journal` after every write (`_sync_question_store`, skipped under tests).

**Reading a chat in the reader:** `grab read` / `grab chat read` (`GrabChatRead`) = grab chat, then open it in the reading room. Miller -> entry -> "📖 Read the chat in the reader" (`OpenJournalChatInReader`). Reader type `chat`, spec `chat:<entry id>`, URL `/chats/...`, resolver `reader_collection.resolve_chat`.
- **Items:** HER ENTRY first (title + body, question rounds cut off), never the first prompt (the standing prompt plus the pasted entry). Then each follow-up she sent and each reply, every one its own item so it starts on a fresh page.
- **Opens on page 1 of the NEWEST reply** (`index=9999` = targeted, which the server clamps), never a remembered page part-way through.
- **Rich pages:** `chat_render` turns markdown into real HTML (strong/em, h2/h3/h4, ul/ol, blockquote, pre, rules) with data-b on every text block. `_md_inline` returns html AND plain together, so each block's text is exactly the element's textContent, and highlights, grabs and numbering line up (pinned in `test_reader_chat.py`). ChatGPT's layout widgets (`<box>`/`<row>`/`<icon>`) are dropped, and `<divider>` becomes a rule. A line that is all bold becomes an h4. CSS is `section.chat` in read_server.py, sized in em off the section so headings are LARGER than the text (the generic `h2.sec` came out smaller).
- **Grab works** (numbers, mouse, `special N to M`). The quote is filed like a chat-tab "grab quote": source kind `ai`, creator = the model, title = the entry. The chat url goes in as `--context-file`, which links the quote to the journal entry (`captured.journal_ids`). The highlight is painted back via `chat_highlights`.
- **Up favourites:** it saves the current selection into the favourite `special` store, or else bumps the newest quote grabbed on that page (`last_quote`).

**Next (not built):** replying from the reader instead of jumping back to ChatGPT, e.g. sending through the ChatGPT backend `ChatGrab` already reaches.
