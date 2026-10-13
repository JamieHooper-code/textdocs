---
tags: [programming, poetry, writing, media-system, reader, ahk, caster, tagging, design]
created: 2026-09-14
status: design-agreed-not-built
related: ["[[QUOTES_SYSTEM]]", "[[STORIES_SYSTEM]]", "[[JOURNAL_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[READING_ROOM_BOOKS]]", "[[MEDIA_SYSTEM]]", "[[PEOPLE_SYSTEM]]", "[[SETTINGS_SYSTEM]]", "[[POEM_PRINT_AND_DESIGN]]"]
---

# Poetry System

Jamie's own poems (pen name **Pollen**) as a first-class store, plus **scraps**
(poem ideas, loose lines, images, names). The third parallel text system after
[[QUOTES_SYSTEM]] and [[JOURNAL_SYSTEM]].

**Status (2026-09-14, night): steps 1-4 BUILT, poems IMPORTED.** 464 poems and 508
scraps are in `E:\Media\catalog\poem.json`, imported on the importer's guesses at
Jamie's call (the review list still works, and now changes the store directly). The
reader has a `poetry` type, "show this" edits a poem, new poems and scraps can be
written straight into the store, and the `open poems` Miller is done. Not yet: voice
phrases (she picks from options), the poem-poster move, journal-a-poem, reader-wide
"tag X", Doc sync (v2).

**Doc sync + GitHub backup (2026-10-10): BUILT** -- see § "Doc sync" and § "Backup
to GitHub" below. The Doc is now reached through the Google APIs, both ways.

**Print / Instagram / poster (2026-10-05):** the poem-poster pipeline is being
replaced -- see [[POEM_PRINT_AND_DESIGN]]. Print is built (Miller rows "Print
preview" / "Print" on every poem).

### What exists

| Piece | Where | Notes |
|---|---|---|
| Store engine + CLI | `Scripts\poems\poems.py` | `E:\Media\catalog\poem.json`. Guarantees in its docstring. `viewer` = the Miller's TSV feed; `doc-format`; `refresh` re-derives after a rule change |
| Importer | `Scripts\poems\poem_import.py` | `verify` / `import [--review] [--proposal] [--commit]` / `decide` (also changes the store after import) / `unit` |
| Header codes | `INIDATA\poem_codes.json` | read by the importer; a newly-learned code is one edit here |
| Review Miller | `Helpers\PoemImportReviewMenu.ahk` + `Scripts\PoemImportReviewViewer.ahk` | `OpenPoemImportReview`; also reachable from the poems Miller |
| Review decisions | `INIDATA\poem_import_decisions.json` | keyed by a hash of the unit's opening text, so they survive a re-parse |
| Review file | `E:\Media\catalog\poem_import\proposal.json` | the guesses only (AHK's JsonParse is ~1s/300 KB) |
| Poem bridge (AHK) | `Helpers\PoemsFunctions.ahk` | `EditPoem` ("show this"), `AddPoem`, `AddScrap`, `CopyPoemForDoc` |
| Reader | `Scripts\reader\reader_collection.py` + `read_server.py`, openers in `Helpers\ReaderFunctions.ahk` | reader type `poetry`; `OpenMyPoems` `OpenMyScraps` `OpenAllPoems` `OpenFavoritePoems` `OpenRandomPoem` `OpenRandomPoemAnywhere` `OpenPoemsRanked` `ReaderShowThis` |
| Poems Miller | `Helpers\PoemsMenu.ahk` + `Scripts\PoemsViewer.ahk` | `OpenPoems`, `OpenPoemsAtItem`; settings `poems.list_order`, `poems.preview_meta` |
| Cover letter | `E:\Media\private\poet_profile.json` | outside every git allowlist on purpose (address, phone) |
| Tests | `Scripts\codebase_tools\tests\test_poems_store.py`, `test_poem_import.py`, `test_reader_poetry.py` | 77, all green |

### Dry run against the real doc (HTML export, 2026-09-14)

- Coverage OK: all 9,086 non-empty paragraphs land in exactly one unit.
- Plain-text cross-check OK: all 9,087 lines identical, in order.
- **464 poems, 508 scraps, 6 folded in as notes**; 49 favorites prefilled from 9s/10s; 7 special.
- 225 heading poems (the h2s, incl. new "Thumbprint"), 126 bold-title poems, 113 untitled.
- 93 guesses left for Jamie: 84 untitled 6+-line pieces (poem or scrap?), 6 loose-line
  groups under a heading poem (draft note?), 3 one-line heading poems.
- People linked: Charli 26, Cedar 3, Emily 2, Nata 1.

### Rules the real doc forced (each cost a parse pass; all pinned in test_poem_import.py)

1. **Stanza breaks mean different things by era.** Heading-era poems use an empty
   paragraph as a stanza break (2+ empties end the poem); older poems use a literal
   `-` line, so there a single empty paragraph ENDS the piece. Decided per container
   (any lone `-` in it), not by position -- the eras overlap.
2. **Endings.** `{tags} : {…}` ends the text and tags it (tokens that are exact-case
   person names become person links, not tags). A separator bar (`-----`, `====`,
   `(Printing)`) or a `POEM:` / `Poem:` label also ends the text. Everything after is
   a NOTE on that poem (prose versions, "Send to Jordy", alternate lines).
3. **Bold titles** split a heading's body only in the dash/bold era, or when the bold
   line carries her codes (`(NC/C) (8) Ribs`). Splitting everywhere cut the grief
   essay into Abstract / Main Body / Conclusion "poems".
4. **The line right under a heading is never a title.** It was a date
   (`12/10/2025`, `(very old, edited on 9/22/2025)`) and became its own poem.
   Pure dates become `written`; "edited on" dates go to `doc_edited`.
5. **A title that IS a name is not a person link.** Summer, Forest, Olive and 10 are
   all people in the store and all poem titles. Only `(Name)`, `For Name` and
   `{Name}` link, and only on an exact-case match.
6. **Bold-era title over 1-2 lines is a scrap** ("Relay / Running a relay race…"):
   an idea with a name, not a poem.
7. Untitled pieces of 5 lines or fewer are scraps outright (grow one later); 6+
   lines is a guess, because the bold era also holds other people's verse (a
   Wordsworth ode sits there untitled).

**Decision 13, as built:** the review actions are poem / scrap / note on the piece
above / part of the piece above. "Split" was dropped: the parser now splits
aggressively and "part of the piece above" undoes a wrong split.

### Resolved with Jamie (2026-09-14)

- `political` is its own tag, with `theory` as its parent -- the books-era alias that
  folded it INTO theory was removed. `poem` was added to
  `media_catalog.EXTRA_APPLIES_TYPES` so the shared vocabulary knows the type.
- Ty is a person (he/him); `{Ty}` links to him.
- The review list is not a gate: she imported on the guesses to review later, so
  `poem_import.py decide` applies each decision to the store (poem <-> scrap flips;
  note / join move the exact words and push a revision; an edit made since REFUSES).
- The cover letter lives in `E:\Media\private\`: E:\Media and TEXTDOCS both push to
  GitHub, and the media repo's allowlist `.gitignore` ignores every top-level folder
  except `catalog/` and `Books/`.

### Steps 3-4 as built

**Reader** (reader type `poetry`, `reader_collection.resolve_poetry`): specs `poems:`,
`scraps:`, `allpoems:` with `;`-joined filters -- `rank=8,9,10`, `fav` / `fav=no`,
`status=NE`, `special`, `tag=grief`, `person=charli`, `year=2026`,
`order=random;seed=N`. Her stanza marks (empty line or lone `-`) become one
`[[stanza]]`; untitled poems don't print their first line twice; the footer line is
date · rank · status. Favourites write to whichever store the id belongs to. Grabbing is
refused in this type (it would file her poem into the quote store as a stranger's).
Quote-store poems now render as verse too.

**Writing here:** `AddPoem` / `AddScrap` (dated today, same header lines as the editor),
`EditPoem` via "show this", and in the Miller "Not in the doc yet" + "Copy for the doc"
(`poems.py doc-format`: her own `(NC/C) (9) Title - Date` / poem / `{tags}` shape) -- the
v1 of "write in the new system and update the Doc afterward".

**Miller** (`OpenPoems`): All · Favorites · By rank (each rank -> Not favorites /
Favorites / All) · Needs editing · status · Special · person · tag · year · Versions
(only once one exists) · Scraps by type · write new · random · guide · import review ·
trash. A poem drills to pick-lists (rank, status, special, tags, kind, scrap type) and
carries row actions 1 read · 2 edit · 3 +1 · 4 -1.

**Bugs the real data caught** (all pinned by tests):
1. Doc order sorted backwards -- position 0 is the NEWEST poem.
2. `urllib.parse.urlparse` splits `;params` off the last path segment, so a bare
   `/poetry/poems:rank=8;fav=no` silently lost its favourite filter; `;` is now encoded.
3. Versions grouped by title alone: two different "Voices" (2026 and years older) became
   one poem in two versions, and a shuffle dropped the older. A version is now only a
   LABELLED one (v1/v2/v3). None of the 22 labelled poems has a sibling in the Doc -- she
   kept only the latest draft -- so Versions fills in as she makes versions here.
4. The viewer's TSV escaper turned `0` into `""`, and AHK's `"" + 0` throws.
5. The Miller engine's `_Mcp_FitColumns` never subtracted the vertical scrollbar, so the
   last column of every scrolling list was clipped (fixed in the engine for every Miller).

### Step 6 -- poem-poster move (built 2026-09-14; old folder not archived yet)

- **Code** copied to `AutoHotkey\Scripts\poem_poster\` (templates + fonts travel with it).
  Every script now gets its paths from `poster_paths.py`: `CODE_ROOT` = the script
  folder, `ASSET_ROOT` = `E:\Media\Poetry\poster` (override with the
  `POEM_POSTER_ASSETS` env var). Its CLAUDE.md has a "MOVED" note at the top.
- **Assets** copied (robocopy, not moved) to `E:\Media\Poetry\poster\` -- backgrounds,
  inspiration, letters, completed, out, clipart. E:\Media's allowlist `.gitignore`
  keeps them out of git.
- **Repointed:** `OpenPoetryMenuAt` (`Cfg.AHKBase "\Scripts\poem_poster"`),
  `~\.claude\scripts\printer.ps1` `$PoemPosterRoot` (+ printer.md), and the
  ahk-functions skill's working-dir example.
- **Store ↔ poster:** `poems.py poster-link --commit` gave 189 records their old
  `NNNN-slug` stem (all distinct; matched on the first 3 lines skipping date lines, then
  1 line, then a unique title) and carried 3 files' `categories`. `poems.py
  export-poster` writes all 464 poems to `E:\Media\Poetry\poster\poems\` (front matter
  title/rank/status/special/date/categories/id; blank-line stanzas become `-`).
  poem-poster's `poems\*.txt` is now an export -- edit in the store, re-export.
- **Verified:** every asset folder matches the original file-for-file and
  byte-for-byte (clipart 38,590 files / 29.19 GB); a variant render of
  1188-watering-a-dead-tree ran entirely from the new code + E: assets.
- **Left:** archive (not delete) `Desktop\Important\projects\poem-poster` once Jamie
  says so.

Original plan, for the record:

Code finds everything through `Path(__file__).parent` (render, finalize, publish, menu,
ingest, browse, search, contact_sheet, sources/_tags). Sizes: clipart 27.8 GB / 38,590
files, out 1.1 GB, completed 175 MB, inspiration 61 MB, backgrounds 36 MB. Plan: code +
templates -> `AutoHotkey\Scripts\poem_poster\`, one shared paths module pointing assets
at `E:\Media\Poetry\poster\` (outside the media repo's allowlist, so never committed);
COPY, verify, repoint `OpenPoetryMenuAt` and `~\.claude\scripts\printer.ps1`
(`$PoemPosterRoot`), then archive the old folder. `poems\*.txt` becomes an export from
the store keeping the old `NNNN-slug` stems that `completed/` and `out/` are keyed by.

### Doc sync (built 2026-10-10)

Jamie kept writing in the Doc instead of the store because the Doc is reachable from
anywhere. So the store and the Doc now stay in step, both directions, with no
browser and no paste.

| Piece | Where | Does |
|---|---|---|
| Google access | `Scripts\poems\gdoc.py` | `fetch` (HTML zip + txt snapshot via Drive export), `paragraphs` / `insert_block` / `replace_range` (Docs API), `auth` |
| Doc -> store | `poem_import.py sync` | three-way merge per field against `doc.baseline` |
| Both, orchestrated | `Scripts\poems\poem_doc_sync.py` | `pull`, `push <id>`, `push-pending`, `auto <id>` |
| AHK | `PoemsFunctions.ahk` | `PoetryDocSync`, `PoetryDocPush(id)`, `PoetryDocPushPending`; Miller rows "Sync with the Google Doc" (root) + "Send to the Google Doc" (each poem) |
| Setting | `poems.doc_push` | `all` (default) / `finished` (NC/C or F only) / `off` |
| Logs | `INIDATA\poem_doc_sync.log` (+ `.out.log` for the detached runs) | |
| Snapshots | `E:\Media\Poetry\doc_snapshots\<stamp>\` | last 60 kept; outside the repo allowlist |
| Tests | `test_poem_doc_sync.py` | |

**Auth.** OAuth client `claude-workspace` (same one as the workspace MCP), consented
as natehoop@gmail.com, token `~\.claude\secrets\google\poems_doc_token.json`.
**TRAP: while the OAuth app is in "Testing", Google expires refresh tokens after 7
days** -- the MCP's token had died that way (expiry April). Every failure raises
`GDOC_AUTH_EXPIRED`; the auto push toasts it. Fix for good = publish the consent
screen app to "In production" (personal use, no verification needed for 1 user;
Google shows an "unverified app" screen once). Re-auth: `py gdoc.py auth`.

**Pull (Doc -> store)** = fetch + `sync`. Matching is one-to-one, strongest first:
heading id -> unit key -> title+date -> text similarity >= 0.75. Then per field
(text, title, rank, status, special, written, header, notes): Doc == baseline ->
nothing; store == baseline -> take the Doc's (old text kept as a revision, `why:
doc`); both moved -> CONFLICT (store kept, `doc.conflicts`, reported). Tags/people
merge by provenance (only `src: doc` rows move). Never deletes: gone from the Doc =
`doc.missing`. A no-op sync writes nothing (no backup copy, no commit). Moves count.

**Push (store -> Doc)**, called detached after every `save-box` of a poem:
- not in the Doc -> inserted ABOVE THE NEWEST POEM (first Heading-2), never at
  index 1: the Doc opens with its Title paragraph + 4 smart-chip links, and a poem
  above them would swallow them on the next read. Shape: Heading-2 header line, one
  empty paragraph per stanza (two empties END a heading-era poem), `{tags}` line,
  4 empty paragraphs after. A poem using any lone `-` gets `-` for EVERY stanza
  break (dash-era rule: one empty paragraph ends it).
- already in the Doc -> its block (bounded exactly as the importer bounds it,
  dash era per container, stopping after `{tags}` so notes are never touched) is
  replaced -- ONLY if the Doc's copy still reads like the last sync. Edited in the
  Doc too -> nothing written, toast. Her own header line and `{tags}` line are KEPT
  verbatim when their values didn't change (her headers vary -- `(8) (NE)`, times,
  a date on the next line, `(edited on ...)` notes -- and her tags are her words:
  `{Death}` where the store keeps `mortality`). Measured: 238 of 240 heading poems
  round-trip as a no-op; the other 2 are correct refusals.
- every push is followed by a pull, so the record learns its heading id at once.
- **Scraps** live under her `POETRY SCRAPS` Heading-2 (above POETRY COVER LETTER,
  framed by `====` bars; she moved them there 2026-10-10). The importer reads that
  section like the old top of the Doc: everything is a scrap, ONE empty paragraph
  separates them, bold means nothing (most are bold -- the bold-title rule would
  have split them into "poems"), bars are furniture (a bar alone is never a
  record). Push: a new scrap goes to the TOP of the section, under the bar; a
  blank line inside a scrap is written `-` (an empty paragraph would split it);
  an edited scrap is found by its baseline text among the section's runs and
  replaced, refused if the Doc's copy changed too. Verified on a Doc copy: insert,
  replace, refusal, read-back as one scrap, and the rearranged real Doc matched
  all 991 pieces (moved, none new/missing).
- A scrap edited HEAVILY in the Doc (similarity < 0.75, e.g. a one-liner that
  doubles) can't be matched: it comes in as new and the old record is flagged
  `doc.missing` -- never deleted, visible in the sync report.
- **`sync poems`** (voice, generic store) = `PoetryDocSync` = `poem_doc_sync.py
  sync`: push everything pending (refusals reported, not fatal), then pull. So a
  push missed while the login was expired or offline goes up on the next sync.

**Bugs the real Doc caught (pinned):** the Drive API export INLINES styles
(`<span style="font-weight:700">`) where File > Download uses CSS classes -- the old
parser lost every bold-era title (126) silently; the Doc gained a Title-styled
"Lyrical nonsense" paragraph that became a scrap; a Shift+Enter soft break is `<br>`
in HTML but a space in the API's text export (cross-check now tolerates it).

**First run (2026-10-10):** 15 new poems (Sept 18 - Oct 10) + 4 scraps imported,
Thumbprint updated (text + rank 7 -> 8), 0 conflicts, 0 missing.

### Never out of step without saying so (built 2026-10-10)

Jamie's rule: the store and the Doc must not drift apart unnoticed.
- **Health** (`poem_doc_sync.health`, no network): the login's age (`authed_at` in
  the token file, written by `gdoc.py auth`; Testing-mode tokens die after 7 days),
  the last Google round-trip's outcome (`INIDATA\poem_doc_sync_state.json`, written
  by every auto push / sync / reauth) and the attention count. Rides on the Miller's
  counts feed as row `dochealth` (no extra process).
- **Where it shows:** the TOP row of `open poetry` ("⚠ Doc NOT syncing - Google login
  expired" / "...expires in N hours"; Enter fixes it), and the new-poem / new-scrap
  writing box's headline + a tooltip (`_PoemBoxDocWarning`). The failure toast is
  clickable: `PoetryDocReauth` -> `poem_doc_sync.py reauth --notify` in its own
  process (browser consent, then a full sync, then a tooltip with what went up).
- **Needs attention** (row 1-2 of the Miller when non-empty; `poems.doc_attention`,
  TSV column 19): `conflict` (edited both sides -- NEVER auto-pushed, `push`
  refuses), `missing` (gone from the Doc), `not sent` (no Doc copy 10 min after its
  last save), `behind` (Doc has an older copy -- includes Miller rank/status edits,
  which don't push on their own). Each poem then shows its fixes FIRST: keep this
  version / take the Doc's / put it back / it left on purpose / send now
  (`poem_doc_sync.py resolve <id> here|doc|reinsert|dismiss`, AHK
  `PoetryDocResolve`). `sync poems` also re-sends everything `behind`.

### Journaling about a poem (built 2026-10-10)

`journal poem` (pick list, newest first), "journal this" on a poem in the reading
room, and the Miller's "Journal about this poem". Source kind `poem`, numbered per
poem: `Poem: Thumbprint #2: `. Details in [[JOURNAL_SOURCES]] § "Her own poems".

### Backup to GitHub (built 2026-10-10)

`Scripts\writing_backup\writing_backup.py`. `media_catalog.write_media_type` (the one
writer every store uses) calls `schedule(path)`; for `poem.json`, `journal.json`,
`story.json`, `practice.json` it spawns a detached windowless worker that waits for
20 s of quiet, commits ONLY those files (`git commit -- <paths>`) in `E:\Media`
(private repo JamieHooper-code/media-catalog) and pushes. Offline = committed
locally, pushed next time. Log `INIDATA\writing_backup.log`; `py writing_backup.py
status|now`. `WRITING_BACKUP_OFF=1` disables (tests). `catalog/*.backup-*.json`
are now gitignored (git history is the backup).

### The writing box + brace markers (built 2026-09-23)

`AddPoem` / `EditPoem` / `AddScrap` open a standard `WritingBox("page")` -- full
screen, the same surface as "journal classic" -- in their OWN process (the old box
lived in the MAINFUN dispatcher, so any voice command closed it). The box is one
text: the poem plus brace markers that can sit anywhere and add up --
`{title X}` `{rank 9}` `{needs editing}` / `{nearing completion}` / `{finished}`
`{in progress}` `{special}` `{favorite}` `{tags grief, Charli}`. Editing opens with
the current values as markers, so deleting one clears it. `poems.py box-text` builds
that text; `poems.py save-box` reads it with the shared `Scripts/journal/markup.py`
and applies everything in ONE locked write (the old box made up to seven). Old
`title:` / `rank:` header lines at the very top are still read, for drafts.

Voice, only in front of a "Writing box": `tag <words>` (tags and people), `title <words>`
(title-cased on save by `markup.title_case`, the twin of AHK `TitleCaseText`; a title
she already has is never recased; the last one said wins), `rate <n>`,
and `mark ten`..`mark one`, `mark editing` / `nearing` / `finished`, `mark progress`,
`mark special`, `mark favorite` (the Google Doc's own "Mark ..." words, so her mouth
already knows them). Tests: `test_markup.py`, `test_poems_store.py` (box section).

### In progress + the `poetry` namespace (built 2026-09-23)

**In progress** is a boolean `in_progress` on the record, beside `special` -- NOT a
fourth status. Jamie: an in-progress poem is still a needs-editing one, it just
must not drown among the ~50 NE poems (some untouched for years). So turning it
ON also sets status `NE` (`poems.set_in_progress`); turning it OFF leaves the
status alone. It shows under both "In progress" and "Needs editing".

- Store: `poems.py set-progress <id> on|off`, filter `--in-progress`, `add --in-progress`.
  Viewer TSV column 18 (appended -- PoemsMenu reads columns by position).
- Miller: "In progress" is ROW 2 (right under All poems), last-edited first; each
  poem has an "In progress: yes/no" line (Enter switches).
- Writing box: a `progress: yes|no` header line in EditPoem and AddPoem.
- Reader: spec word `poems:progress`, an Any/yes/no toggle in the filter pane,
  "in progress" in the footer line. `OpenProgressPoems()` = `poems:progress;sort=edited`.
- Tests: 4 in `test_poems_store.py`, 1 in `test_reader_poetry.py`.

**Voice moved to `poetry ___`** (domain-first, like the practice commands):

| Phrase | Does |
|---|---|
| `open poetry` | `OpenPoems` (the Miller) |
| `poetry progress` | `OpenProgressPoems` -- in-progress poems in the reader |
| `poetry show [filter] [sort]` | `ShowPoems` -- filter words now include "in progress" |
| `poetry all [filter] [sort]` | `ShowAllPoems` |
| `poetry scraps [sort]` | `ShowScraps` |
| `poetry new` / `poetry new scrap` | `AddPoem` / `AddScrap` |
| `poetry random` | `OpenRandomPoem` |
| `poetry poster` | `OpenPoetryMenuAt` (the poem-poster menu) |
| `edit this` (reading room) | `ReaderShowThis`, same as "show this" |

The old `open poetry` was a DIRECTORY entry (`poem_poster` folder in Claude
desktop); it was renamed to `poem poster` ("open poem poster") to free the phrase.
The older phrases (`open poems`, `show poems ...`, `add/make poem`, `add/make scrap`)
still work during the switch; retire them once the new ones are habit.

### Voice + sorting (built 2026-09-14)

Jamie picked the phrases (her reply replaced the recommended `read ...` / `see ...` set):

| Phrase | Does | Where it lives |
|---|---|---|
| `show poems [filter] [sort]` | `ShowPoems(filter, sort)` -- her poems in the reader | caster `book_commands.py` (Choices) |
| `show all poems [filter] [sort]` | `ShowAllPoems` -- hers + quote-store poems | same |
| `show scraps [sort]` | `ShowScraps(sort)` | same |
| `open poems` | `OpenPoems` (the Miller) | generic command store |
| `add poem` / `make poem` | `AddPoem` | generic store |
| `add scrap` / `make scrap` | `AddScrap` | generic store |
| `show this` (reading room only) | `ReaderShowThis` | generic store, context `reading_room` |

- **Filter words:** favorite(s), not favorite(s), a rank one-ten, "<rank> and up", and
  those combined ("eight and up not favorites"). AHK tokens `fav notfav 8 8+`.
- **Sort words:** newest (default) · oldest · random / shuffled · by rank · most loved ·
  by title · recently edited. Spec form `sort=newest|oldest|random|rank|loved|title|edited|written`
  (`poems.sort_order` is the one map; `sort=random` = the seeded `order=random`). Works in
  the address bar too, e.g. `/poetry/poems:fav;sort=oldest`.
- **Re-sort what's on screen:** `ReaderSort(sort)` -> `/api/sort`, spoken `sort <word>` in
  the reader only (`rules\reader_commands.py`; needs "reboot caster" then "enable reader
  rules" once). Words: newest, oldest, random / shuffle, by rank, most loved, by title,
  recently edited, by date written, reverse. Keeps the filters, starts from the top.
- **Filter & sort pane -- a GENERIC reader feature (2026-09-14):** the ⇅ button beside the
  gear opens a pane docked on the left like the contents (both can be up; the contents
  move over). Source radios; sort styles + a direction pair (the style's own direction
  first) + Shuffle; then checkbox groups with counts -- Favorites, Rank (chips 10s / 9 and
  up / 8 and up / 7 and up), Status, Special, Year written, People, Tags. Scraps get Kind of
  scrap instead of rank / status / special / people, and switching source drops filters the
  new source doesn't have. Boxes in a group are OR, groups AND; each count is what ticking
  that box gives (a group is counted with every OTHER group's filters). Jamie
  (2026-09-16): "any of" everywhere unless a group is inherently exclusive, and those are
  ONE toggle -- Favorites and Special render as Any / yes / no chips (`kind: "toggle"`);
  every other group (`kind: "multi"`) has a master box in its header that ticks the whole
  section, unticks it when all are on, and shows a dash when some are. The open flag is
  server-held like `toc` (`/api/filters`, AHK `ToggleReaderFilters`, spoken "show filters"
  in the reading room -- generic store row); `/api/facets`
  describes the pane, `/api/refine` applies the page's whole selection, and a combination
  with nothing in it is refused with the view kept. A type opts in through
  `reader_collection._REFINERS` (options / spec / resort) and the page never learns what a
  poem is -- the journal plugs in the same way. ~20 ms per tick on the real store.
- **Spec additions:** `dir=asc|desc`; `status`, `year`, `person`, `tag`, `stype` take comma
  lists (any of). `poems.query` gained `any_tags`, `reverse` and list-valued filters;
  `poems.sort_direction` / `NATURAL_DIR` own what each sort word means.
- **Reader bug found while testing:** Back from a poem view into an earlier book sent the
  book's index to the poems. The page now reloads an address that belongs to another
  reading (`popstate` in read_server.py).

---

## Doc sync fixes (2026-10-13)

- **Menu edits push too.** `set-rank/-status/-special/-progress/-title/-text/-kind/add-tags/remove-tags`
  schedule the same detached `auto` push as a box save (`_simple(..., push=True)`). Before this a rank or
  status set from the Miller never reached the Doc and the poem sat under Needs attention as "behind".
- **One Doc writer at a time.** Every `poem_doc_sync.py` command except `health` takes
  `INIDATA/poem_doc_sync.lock`. Two quick saves used to run side by side; the second read the store before
  the first had pulled the new heading id back and refused ("no Doc heading id yet").
- **A new heading id alone is saved.** A replace gives the heading a new id; the no-op sync used to drop
  it, so the next push could not find the poem. `merge_from_doc` reports `relinked`.
- **Kind change moves sections.** `set_kind` records `doc.kind_in_doc` (the kind the Doc still has);
  the next push deletes the old copy -- only if the Doc text still equals the last-synced text, and for a
  heading-less scrap only if that text appears exactly once -- then inserts it at the top of the right
  section. Location alone never triggers a move: ~480 scraps live among the older poems on purpose.
  `doc.in_scraps` records where each piece sits.
- **"type poem" / "type scrap" in the writing box** (the journal markup rule's `type <words>`, live in
  every writing box) sets the kind on save: a scrap box saved with `{type poem}` is born a poem, never
  a scrap. Any other `{type ...}` stays visible in the text. `KIND_MARK_RE` in poems.py.
- **Grow vs. make it a poem.** "Grow" makes a NEW poem and keeps the scrap (both end up in the Doc).
  To turn a scrap into a poem, use the Miller's "Kind: scrap - make it a poem" or say "type poem".

## Version numbers (2026-10-10)

Every saved change to a poem's text or title is a new version: `1`, `1.1`, `1.2` ... (her
rule). An edit that leaves the poem at least 90% the same (word-level difflib ratio,
`MINOR_SIMILARITY` in poems.py) bumps the minor number; below that the major number goes
up. The comparison is against the text the current MAJOR version started from, not the
last save, so small edits that add up to a rewrite still become the next major version.
A title-only change is a minor bump; a notes-only change is no bump.

**A writing session is one version** (Jamie, 2026-10-10). The writing box autosaves a
draft file every ~2s (nothing lost) but writes the store once, when she saves and closes
it; that save is one version even when it changes the title AND the text, via
`poems.one_version(rec)` (holds the number, stamps in-between revisions with the
pre-save version, bumps once on exit). One Doc edit pulled by sync, and a "take the
Doc's version" resolve, are wrapped the same way.

Stored, not recomputed: `version_no` on the record and `version` on each revision
(stamped by `push_revision`), so changing the rule can never renumber a page already on
paper. Records from before numbering were backfilled by `poems.py refresh` (989 at v1,
Thumbprint v2). `poems.py revisions <id>` lists every version. Unrelated to
`version_label`, which is a "v3" she typed into a title. Printed in the page corner --
POEM_PRINT_AND_DESIGN.md § Version mark.

## Decisions (Jamie, 2026-09-14)

Reply was `5.` + prose + `0.00` (= yes to all remaining recommendations, Claude's
call on the rest, prose overrides). Numbers refer to the 50-item design list.

| # | Decision |
|---|---|
| 1 | **Own sibling store** `E:\Media\catalog\poem.json`, engine `Scripts\poems\poems.py`. Shares the tag vocab (`media_tags.json`), person store, affinity engine and reader. NOT inside quote.json (its dedup / recapture / repair-breaks machinery treats changed text as a bad re-grab). |
| 2 | Own top-level node in the `open media` hub. |
| 3 | Poems and scraps in the SAME store, `kind: poem / scrap`, so promotion or fixing a misclassification is one field change. |
| 4-7 | Name = **scraps** (not seeds/threads/sparks). Voice phrases get picked from options at build time. |
| 8 | A scrap that becomes a poem stays, linked (`grew_from` / `grew_into`), under a "Grown" branch. Never deleted. |
| 9 | Scrap subtypes only: line / idea (auto-detected), name, image/phrase (from her bottom-of-doc lists). |
| 10-11 | **Import from the HTML export**, not plain text. Jamie downloaded it herself (see Sources). The .txt is the line-coverage cross-check. |
| 12 | The 4 lines at the top of the doc ("Poetry Challenges Practice", "Poetry Ideas Unwritten", "Poetry Images Phrases Names Things", "Poetry SongWriting Objects Places Things") are **links to other documents. Ignore them** (treat as furniture). |
| 13 | Uncertain blocks (mostly the older half) go into a **numpad review queue in a Miller**: 1 poem · 2 scrap · 3 join with previous · 4 split. |
| 14 | Import NEVER deletes. Missing-from-doc = a flag. Delete is explicit and soft (trash, restorable). |
| 15 | Every text change keeps the previous text as a revision. |
| 16 | Coverage guard (journal's): every non-blank doc line lands in exactly one poem / scrap / known furniture line. `import --expect N`. |
| 17 | Tag provenance `src`: `doc` (her `{tags}` line) / `manual` / `auto`. Re-import rewrites only `doc` rows. |
| 18 | Every import: dry-run report (new / changed / edited-here / conflict / missing) + dated backup before commit. |
| 19-22 | **BUILT 2026-10-10 -- see § Doc sync.** Originally: **Ongoing sync + write-back to the Google Doc = v2.** Plan: write in the new system, update the doc by hand afterward. Still store each poem's doc text as it was at import (cheap), so the v2 three-way merge has a baseline. |
| **dates** | **Keep the date each poem was WRITTEN (from its header) AND every date it was EDITED** (revision timestamps + `modified`). |
| 23 | Keep the header line verbatim alongside the parsed fields. |
| 24 | Keep her lone `-` stanza marks verbatim in stored text; reader + poster render them as stanza gaps. |
| 25 | Versions (Chrysalis v3, Amber v2) = separate poems linked as versions. Random/favorites show the newest. |
| 26 | Prose drafts / notes under a poem (`===`, `-----`, `POEM:` patterns) are sections of that poem, hidden while reading. |
| 27 | `{Charli}`, `(Emily)`, "For Cedar" → person-store links, not tags. |
| 28 | Never auto-generate titles. Untitled poems display their first line. |
| 29 | Local LLM may SUGGEST tags (unticked). Never auto-applies tags to her poems. |
| 30 | **Header codes are defined in the guide at the very bottom of the doc. Decode from there.** Unknown codes (P, PM, trailing C, 24, 99, 2) are kept raw. |
| 31 | Reader sources: "my poems", "my scraps", "all poems" (hers + quote-store poems), rendered as VERSE. |
| 32 | "All poems" does NOT include unfavorited poetry-book chapters. |
| 33 | Random poem (mine / all), and "next" stays random. |
| 34 | **Favorites use the existing scoring AND the affinity system** (`Scripts\MediaCatalog\affinity.py` bands: favorite ≥1, loved ≥3, essential ≥5, buried ≤-1). |
| 35 | **OVERRIDDEN: pre-fill favorites from rank 9 and 10 poems.** |
| **rank** | **Filter/sort by rank, combinable with favorite state**: "all the 8s", "8s that are not favorites", "8s, 9s and 10s". Needed in the Miller and in voice. |
| 36 | Slim header line in the reader: title · written date · rank · status · ★ special · people · tags. |
| 37 | "show this" (reading-room scoped) opens the current item in the house editor (`_SingleFieldSkipableInputGui`, line numbers, draft key, Ctrl+Enter). Phrase is free. |
| 38 | "show this" works for ANY reader item (exercises, quotes too). |
| 39 | Editor carries inline header lines `title:` `rank:` `status:` `tags:` (journal markup convention, `SplitLeadingMarkup`). |
| 40 | Tagging in the reader: numpad pick-list overlay + voice. **Future: a generic reader-wide "tag Charli" / "tag X" that tags whatever item is currently open.** |
| 41 | Fix the quotes-viewer tag bug (below) while in there. |
| 42 | `open poems` roots: All (newest) · Favorites · Needs editing · By rank · By status · Special · By person · By tag · By year · Versions · Scraps · Doc is behind · Guide. |
| 43 | Item actions: read · edit · tag · set rank/status · favorite · grow (scraps) · open in Google Doc at that poem · send to poster. |
| 44 | Afterward/Guide imported verbatim → Guide page (Miller root + vault note); its code legend becomes the importer's data file. |
| 45 | Cover letter + bio → private vault note only, NEVER the repo (address + phone). |
| 46 | Walden quote → quote store. "Names that I like" / "PHRASES AND IMAGES" → scraps. |
| 47 | poem-poster code → `AutoHotkey\Scripts\poem_poster\`; assets (38,590 clipart files, backgrounds, fonts, out, completed) → `E:\Media\Poetry\poster\`. |
| 48 | Retire `poems\*.txt` + `sync_poems.py`; poster reads from the store. Carry over `categories` (3 poems) + completed posters. |
| 49 | Repoint `OpenPoetryMenuAt` (`Helpers\OpeningAndClosingFunctions.ahk:747`), `printer poster` (`~/.claude/scripts/printer`), project docs. Archive (not delete) the old folder after verification. **No printmaking work** beyond making it run from the new home. |
| 50 | Build order: store + importer (dry-run) → review queue → commit → reader → Miller → voice (Jamie picks phrases) → poster move → (v2) sync. |

### New requirement: journaling about a poem (BUILT 2026-10-10, see above)

`journal poem` → journal about a specific poem; defaults to ASKING whether it is
the newest poem. `journal this` from the reader journals about the poem on screen.
This is a new journal `source` kind `poem` per [[JOURNAL_SOURCES]]: the reader's
`/api/state` already returns the current item, so it is a `source_fn` branch on
the reading-room context plus a `poem` kind in `SUBJECT_KINDS`, numbered per poem.
Eventual, not v1.

---

## Sources

| Thing | Where |
|---|---|
| Google Doc "Lyrical nonsense" | id `1LVSR4mEU1Y_ZL3cpB_XznQYXuB9jfdpJG9qI6wCeGAI`, owner natehoop@gmail.com, created 2020-03-12. **The doc is the word of God for the initial import.** |
| HTML export (import source) | `E:\Downloads\Lyrical nonsense.zip` (Google "Web page, zipped", 2026-09-14 11:51). Contains one poem newer than the first .txt. |
| Plain-text export (coverage cross-check) | `E:\Downloads\Lyrical nonsense.txt` (re-downloaded 11:53, includes the new poem). UTF-8 with BOM, CRLF, no U+FFFD. |
| Old poster project | `C:\Users\jamie\Desktop\Important\projects\poem-poster\` (not a git repo; OneDrive-synced Desktop) |

---

## What the survey found (so the next session does not redo it)

### The doc (first .txt, 12,892 lines)
- **Reverse chronological**: newest at the top (Sept 2026), older song lyrics at the bottom.
- Top ~120 lines: the 4 doc links, then scraps separated by 2 blank lines.
- **Recent header format**: `(NC/C) (9) Charli -  September 10, 2026`. Codes can come before or after the title; dates appear as `Month D, YYYY`, `M/D/YYYY`, on the next line, or in notes like `(very old, edited on 9/22/2025)`. Trailing `(PM)`, `(P)`, `(Charli)`, `(S) (Cedar)`, `v1/v2/v3`, `POEM:` prefixes also occur. 184 lines carry a code token; code counts: NC/C 99, NE 54, (8) 40, (9) 38, (7) 18, F 15, (?) 14, (C) 12 (mostly CHORDS in songs), (10) 11, PM 8, S 7, P 3.
- **`{tags} : {Grief} {Charli}`**: 32 poem-ending tag lines, all in the top ~7,200 lines. Several tokens are people (Charli, Ty).
- **Stanza marks**: 392 lone `-` lines (her own convention, literally in the text).
- **Furniture**: `====` bars (18), `(Printing)=====(Printing)` at ~line 3589, long `-----` separators, `TITLE` placeholders, `IN PROGRESS`.
- **Blank-line runs** are almost always even (2/4/6/8…). Runs of ≥4 very often precede a poem header; 2-blank runs separate scraps AND older poems. A usable secondary signal, not a boundary rule.
- **Past ~line 5,800 headers mostly stop**: bare titles, chord songs (`(C) when you punch my number`), dated hikes (`4/12/2024 hike with Geo`), one-line ideas ("Song about a dog that's jealous of his 3 legged brother…"). Poems and scraps interleave here, which is why the review queue exists.
- **Bottom sections**: loose Tortilla Flat notes and lyric fragments, then `Quotes:` (Walden), `Names that I like:`, `PHRASES AND IMAGES:`, `POETRY COVER LETTER` (name/address/phone/email + bio), then **`Afterward/Guide/Whatever`** which defines: 1-10 = her ranking on the day; NC/C = nearing completion/complete; NE = needs editing; F = finished (probably posted); S = special (unranked, about someone she loves); other numbers = letter-sum codes (41 = FUN).
- **Plain text loses heading styles.** The old sync found boundaries via Google Docs H2 headings (189 poems) and noted ~150-250 older poems marked only by **bold titles**. Hence the HTML export.

### poem-poster (old, outside AHK)
- 189 poems `poems\1000-sunlight.txt … 1188-watering-a-dead-tree.txt` (+ `sample.txt`), YAML frontmatter (`title, rank, status, special, date, poetry_month, letter_code, song_tags, categories`). `poems.backup\` differs by 4 files.
- **All 189 are in the doc**: 185 titles found verbatim, the other 4 were title munges; every opening matched verbatim. Only poster-only data: `categories:` on 3 poems, `completed\` (1188-watering-a-dead-tree, for-winter from `letters\`, `_gallery`).
- `1000-sunlight.txt` (125 KB) is a whole un-headed song pile the old sync lumped into one "poem". Do not repeat that.
- `sync_poems.py` depended on three Workspace-MCP caches (no longer on disk) and title-equality matching. Superseded.
- Clipart: 38,590 files. `out\` 781 files.

### Quote store (`E:\Media\catalog\quote.json`)
- **Zero of her poems.** 4 poems total (Audre Lorde ×1, Andrea Gibson ×3). Her own writing = `source.kind: "original"`, `group: "mine"`, 2 short prose records.
- Favorites = `affinity` (`quotes.py affinity <id> --by ±1`; bands stored as `src:"affinity"` tags). Exposed only in the reader (Up/Down, "mark favorite" in `generic_commands.json`, reading_room scope).
- Random: `quotes.py pick --pool` over `INIDATA\display_pools.json`; no entry_type or affinity filter; no voice command.
- **BUG (verified)**: `cmd_set_tags` (`Scripts\quotes\quotes.py:1590`) re-stamps every remaining tag `src:"manual"`, so the viewer's untick (`QuotesMenu.ahk` `_QTagToggleNodes`) erases auto/import/affinity provenance.
- Importers skip existing dedup keys, and `add --replace` DROPS tags/facets/affinity. Another reason poems are not in this store.

### Reader (`Scripts\reader\read_server.py`, port 8289)
- Collections resolve via `resolve()` (`reader_collection.py:716`); non-quote sources plug in as spec prefixes like `read:` / `smut:`. Poems need a `poems:` / `scraps:` / `allpoems:` source the same way.
- `/api/state` returns the current item (spec, type, index, `item.id`), which is the hook for "show this", tag-current, and `journal this`.
- **Gaps**:
  - No random.
  - No tag UI.
  - No edit command.
  - Openers don't pass `&type=`, so `/api/open` falls back to `exercise` (`read_server.py:1429`).
  - Collection subtitle hardcoded "%d practices" (`reader_collection.py:742`).
  - **Verse flag is set only for EPUB chapters** (`reader_collection.py:475`), so store poems render with prose CSS.
- Editing today exists only after a grab (`ReaderReviewQuote` → `_QReviewQuote`, `QuotesMenu.ahk:1351`).
- Traps: the server holds its code in memory, so run `py read_server.py restart` after edits.

### Voice phrase collision notes (voice_index, 2026-09-14)
- `show this`: WARN only (siblings `show <slash_choice>`, `show <list_slot>`). Free; scope it to the reading room.
- `tag this` / `tag <X>`: **COLLISION** with journal markup `tag <textnv>` (`JournalMarkupRule`, function_context-scoped to the writing box + Google Docs). A reader-scoped `tag <X>` must use a context that excludes those.
- `read poem…`: siblings `read <reading_book|person|genre>`. Watch Choice keys.
- `open poems` / `open scraps`: siblings `open <program|directory|open_item>` Choices. Words must not be keys there.
- `random poem`: lint, not verb-first.

---

## Record (sketch from the design session -- `poems.py new_record` is now the source of truth)

As built, a record also carries derived `words`, `lines`, `stanzas`,
`display_title`, `year` (refresh_derived), `affinity_prefill` when the importer set
a favorite from a 9/10, and `doc.unit_key` / `doc.absorbed_units` tying it back to
the review decisions.

```json
{
  "id": "poem:charli-2026-09-10",
  "kind": "poem | scrap",
  "scrap_type": "line | idea | name | image",
  "title": "Charli", "title_source": "doc | manual",
  "text": "…her lines, lone '-' = stanza…",
  "sections": [{"role": "draft|note", "text": "…"}],
  "written": "2026-09-10", "written_raw": "September 10, 2026",
  "rank": 9, "status": "NC/C", "special": false,
  "codes_raw": ["(NC/C)", "(9)"], "header_raw": "(NC/C) (9) Charli -  September 10, 2026",
  "version_label": "", "version_of": null,
  "grew_from": null, "grew_into": null,
  "tags": [{"tag": "grief", "src": "doc"}],
  "people": [{"id": "person:charli", "src": "doc"}],
  "affinity": 0,
  "doc": {"position": 0, "baseline_text": "…", "imported": "2026-09-…"},
  "poster": {"categories": [], "completed": []},
  "revisions": [{"at": "…", "text": "…", "why": "edit|import"}],
  "status_flags": [], "added": "…", "modified": "…"
}
```
