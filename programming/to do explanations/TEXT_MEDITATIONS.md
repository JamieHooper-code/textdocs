---
tags: [programming, design-doc, meditation, reader, ifs, quotes, ahk, caster]
created: 2026-11-11
status: v1 built 2026-11-11 (reading, votes, both views, arrange page, voice)
related: ["[[GUIDES_SYSTEM]]", "[[MEDITATION_PAGE]]", "[[QUOTES_SYSTEM]]", "[[NEW_TAB_PAGE]]", "[[SETTINGS_SYSTEM]]"]
---

# Text meditations

Guided meditations made of TEXT, read in the reading room, built section by
section so Jamie can jump to whatever she needs: grounding, unblending, "when
nothing's answering". The first one is **Daily parts meditation** ("meditate
parts"), distilled from the ten No Bad Parts exercises. It is meant to be the
first of many.

## What she asked for

- Sections she can jump between by saying their names, plus a contents sidebar.
- Two views: **all sections together** (default) or **one section at a time**,
  switchable by a setting and by voice.
- **Up/down votes on every line**, so a section can be condensed (downvote
  what doesn't land) or grown (add lines) without getting overwhelming, and so
  we learn what she likes per section.
- **Per-section display rules**, like the new tab page panels: show all, or
  show N; order favourites-first (the default), random, or random weighted
  toward favourites.
- **Recommended next sections** at the top of each section. Each section has
  its own links, so the meditation reads in order but also works as a web
  ("nothing's answering" points to Making space and Arriving).

## Decision: build it in the reader (2026-11-11)

Weighed against a separate page on the meditation server (:8295). The reader
already has everything a reading surface needs: 25 display prefs, paged and
continuous flow, the contents sidebar with voice jump numbers, page turning
from voice / pedals / remote (~80 AHK reader functions), block numbering for
"up 4", history and resume, ambient sound, grab, the writing box. A second
page would copy a thinner version of that and drift. So meditations are one
more reader content type, kept in their own files so the reader itself grows
very little.

## Shape

| Piece | Where |
|---|---|
| Meditation structure (sections, their rules, next links, section votes) | `E:\Media\catalog\textmed.json`, ONLY writer `Scripts\meditation\text_meditations.py` |
| The lines | quote store rows, group `meditations/<slug>/<section>`, entry type `meditation_line`, votes on the shared affinity scale |
| Reader plug-in | `Scripts\reader\reader_meditation.py` (`medtext:<slug>` specs) |
| Arrange page | served by the reader at `/medarrange/<slug>`, its own HTML file |
| Global defaults | Settings schema `meditation.json` (`text_view`) |
| AHK | `Helpers\TextMeditationFunctions.ahk` |
| Voice | "meditate <name>" (shared with the audio packs), section names while a meditation is open |

**Why lines live in the quote store.** Votes, burying (score <= -1 is never
drawn), weighted draws and the gentle-lines review workflow already exist
there; the new tab check-ins are the same pattern. Editing a line with
`quotes.py set-text` keeps its id and votes. The meditation record keeps the
ORDER of each section's lines (ids), which is the authored order used as the
tie-break.

**Why votes on lines do not reload the page.** A reload re-resolves the
collection, which would reorder a favourites-first section under her eyes. The
line vote endpoint writes the score and the page updates that one line in
place; the new order shows next time it opens.

**Stable random.** The spec carries a seed (`medtext:parts;seed=N`), made when
she opens it, so a random or weighted section keeps the same picks through
page turns and reloads, and a new open is a new draw.

## Section rules

`show`: 0 = all, else how many lines. `order` decides which lines are drawn:
`favorites` (the top scores), `random` (all equally likely), `weighted`
(seeded weighted sample like the new tab: weight 1 + score). Buried lines are
never drawn. `next`: section keys. Defaults: show all, favourites first.
Since 2026-11-13 the drawn lines always SHOW in written order (plus roles,
below). Before that, a voted-up closer jumped to the top of its section.

## Voice

- `meditate parts` opens it (MeditatePack routes a `medtext:` name to the reader).
- `meditation together` / `meditation one at a time` flips the view.
- Section names ("arriving", "making space", "forcing it") jump while a
  meditation is open in the reader (phase 3).
- Up/Down arrows vote on the whole section; line votes by the buttons, or
  "favorite 4" / "bury 4": each line's vote column carries a kit jump number
  and those are the page's own action words (no Caster grammar, no reboot).
- Section names are a Caster rule scoped to tab titles containing
  " · Meditation · ReadingRoom" (the reader adds " · Meditation" for these),
  so "closing" never fires over a book.

## Built (2026-11-11)

- "Daily parts meditation": 16 sections, 108 lines imported from the guide
  of the same name (`import-guide`). The guide still exists; "guide parts"
  and "meditate parts" both work, and they will drift if one is edited.
- Reader: `medtext:` spec, one item per section plus an opening item, "Start
  with" / "Next" strips, line arrows (no reload), Up/Down on a section,
  both views, tab title " · Meditation · ReadingRoom".
- Voice: "meditate parts" (also "daily parts", "parts meditation"),
  "lockout meditate parts" just opens it, bare section names, "meditation
  together" / "meditation one at a time", "arrange meditation",
  "favorite N" / "bury N" on the numbered lines.
- Arrange page `/medarrange/<slug>`: show All / N, order chips, next links,
  per line vote / move / remove, add a line (saved to the quote store).

## Reading mode and editing mode (2026-11-11, second pass)

Jamie's review after first use: settings belong in the reader, not on a
separate page, and the buttons should only be there when she is editing.

- **Reading mode** (default): text, the "Start with" / "Next" links (now a
  quiet stacked list with a thin accent rule, the spoken words in bold), no
  vote buttons, nothing numbered on the lines.
- **Editing mode** ("editing mode" / "edit meditation", the footer's Edit
  button, setting `meditation.text_editing`): every line gets ▲ ▼ ✎, each its
  own number (grouped per line so they run consecutively); each section gets
  Rename and a Section settings pop-up (show All / N, order, next links -- the
  arrange page's controls, in place). Edits go through `quotes.py set-text`, so
  a line keeps its id and votes. The mode reaches an open page on the poll
  (`med_editing` in the state), never by a reload, so she keeps her page.
- **Footer**: "One at a time" / "All together" and "Edit" / "Done editing",
  beside the reader's own buttons.
- **One at a time**: a section that fits in one column is shown as one
  centred column instead of a two-page spread.
- **Contents sidebar**: a "Next from <section>" box on top; rows show the
  spoken words in bold; the row's own count is hidden so the only number is
  the voice one. (Pinning rows to 1-17 is impossible: the reader pins 11-21 to
  its own controls.)
- **Spoken numbers**: the kit's number rule (`rules/local_pages_commands.py`)
  skips every ReadingRoom tab (`_OWNS_DIGITS`: the reader's own "make grab"
  numbering reads typed digits). Meditation tabs are let back in by their
  title ("meditation · readingroom", `_MED_MARK`), so line, link and contents
  numbers are sayable there. Found 2026-11-11 when none of them worked by
  voice -- the first build was only tested by calling the kit directly.
- **Voice**: bare "contents" / "filters" / "reader settings" in the whole
  reading room (generic store, reading_room context); "editing mode" /
  "reading mode" in meditations.

The arrange page (/medarrange/<slug>) still works and shows everything at
once, but day to day the in-reader controls replace it.

## Third pass (2026-11-11): chips, add/remove, paging

- Next links are dark filled chips. Editing mode adds "+ Add line" per section
  and a ✕ per line (press twice; the line leaves the section, its quote stays),
  and a number box in Section settings (type how many, Enter; 0 = all).
- Voice numbers sit in room each button already has (absolute ::before), so
  their arrival after a page turn never reflows the page.
- Spoken button names: "edit" / "done editing", "one at a time" / "all
  together". The reader footer's icons now carry words (Contents, Settings,
  History, Filter), reader-wide.
- READER CORE (read_server.py), both reader-wide but inert for books:
  `SHIFT` lets a page start on any column; jump() in a meditation puts the
  section's column at the top-left (`anchorItem`, re-anchored once after 700ms
  unless she turned a page), and in all-together view each section starts at
  a column top (`break-before: column`). Held arrow / Page keys turn at most one
  page per 240ms (`turnTooSoon`), and the place bookmark is debounced 400ms.

## Paths and stops (2026-11-11, fourth pass)

Ask: going through every section in the order it was written was too long, and
some steps that come one after another really happen together ("finding a
part" goes along with "letting them know you" / "opening the door"). She wanted
a short **main path** shown first, with every other section still there and
linked, and all of it modular: many paths, shared stops, any order.

- **Stop:** one page drawing lines from one or more sections. Each source has a
  default amount (N, all, or the section's own show rule). Stops are reusable.
- **Path:** an ordered list of stops and/or plain sections. A path can override
  any stop's amount for one source, for that path only. `default_path` is what
  opens; `;path=<key>` or `;path=none` in the spec picks another.
- **Reader:** path pages come first after the opening, and each page's Next
  starts with the path's next stop. The other sections follow in their written
  order. The contents sidebar shows the path under "Next from", with the
  current stop in bold. Stop item ids are `textmed:<slug>/@<key>`. Line votes
  and remove work on stop pages, since each line carries its source section. A
  section vote on a stop is refused.
- **Built for parts:** stops *Who's here* (3 from Checking in + 2 from Opening
  the door) and *Being with a part* (3 from Finding a part + 3 from Getting to
  know a protector), each with its own curated "When…" side trips. Paths: **Main
  path** (default) is Arriving → Who's here → Making space → Being with a part →
  Letting them know you → Closing. **Short sit** is Arriving → Who's here → Being
  with a part → Closing.
- CLI: `text_meditations.py paths | stop-set | stop-take | stop-drop | path-set |
  path-take | path-move | path-drop | path-default`. Tests:
  `Scripts/codebase_tools/tests/test_text_meditation_paths.py`.

### Editing on the page and by voice (same day)

- **Footer "Path: <name>"** opens the Path pop-up. In reading mode it only
  switches the path you're walking (each path, or All sections). In editing mode
  it also has Make default, Rename, New stop (which goes onto this path, then
  opens its settings), Copy as new path and Delete path (press twice), plus
  every entry with ▲ ▼ ✕ and Stop settings, and an Add row with every stop and
  section not already on the path.
- **Stop settings** (on a stop page, or from its row): Rename; an Amounts
  switch between *Stop default* and *Only on <this path>*; one row of amount
  chips per source (Its own rule, 1–6, All; in path scope also "Stop
  default (N)" to clear the override); Sources (toggle sections in, a new
  source takes 2); Side trips; Delete stop (only when it's on no path).
- All of it goes through `/api/med/pathedit?op=...` (`reader_meditation.path_edit`).
  Switching is `/api/med/usepath`, which keeps the open's seed and lands on the
  path's first stop.
- **Contents bar:** under the whole contents list is a **Paths** list. Each
  path is collapsed to its title, with its spoken name in bold, plus All
  sections, and the one you're walking is marked. Pressing one walks it.
- **Voice** (Caster `text_meditation_commands.py`): say a path's name
  ("daily path", "short sit") to walk it, or "path <name>". "All sections" /
  "no path" walks everything. A path's voice names are its title plus `spoken`.
  Stop names jump like section names ("who's here"). Path names live in
  `INIDATA/VoiceChoices/text_meditation_paths.json`, refreshed by
  `voice-choices`, `stop-set` and `path-set`.
- All parts sections now use **random, favorites likelier** (`weighted`), so
  stops vary from one open to the next.
- Authoring guidance for Claude: `~/.claude/skills/gentle-lines/references/meditation-paths.md`.

## Roles and kinds (2026-11-13)

**Why.** Some lines are written to END a section. Arriving's "When you're
ready, let your attention turn inward, the way you'd turn toward someone you
love" finishes it: everything before it prepares the turn inward. A random
draw doesn't know that, and could pull three body lines and no mind.

**Roles** (per section, `sec["roles"] = {qid: role}`, absent = middle):
- `opener`: shown first in its block.
- `closer`: shown last in its block.
- `pinned`: always drawn, in its written place.

When a section has several openers or closers, one is drawn at random; scores
only tilt the odds, so a +2 opener is likelier, never certain. Openers,
closers and pinned lines count toward `show`. When there isn't room for
everything, the closer comes before the opener, and a block always keeps at
least one middle line.

**Kinds** (per section, `sec["kinds"] = {qid: kind}`). Vocabulary:
`text_meditations.py kinds`. They are senses, body, feeling, noticing, insight,
imagery, guidance, permission, ask (a question to a part) and offer
(something said to a part). Claude tags; Jamie corrects. Balancing is
**gentle**: each line already drawn of a kind multiplies the next one's odds
by `KIND_DAMP` (0.55). A mix is far likelier, but two of a kind can still
happen, so a draw never feels the same every time.

**In a stop**, each source keeps its own block in order (opener, middles,
closer). The first source may use its opener and the last its closer; a
source giving 3+ lines may use both, so a stop can close twice when it reads
right. Kind balance runs across the whole stop.

**Editing.** In editing mode only, each line's buttons start with a chip
naming its role and kind ("closer · feeling"); pressing it opens a pop-up of
role and kind chips that saves at once. Reading mode shows nothing. CLI:
`line-tag <slug> <section> <line> --role R --kind K` (line = position, quote
id or its tail) and `tag-batch <slug> --file f.json` (all or nothing).
`show <slug>` prints each line's tags.

**First tagging (same day).** Claude tagged all 108 lines of the parts
meditation. Every section has an opener; most have a closer. Opening the
door, Getting to know a protector, Being here and Closing each have two
closers, so their endings vary. No line is pinned yet.

## Not yet

- Adding a line from a stop page (add-line needs a section; add it on the
  section's own page for now).

- Editing a section's spoken names or title from the arrange page (CLI:
  `set-section --spoken --title`).
- More meditations: write a guide-shaped markdown, `import-guide`, or add
  `add-line` by hand.
