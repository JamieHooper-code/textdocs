---
tags: [programming, writing, new-tab, local-viewer, journal, poetry, stories, prompts]
---

# Writing Page

A page of things to write, at **http://127.0.0.1:8293/write**: the ✎ Writing
link in the hub header on every local page, or `MAINFUN.bat OpenWritingPage`.
Built 2026-10-19.

## The decision: a second board on the new tab server

Jamie: a page "within a new tab page", reached from the header, not shown on
the new tab itself. It is literally the new tab's board (same `index.html`,
same `app.js`, same panels, drag, collapse, ↻) with a different panel list. The
server serves `/write` with `write_panels.js` in place of `panels.js` and sets
`window.PAGE_NS = "write:"`, so this page's layout and settings live under their
own keys in `INIDATA/newtab/storage.json` and a drag on one board never moves
the other.

This is the first page that shares a server with another page. Jamie, the same
day: "I really don't care about" one crash taking several pages down, so
separate servers are a default now, not a rule ([[LOCAL_APP_HUB]] § "Why not one
server"). `apps.json` rows may share a port and differ by `path`; kit.js
highlights the row whose path best matches the page.

## Panels

| Panel | Reads | A click |
|---|---|---|
| Today's Spark | one web prompt; family rotates by day | the right box for its family |
| Journal Questions | 5 unanswered AI questions (`journal.question_bank`) | `OpenJournalPrompt` |
| Journal Prompts | 4 web prompts, one per source | journal box, source kind `prompt` |
| Write From Your Day | today's most-played songs (scrobbles), the current book, what she logged today (not the programming log) | a card |
| A Letter | people she talks to most, longest since a letter first; parts too | `JournalLetter` |
| Poetry Prompts | 4 web prompts of DIFFERENT styles + 1 object-writing word | new-poem box / `StartPracticeOn object` |
| Constraint Cards | a form + a rule (`INIDATA/writing/constraint_cards.json`) | new-poem box |
| Collisions | adjective+noun, noun+verb, noun+noun from the practice word store | `StartPracticeOn` |
| Line Seeds | a line from her own unfinished poems or scraps | new-poem box |
| Look, Then Write | one work from the art library (favourites likelier) | opens the art, then the new-poem box |
| Your Poems | in progress · needs editing · nearly there · from the archive | `EditPoem` |
| A Poem to Read | the poem-of-the-day pair | `ReadPoemOfTheDay` |
| On This Day | journal entries + poems from this date in earlier years (this week if none) | entry / `EditPoem` |
| From Your Books | a book highlight from the quote store | journal box |
| Story Prompts | 4 web story prompts | new-story box |
| Your Stories | stories in progress + idea sheets | `EditStory` / `AddStoryFromIdea` |

Each row of the drawn panels has a ↻ (another one today). Web-prompt rows also
have ▲ / ▼ (the shared affinity scale; ▼ to −1 buries the prompt and swaps it
off the page) and ↗ (the post it came from). The streak / writing log was
proposed and declined.

## Object writing is its own style

The 4,987 words in the practice store (`prompt.json`, [[PRACTICES_SYSTEM]]) are
ONE kind of prompt: a single word for Pattison's ten-minute sensory writing.
Jamie: "it is one very specific thing, and it should be considered as such as we
add more poetry prompts of different styles." So the written prompts got their
own store, and object writing shows up as the `object_writing` style: one row
among the poetry prompts, drawn from the word store and never copied.

## The prompt store

`Scripts/writing/writing_prompts.py` → `E:\Media\catalog\writing_prompt.json`.
One record per prompt, id = hash of its words: `family` (journal / poetry /
story), `style`, `moods`, `source` + `source_label` + `url` + `author` +
`published`, `context` (the poem or book it is built on), `affinity`, `used`.
Families, styles and moods are DATA in `INIDATA/writing/prompt_styles.json`.
`classify` labels style + moods with the local model (task
`writing_prompt_classify`, ~1.4 s a prompt).

## Sources

`INIDATA/writing/prompt_sources.json`, one row each; readers by PLATFORM in
`Scripts/writing/prompt_sources.py`:

| Platform | Sources | How |
|---|---|---|
| `pw` | Poets & Writers "The Time Is Now" | list pages carry the whole prompt + type + date; ~1,800 back to 2011 |
| `reedsy` | Reedsy weekly contest prompts, 17 mood/genre categories | `<h3 class="text-prompt">`; the category becomes a mood hint |
| `wordpress` | NaPoWriMo (a prompt a day every April since 2013) | REST API; model lifts the prompt out |
| `substack` | Sheetghost, Poetry Unbound, Poetry Outdoors, 100 Poems, Art of Flash Fiction, Prompted, The Writer's Den, Writers' Hour, The Clearing, The Isolation Journals, Daily Journal Prompt, Pen Pals, Earth Poet | `/api/v1/archive` + `/api/v1/posts/<slug>`, FREE posts only, 4 s between requests (Substack answers 429 to a burst); model lifts the prompt out |

**The model may not write prompts.** Task `writing_prompt_extract` is told to
copy the author's words. Two checks back that up, because the first live run
broke both rules: `in_authors_words` (80% of the prompt's words must be in the
post; it had invented "Write a poem that explores the concept of curiosity")
and `reads_as_prompt` (a question or an instruction; it had returned the
newsletter's quote of the day). Every prompt keeps its url.

Reading is resumable (`INIDATA/writing/source_state.json` lists the posts read).
`py prompt_sources.py test <id>` tries a source without writing;
`py prompt_sources.py refresh` = read everything new, then classify.

**Weekly task `WritingPromptsRefresh`** (registered 2026-10-19 via `schtasks
/XML`): Sundays 4:00 AM, `pythonw prompt_sources.py refresh --max-minutes 120`,
4-hour cap, runs late if the PC was off. Under pythonw there is no stdout, so
the script appends its output to `INIDATA\writing\refresh.log`. A run from Git
Bash does NOT exercise that path (pythonw inherits the shell's pipes); launch
it detached to test.

## The day's draw

`Scripts/writing/writing_desk.py` builds every panel. A drawn panel keeps the
same rows all day (`INIDATA/writing/today.json`), as snapshots, so the slow
reads (9,600 art sidecars, every poem) happen once a day, not per page load.
`shown` keeps an offered row away for 45 days. An empty draw is never kept: on
Windows a read landing while a writer's `os.replace` swaps the file fails, and
`load_media_type` turns that into an empty store (seen live 2026-10-19).

## Clicks

Rows that an existing store already knows how to open keep that action. The
rest are CARDS: `WriteFromCard <panel> <key>` (`Helpers/WritingDesk.ahk`) hops
to the GuiHost, asks `writing_desk.py card` for the text, then opens:

- journal → `AddJournalEntryForSource` with source kind **`prompt`**
  (`prompt_id`, title = the ask, header = the prompt + where it is from). New in
  `journal.py` `SOURCE_FIELDS` / `SUBJECT_KINDS`, and in `_JSourceArgs`.
- poetry → `AddPoem(prompt)`; story → `AddStory(prompt)` (both gained an optional
  prompt shown above the field).

Only the panel and key cross the process boundary: a card's text carries quotes
and line breaks that a command line mangles. Keys are cmd-safe hashes for the
same reason.

## Traps

- Edit `newtab_server.py` → `py Scripts/newtab/newtab_server.py restart`. Edit
  `writing_desk.py` / `writing_prompts.py` → nothing to restart (run per call).
- Delete `INIDATA/writing/today.json` to redraw everything today.
- Tests: `Scripts/codebase_tools/tests/test_writing_page.py`.

## Later

- Prompts from books she reads (the quote store's `prompt` entry type already
  holds 15) as their own source.
- A voice command ("open writing").
