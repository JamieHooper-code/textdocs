---
tags: [programming, journal, caster, ahk, chatgpt, writing, media-system]
created: 2026-09-03
related: ["[[JOURNAL_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[EXPORT_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[JOURNAL_SOURCES]]"]
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

## Voice commands

| Phrase | Does |
|---|---|
| `grab questions` | newest reply's questions → clipboard + the entry it came from + the writing box |
| `grab questions here` | same, into the NEWEST entry, no matching — the escape hatch |
| `see questions` | clipboard only, nothing written |
| `make answers` | newest round + her answers → back into the conversation, staged not sent |
| `add prompt` | the selection / the reply's questions / an entry's rounds → the quote store as prompts |

All four live in `journal_commands.py` with the `grab chat` family — same engine,
same store, same global scope. `make answers` rather than `send answers` because
`send <choice>` is three separate Choice rules and a literal sharing that prefix
is asking for trouble.

Nothing is ever submitted. Same rule as `AskAIAboutEntry`: the message is staged
and she presses Enter, so a mis-grab is one Ctrl+A away from being discarded
rather than something already sent.

## Settings

- `journal.ask_recall_hours` (12) — how long after "ask an AI about this" a
  grabbed round still assumes it belongs to that entry.
- `journal.questions_open_editor` (on) — land in the writing box after grabbing.

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
