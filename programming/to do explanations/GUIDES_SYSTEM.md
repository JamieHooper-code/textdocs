---
tags: [programming, design-doc, guides, journal, people, reader, mental-health, ahk, caster]
created: 2026-10-18
status: v1 built 2026-10-18 — "offered at the right moment" (check-in, new tab) is v2
related: ["[[JOURNAL_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[PEOPLE_SYSTEM]]", "[[REMINDERS_SYSTEM]]", "[[STORIES_SYSTEM]]", "[[HEALTH_SYSTEM]]", "[[SETTINGS_SYSTEM]]"]
---

# Guides

Short pieces Jamie keeps for particular **states**: "a plan for intense or
activated moments", "what to do when I feel aimless". Steps, a list of things
to try, a few paragraphs about not catastrophising. Before this they landed in
some random document and vanished, which matters most exactly when she needs
them. So the design is mostly about **getting back to them**, not storing them.

## What a guide is, and why it is its own store

The journal is things she wrote once, about a moment. Quotes are other people's
fragments. Reminders are ordered by attention. A guide is a living reference she
**revises** and comes back to, reached by **how she feels**. So it gets its own
store, a sibling of the others: `E:\Media\catalog\guide.json`, written ONLY by
`Scripts\guides\guides.py` (same atomic write + lock helpers as the rest of the
catalog). She chose a JSON store in the AutoHotkey system over vault markdown
files: "that's where everything gets saved", and she already edits every other
piece of her writing in the writing box.

## The text is the truth

A guide is one markdown text. `## Heading` lines split it into sections, and each
section's kind is READ OFF its lines, never stored:

| kind | what makes it | used for |
|---|---|---|
| steps | mostly numbered lines | the shape of a plan |
| list | mostly `- ` bullets | **"pick one for me"** rolls a random line from these |
| prose | anything else | the paragraphs |

So editing a guide is just editing text, and there is no second structure to keep
in step. The writing box opens with three header lines above the guide:
`title:`, `states:`, `say:` (the word for "guide <word>"). Old text goes to
`E:\Media\catalog\guide_history\<slug>.jsonl` before every change (the
[[STORIES_SYSTEM]] pattern), so revising never loses the earlier wording.

## States are data

`INIDATA\guide_states.json`: one row per state (key, label, extra spoken words,
group `inside` / `with people`). Order = picker order. A guide carries a list of
state keys; one guide can serve several. **A state with no guide is never
offered**, so the registry can hold every state worth exploring without the
picker showing empty ones. Seeded 2026-10-18 with 20:

- inside: activated, aimless, overwhelmed, low, numb / shut down, anxious / dread,
  ashamed / self-critical, stuck / frozen, body hurting, grief, good day
- with people: lonely / disconnected, after a conflict, before a hard
  conversation, waiting on a reply, rejected / left out, resentful,
  people-pleasing / can't say no, socially drained, missing someone

## Ways in

| | how |
|---|---|
| By name or feeling | voice **"guide <guide_name>"**: a guide's own word, its title, or a state's word ("guide wired" opens the activated guide). A state with one guide opens it; several open together. |
| By the menu | **"open guides"** / **"I feel"**: the states that have a guide on top (Enter opens), then All guides, Reach-out people, Add. |
| Offered at the right moment | **v2, not built.** E.g. the health check-in sees short sleep + racing thoughts and offers the activated guide; a new-tab row. Easy to make naggy, so it waits until the two above have proven themselves. Use the `gentle-lines` skill for any wording. |

`<guide_name>` is generated: `guides.py` rewrites
`INIDATA\VoiceChoices\guide_names.json` on every change and, **only when the list
changed**, bumps the reload marker in `caster\rules\guide_commands.py`, because
Caster re-reads the JSON only when the rule file's content changes.

## Reading

Reader type `guide` (`reader_collection.resolve_guides`, URL `/guides/`), specs
`guides:all` and `guides:state=<key>`. Each guide is a `rich` item rendered from
its markdown by the same renderer a saved chat uses (`chat_render`), flow paged.
A whole page of text is fine — she turned down one-step-per-page. A single guide
opens via `/api/open?type=guide&id=<id>` among every guide, so Left/Right walks on.
"Show this" (`ReaderEditItem`) on a `guide:` id opens `EditGuide`.

## Reach out

Every guide ends with ONE shared block, built live at read time: the people
tagged **`emergency`** in the person store ([[PEOPLE_SYSTEM]]), each with
`say "text <name>"` when they have a phone. Tagging someone is the whole edit, so
the block never needs maintaining. Toggle it from the person's page in
**"open people"** (row "Emergency person") or from the guides menu (Reach-out
people → Add or remove someone). Parts are never listed. She asked for a block
about reaching out to a friend and **no crisis-line / calling content**. Don't
add any. Settings `guides.show_reach_out`, `guides.reach_out_heading`.

## How it went (learning from it, lightly)

Not stored here. **"how did it go"** (or "journal this" with a guide open, or the
menu row) opens the journal writing box with source kind **`guide`**, numbered
per guide ("Guide: When I'm activated #3: "), and a header asking for a 0-10
("rate 7") plus a line or two. Esc skips it. It goes through the existing
[[JOURNAL_SOURCES]] machinery: `guide` is in `SEQ_KINDS` and `SUBJECT_KINDS`,
`guide_id` is a `SOURCE_FIELDS` entry, and it is in `_JSourceArgs`'s map (a field
missing from that map is dropped in silence — the known trap).

Opens are stamped on the guide (`open_count`, `last_opened`, `last_opened_via`),
so the menu can show "opened 3 times".

## Making guides: "send guide"

Guides are built WITH an AI, iterating. The instructions for that are themselves
a guide, **`guide:making_a_guide`** ("Making a guide", no states, so it is in
All guides but never in the "I feel" list): what a guide is for (processing vs
doing, about half check-in and half a way forward), gentle parts check-ins that
never feel like a chore or a crowd to placate, the shape rules (the "pick one"
bullet rule), the states, and how a round works (numbered questions PLUS ideas
and sample wording, a full draft early, the whole guide again every revision).

**Audience sections:** a heading starting **"For ChatGPT"** goes only to ChatGPT
(the one-code-block-every-time rule); **"For Claude"** only to Claude (the store,
save-box, states file, how to check and open it). The reader shows both.

**The send:** send `guide` (INIDATA\ClaudePrompts\guide.txt, contexts `*`,
auto_send off so she can add "I want a guide about eating") has the body
`{guide:making_a_guide}`. `_SendExpandPlaceholders` (ClaudeFunctions.ahk) expands
any `{guide:<id or say word>}` through `guides.py send-text --for chatgpt|claude`,
chosen by `IsForegroundContext("chatgpt")`, so the send is always the guide's
CURRENT text. Any other send can embed a guide the same way.

## Commands

| voice | AHK |
|---|---|
| guide <guide_name> | `OpenGuide(words*)` |
| open guides / I feel | `OpenGuides` |
| pick one for me | `GuidePickOne([id])` (no id = the guide open in the reader) |
| how did it go | `GuideHowDidItGo([id])` |
| add guide / add guide clipboard | `AddGuide` / `AddGuideFromClipboard` |
| edit guide | `EditGuide([id])` |
| — | `OpenGuidesForState(key)`, `OpenAllGuides`, `ToggleReachOutPerson(personId)` |

## Files

- Engine `Scripts\guides\guides.py` (CLI in its docstring); states
  `INIDATA\guide_states.json`; tests `Scripts\codebase_tools\tests\test_guides_store.py`.
- AHK `Helpers\GuidesFunctions.ahk` (openers, writing box, pick, journal) +
  `Helpers\GuidesMenu.ahk` / `Scripts\GuidesViewer.ahk` (the Miller; actions go
  via MAINFUN).
- Reader: `reader_collection.py` § GUIDES. **Stale-code trap:** restart the read
  server after editing guides.py or the resolver.
- Voice `caster\rules\guide_commands.py`.
- Touched elsewhere: `journal.py` (guide source kind), `JournalCapture.ahk`
  (`_JSourceArgs`), `JournalSources.ahk` (`JournalGuideSource`, reader branch),
  `PeopleMenu.ahk` (Emergency person row), `ReaderFunctions.ahk` (edit dispatch).

## Seeded

`guide:when_i_m_activated` — her four-step plan, kept word for word, plus a
"Somewhere for it to go" list (the five options from step 2 as bullets) so
"pick one for me" has something to roll. An "aimless" guide is the next one to
write together.
