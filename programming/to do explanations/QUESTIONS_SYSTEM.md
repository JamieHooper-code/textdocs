---
tags: [programming, journal, questions, media-system, affinity, ahk, caster]
created: 2026-10-14
related: ["[[JOURNAL_QUESTION_LOOP]]", "[[JOURNAL_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[MEDIA_SYSTEM]]", "[[ART_SYSTEM]]", "[[PEOPLE_SYSTEM]]"]
---

# The question store — one record per question

Every question in the system gets a record with an id that never changes,
affinity on the same scale as the rest of the library, tags, and links.
It started with the questions ChatGPT asks about journal entries. Check-in
questions, programming questions and others are designed to land here too.

## Why a store

Until 2026-10-14 a question existed only as a numbered line inside a journal
entry, re-read on every call and identified by where it sat (`r2q7`). That can
carry a yes/no star and nothing else. Jamie wanted **affinity** (star it twice,
it's a 2, the way art and quotes work) and said questions will be used for more
than this, in more than one place. So they need their own records. The yes/no
star lasted about an hour before this replaced it.

## What is stored, and what is not

| Thing | Lives in | Why |
|---|---|---|
| The question's wording | its source (the entry body) | she edits it there |
| Her answers | the entry body, and journal-prompt entries | she writes them there |
| Stable id, origin, aliases | **the store** | nothing else can hold identity |
| Affinity + dated log | **the store** | the shared scale |
| Tags (`manual` / `auto` / `affinity`) | **the store** | |
| Links (its copy in the quote store) | **the store** | |
| Earlier wordings | **the store** (`text_history`) | an edit must not erase what it was |

`sync_journal` reconciles the store with every entry body. It runs on every
question read (`journal.py question_bank`), costs about a second, and **never
writes a journal body**. The stars on a question's line are written only by
`star-question`, as a mirror of the score.

## A record

```json
{"id": "question:f00c5a1792f1",
 "kind": "ai",
 "text": "You recognize that the Moria part is afraid...",
 "origin": {"kind": "journal_round", "entry_id": "journal:2026-10-13-...",
            "key": "r1q1", "round": 1, "n": 1, "chat_id": "6acec7b7-..."},
 "aliases": ["journal:2026-10-13-...#r1q1"],
 "affinity": 1,
 "affinity_log": [{"at": "2026-10-14T14:04:47+00:00", "by": 1, "score": 1, "via": "typed"}],
 "tags": [{"tag": "fear", "src": "auto"}, {"tag": "favorite", "src": "affinity"}],
 "links": {"quote_ids": ["quote:..."]},
 "text_history": [],
 "status": "active",
 "created": "...", "seen": "..."}
```

- **Kinds** are data, in `INIDATA/question_kinds.json`: `ai` (live), plus `checkin`,
  `programming` and `prompt` (planned). A new kind is a row there plus a sync
  for its source.
- **`aliases`** keep every legacy id a question has answered to. Journal-prompt
  entries made before the store point at `"<entry>#r2q7"`, and the daily-prompt
  state holds those too; both still resolve.
- **`status: gone`**: the question left its source (the round was deleted, or the
  entry was). The record keeps its affinity and tags, and if the question comes
  back it picks up where it left off. Deleting a line of text must never
  silently throw away what she gave it.

## Matching across edits

A journal question is found again by `(entry, key)` first. If its slot now holds
different wording (a re-grabbed round), or the round was renumbered, it is found
by wording within the same entry (content-word overlap ≥ 0.8). Only then is it new.
Pinned in `test_journal_special_questions.py`: renumbered, reworded, removed and
restored, and a typed star counted once.

## Affinity

The SAME scale as art, quotes, poems and comics: `Scripts/MediaCatalog/affinity.py`,
never a copy. "star question 7" is +1 and "unstar question 7" is −1. The score
goes through zero into **buried** (−1 and below). Band tags are derived:
favorite ≥ 1, loved ≥ 3, essential ≥ 5, buried ≤ −1. Every change is logged with
`via` (`voice`, `list`, `typed`, `cli`).

**The stars in the entry are a mirror**: one ★ per point, capped at five,
none at zero or below. The store holds the number. A star typed by hand that the
store hasn't counted is read as a bump on the next sync (`via: typed`), which is
how the yes/no Moria star moved over. Deleting stars by hand does **not**
lower the score (a stale box could do that by accident); say "unstar".

## Where it shows up

- **Voice**: `star question <n>` / `star <n>` / `mark question <n>`, and
  `unstar question <n>` / `unstar <n>`. Said in the entry's writing box or on its
  chat. Routing is in [[JOURNAL_QUESTION_LOOP]].
- **The journal's "★ Special questions" list** (`open special questions`): every
  question at 1+, highest first. Each one previews everything she has written to
  it. Drill in for: write about it again (a new linked entry, earlier answers
  quoted), open the source entry, star again, unstar.
- **The daily journal prompt** (new tab): the pool is unanswered questions plus
  anything at 1+, answered or not, minus anything buried. The weight is the
  entry's importance × (1 + affinity × `questions.affinity_weight`, default 1.5).
  A starred question rests `questions.rest_days` (7) after each showing and is
  never used up. On the new tab it shows its stars and always opens to be
  answered again.

## Answering a question later = a new entry, linked both ways

Jamie's rule (2026-10-14): she writes inside the ORIGINAL entry only while she
is still working on it (the "grab questions" round, answered in its writing
box). Coming back to a question after the fact ALWAYS makes a new journal entry,
dated the day she writes it, so a question answered three times has three
answers on three real dates. Every revisit path already does this:
`OpenJournalPrompt` from the special list, the daily prompt, and the new tab.
The new entry carries `source: {kind: "question", question_id}`.

The two ends link without either body being touched (`journal.question_followups`,
rendered on the fly in `render_entry`):

- **The original** gets `> ↳ answered again: <date — title>` under each such
  question. The answer itself does not appear there.
- **The answer** opens with `> ↩ answering a question from <date — title>`.

In the generated journal document these are `[[#heading]]` links when the other
entry is in the same document, and plain date + title when it isn't (a filtered
export). The single-entry view uses plain text. The link resolves through the
store, so it follows a renumbered question. Pinned in
`test_journal_special_questions.py`.

## Tags

The journal's tag vocabulary (`media_tags.json`, `applies_to: journal`). The local
model only, through the `question_tag` task in `Scripts/local_llm/tasks.json`;
never the paid gateway. New questions are tagged by a detached, windowless
`tag-auto --untagged` spawned by the sync that created them. Proposed new tags go
to `INIDATA/question_tag_review.json`, never straight into the vocabulary.
`questions.py tag <id> --add/--remove` is the manual path (`src: manual`).
Approving proposals is two steps: `quotes.py vocab-add <tag> --applies journal,quote`
(`--applies` REPLACES the list, so pass every type an existing tag already has),
then `questions.py review-accept <tag> ...`, which applies each one to the
questions that proposed it and clears it from the queue. `questions.py review`
lists what's waiting. First batch approved 2026-10-14: trust, body, uncertainty,
pleasure, healing, effort, control. A tag
writer only rewrites its own `src`, the same rule as everywhere else.

## Links

When "add prompt" has saved a question to the quote store as a prompt, the sync
links them (`links.quote_ids`, matched by entry + wording). The quote store
is not written; it owns its own records.

## Tests never touch the real store

`journal.question_bank` syncs on every call, so a test that builds a fake
journal would reconcile the REAL `question.json` against it and mark every real
question gone. That happened once while this was being built: four fixture
records got in, and a later real sync revived everything with nothing lost.
`Scripts/codebase_tools/tests/conftest.py` now points every test at a temp store
(`QUESTION_STORE_PATH`) and switches off auto-tagging. The path is read at call
time, so the fixture works even after import. Any new store that syncs on read
needs the same treatment.

## Adding check-in questions (next)

Check-in questions live in `INIDATA/CheckinTemplates.json` with stable keys per
type and phase. The shape that fits:

- `kind: "checkin"`, `origin: {"kind": "checkin_template", "type": ..., "phase": ..., "key": ...}`.
- A `sync_checkins()` beside `sync_journal()`, matched on `(type, phase, key)`.
- Answers stay where check-ins already keep them (journal.py derives them).

Nothing about the store has to change for this; that was the point of building it first.

## Files

| Layer | File |
|---|---|
| Store + CLI | `Scripts/questions/questions.py` → `E:\Media\catalog\question.json` |
| Kinds | `INIDATA/question_kinds.json` |
| Journal bridge | `Scripts/journal/journal.py`: `question_bank`, `star-question`, `special-questions`, `question-daily` |
| Body parsing + star mirror | `Scripts/journal/journal_questions.py`: `stars`, `set_stars`, `stars_for` |
| AHK | `Helpers/JournalQuestions.ahk` (`StarQuestion`, `_JQStarInBox`), `Helpers/JournalMenu.ahk` (the list) |
| Settings | `questions.affinity_weight`, `questions.rest_days` |
| Tests | `Scripts/codebase_tools/tests/test_journal_special_questions.py` |
