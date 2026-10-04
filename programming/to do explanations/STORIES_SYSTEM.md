---
tags: [programming, writing, stories, fiction, media-system, reader, ahk, caster, design]
created: 2026-10-04
status: built
related: ["[[POETRY_SYSTEM]]", "[[QUOTES_SYSTEM]]", "[[JOURNAL_SYSTEM]]", "[[PRACTICES_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[SETTINGS_SYSTEM]]"]
---

# Stories System

Jamie's OWN short stories, plus the **idea sheets** they grow from. The poetry
system's twin ([[POETRY_SYSTEM]]): same writing box, same Miller shape, same
rank + affinity, same reader. Other people's short stories are NOT here: they
stay in the quote store (entry type `story`, favourited with Up on a story page
in a book), the way other people's poems sit beside hers in the poetry setup.

## What exists (built 2026-10-04)

| Piece | Where | Notes |
|---|---|---|
| Store engine + CLI | `Scripts\stories\stories.py` | `E:\Media\catalog\story.json`. Imports every generic record rule (tags w/ provenance, people, rank, affinity, trash) from `poems.py` -- one implementation, no drift |
| History (old versions) | `E:\Media\catalog\story_history\<slug>.jsonl` | one line per earlier version; the record only keeps `revision_count` |
| Writing box | `Helpers\StoriesFunctions.ahk` | `AddStory`, `AddStoryIdea`, `AddStoryFromIdea`, `EditStory` -- WritingBox "page", own process, draft-key crash-proofing |
| Miller | `Helpers\StoriesMenu.ahk` + `Scripts\StoriesViewer.ahk` | `OpenStories`, `OpenStoriesAtItem`; `_StoriesAsNode` mounted in `open media` |
| Reader | `Scripts\reader\reader_collection.py` (type `mystory`, URL `/my-stories/`) | openers in `Helpers\ReaderFunctions.ahk`: `OpenMyStoryReader`, `OpenMyStories`, `OpenProgressStories`, `OpenStoryIdeas`, `OpenRandomStory` |
| Markers | `Scripts\journal\markup.py` `STORY_KEYS` | `{draft}` / `{stage X}` added; `{in progress}` / `{finished}` reused |
| Settings | `INIDATA\Settings\schema\stories.json` | `stories.list_order`, `stories.preview_meta` |
| Tests | `Scripts\codebase_tools\tests\test_stories_store.py` | 18 |

## Decisions (Jamie, 2026-10-04)

Reply: `9-10- 12 <keep rankings + affinity> 22- 0.00`.

- **Own store** `story.json`, not a kind in poem.json (every poem filter would have to keep filtering stories out).
- **Two kinds:** `story` and `idea` (idea sheet). One store, so a misfile is a field change.
- **Stage** draft -> in progress -> finished, on the ONE record (never copied forward), each change stamped in `stage_history`. Draft = started but parked; in progress = actively working on it (Claude's call on the distinction, per `00`).
- **Ideas grow into stories**, both directions linked, many-to-many (`grew_from` / `grew_into` are lists). The idea sheet is kept.
- **Revisions beside the store** -- the scaling risk: a long story saved often would bloat story.json, which the Miller reads on every level.
- **Rank 1-10 AND affinity** (item 12, overriding the "favourites only" proposal).
- **No word-count display or per-day words** (9-), **no word goals** (10-). `words` is still a derived field on the record (cheap; nothing shows it).
- **Scene breaks:** a lone `***`, `* * *`, `#`, `---`, `~~~` line renders as a rule in the reader.
- **No import** (22-): the store started empty.
- **"open stories" now opens the Miller.** The old shelf is its "Other people's stories" row (and still the Reading Miller's "Favourite stories"). `OpenStoryReader` is kept as the function behind both.
- **Journal-a-story deferred**, like journal-a-poem.

## The writing box

One text: the story plus brace markers anywhere --
`{title The Lighthouse}` `{rank 8}` `{draft}` / `{in progress}` / `{finished}`
`{tags grief, Charli}` `{favorite}`. Editing opens with the current values as
markers; deleting one clears it -- EXCEPT the stage, which a story always has, so
no stage marker keeps the stage. "Start a story from this idea" opens the box
seeded with the idea's title, tags and text plus `{draft}`; saving links the two.

## Voice

| Phrase | Does |
|---|---|
| `open stories` | `OpenStories` (the Miller) -- repointed from `OpenStoryReader` |
| `story new` | `AddStory` |
| `story idea` | `AddStoryIdea` |
| `story progress` | `OpenProgressStories` (reader, last edited first) |
| `story read` | `OpenMyStories` |
| `story random` | `OpenRandomStory` |
| `edit this` / `show this` (reading room) | `ReaderEditItem` -> `EditStory` for a `story:` id |

All in the generic store (`INIDATA\VoiceChoices\generic_commands.json`).

## Traps

- The reader server holds its code in memory: `read_server.py restart` after any edit
  to reader_collection.py / read_server.py / stories.py. (From a shell, `restart`
  spawns the server as a child that keeps the pipe open -- launch it with
  `pythonw ... serve` via `Start-Process` instead.)
- Grabbing is refused in `mystory` like `poetry` -- it would file her own story
  into the quote store as a stranger's.
- Viewer TSV columns are read POSITIONALLY by StoriesMenu.ahk: append, never insert.
