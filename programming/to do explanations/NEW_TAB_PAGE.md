---
tags: [programming, new-tab, dashboard, local-viewer, quotes, completion-log, projects, mindfulness]
created: 2026-03-25
updated: 2026-09-16
related: ["[[COMPLETION_LOG]]", "[[QUOTES_SYSTEM]]", "[[Personal]]"]
---

# New Tab Page

The dashboard Chrome keeps as a pinned tab: to-dos, rotation, check-ins, pacing,
quotes, and what Jamie is reading and watching. It was built 2026-03-25..27 as a
static GitHub Pages site (`JamieHooper-code/new-tab`, **archived 2026-09-15**).
Since 2026-09-15 it has been a **local view over the stores that already own its
data**, served by `AutoHotkey/Scripts/newtab/newtab_server.py` on `127.0.0.1:8293`.

## Why it moved (the 2026-09-15 audit)

- **It was never a real new tab override.** It was the last *pinned tab*, loaded
  from `jamiehooper-code.github.io/new-tab/`. The repo's `manifest.json` pointed at
  a `newtab.html` that never existed.
- **Pushing to GitHub silently did nothing.** On first load each panel copied its
  items into `localStorage` and never read `panels.js` again, so new text never
  reached the screen. Checklists did the reverse: a deleted item came back. That is
  why the page sat unchanged for six months.
- **Every panel duplicated a store that already existed.** To-dos → `projects.json`.
  Rotation → the completion log (`clog dashboard` was documented as "the one-shot
  new-tab payload" and never consumed). Quotes → a `newtab` display pool, also never
  consumed. Check-ins → a second hardcoded list in `Helpers/MindfulnessBellPoller.ahk`.
- **The `newtab` quote pool was the wrong axis.** It selected on tags → 175 quotes,
  mostly narrative passages from *Circe*.

## Decisions (Jamie, 2026-09-15)

| Area | Decision |
|---|---|
| Hosting | Local server on `127.0.0.1:8293`; pinned tab repointed; GitHub repo archived |
| Ctrl+T | **No.** A new tab owned by an extension takes keyboard focus away from the address bar |
| Boot | Started from `StartupOrder.bat`. Chrome restores the pinned tab at launch, so an on-demand start is too late |
| To-dos | The Programming and House panels read those `projects.json` workspaces; the page's own lists retired |
| Rotation | Outside, Stretches, Reading, Meditation (not swimming); checking one logs it |
| Check-ins | A `checkin` entry type in the quote store, imported only after line-by-line review |
| Bell | Reads the same `checkin` pool: buried lines never ring, upvoted ones ring more often |
| Quotes panel | **The curated `remember` group only** (see the rule below; renamed from `reminders` 2026-09-17 — that word now belongs to [[REMINDERS_SYSTEM]]) |
| Votes | ▲/▼ on every line of every pool panel, on the shared affinity scale |
| Novelty | **Pool panels are drawn, not listed:** a fresh weighted sample in a fresh order every time the page is looked at |
| Collapse | Every panel title is a link that collapses the panel. It stays collapsed across reloads and reboots |
| Editing | In-page editing stays, and writes to the owning store |
| Buttons | Resolved server-side per row; POST with an Origin check; the page never names a function |
| Links | **Every clickable thing is a real control** (`<a href>`, checkbox, `<label>`, `<button>`) so Vimium can hint it |
| Voice | Deferred to the very end |

## Content rules

**Check-ins: somatic and inspiring, never judgmental or managerial.** A check-in
invites a sensation or offers permission (*let your jaw soften*, *rest is how you
stay*). It never audits (*are you slouching?*) and never manages (*timebox 25
minutes*). Jamie's notes from the review, for every future line:

- **No assumed position.** She stands for long stretches and sits when she needs
  to. No chair, no keyboard, no hands on the keys.
- **Movement lines offer sitting, standing or shifting alike.**
- **Attention, not hand placement.** "Breathe toward where it hurts" happens in the
  mind.
- **Not "nothing to fix".** She is rarely stuck and often locked in.
- **"Small movement", not "small stretch".**
- **Pain and pacing lines** treat the body's signals as information, not a rule or
  a failure.

**Quotes: genuinely helpful while programming, or not at all.** No pages from
books, no political quotes, nothing that is only interesting. **Don't water it
down.** It is better to draft new lines and wait for good ones than to fill the
panel with near-misses. So the panel reads the `remember` group only, and a line
gets there only by her approval. A book highlight she approves goes in as its own
short reminder; the highlight stays with its book.

It deliberately does **not** pull in "anything upvoted anywhere". That was tried
and dropped the same day: an upvote in the reading room would have put book pages
here.

## Architecture

```
pinned tab → http://127.0.0.1:8293/
   ↓
Scripts/newtab/newtab_server.py       stdlib, on Scripts/viewers/local_viewer.py
   ├─ Scripts/newtab/web/             the page -- read per request; the tab reloads itself when these change
   ├─ INIDATA/newtab/storage.json     layout, display modes, collapsed flags, "Add Panel" panels -- nothing else
   ├─ SOURCES (20 s cache)            one per panel, each read through the owning store's CLI
   ├─ POST /api/write                 panel changes (incl. votes), built into that CLI's command
   └─ POST /api/action                AutoHotkey64.exe MAINFUNCTIONS.ahk <fn> <args>
AHK: Helpers/NewTabPage.ahk           OpenNewTabPage (start if dead + focus), RepointNewTabPinnedTab
Bell: Helpers/MindfulnessBellPoller   reads the `checkin` pool (see "The bell")
```

| Panel | Reads | The page can |
|---|---|---|
| Programming / House To-Do | `projects.py list programming\|house`, todos only, open first | tick, add, delete (with a confirm) |
| Reading | `clog book-current`, the same rows as `open read` | click → `ReadBookHere <id>` |
| Watching | `watchlist.py recent`, the same list as `open watch` | click → `WatchShow <url> <service>`; open-in-new-tab opens the show itself |
| Rotation | `clog last <template>` for each `ROTATION` row | log Outside or Stretches for today; Reading and Meditation are logged for her already, so they show "auto" |
| Check-In | pool `newtab_checkin` | ▲/▼; Edit: text, kind (body / breath / mind), add, delete |
| Pacing (was "Brain / Productivity") | pool `newtab_pacing` | ▲/▼; Edit: text, add, delete |
| Quotes to Remember | pool `newtab` (group `remember`) | ▲/▼; Edit: text, add, delete |

**The draw.** A pool panel shows a weighted sample: weight = 1 + score, and a
buried line (score ≤ -1) is never drawn (Efraimidis–Spirakis, in
`weightedSample`). A new draw happens:

- at load,
- each time the tab comes back into view,
- after an add, edit or delete,
- when the gear's mode or limit changes.

**A vote does NOT redraw.** Reshuffling under her cursor would move the next line
she meant to vote on, so the draw is held as a list of keys between redraws. Edit
mode lists every line with its score, buried ones included, which is where a
hidden line comes back.

**One list, two surfaces.** Reading and Watching run the exact commands the
Millers run, so the page and `open read` / `open watch` can never disagree. A
row's spoken word shows as a pill ("darkness", "blinders"). Panel ids never
change, which is why Pacing is still `brain` and kept its place and its Random-9
setting.

**Collapse.** The title is an `<a href>`. It toggles `panel_collapsed_<id>` in
`storage.json`, and CSS hides the body and gear. It is excluded from the title
bar's drag handling, the same way the Edit and gear buttons are.

**Every write is decided server-side.** Binding to 127.0.0.1 keeps other
machines out, not other websites: any page in Chrome can send a request to
localhost. So every POST:

- must carry the page's own Origin (`ViewerApp(allowed_origins=...)`), and a
  refusal is logged (`app.on_refuse`);
- names a panel, an operation and a row key, which the server checks against rows
  it read itself and against that panel's `writes`;
- can only use that panel's own groups or workspace;
- runs as an argument list, never a shell string. Actions also pass an `ACTIONS`
  allowlist.

**The bell.** `_MB_NextCheckIn` reads a cache of the `checkin` pool
(`%TEMP%\ahk_checkin_pool.json`). Buried lines are skipped, and each upvote adds
another turn in the rotation, capped at 5. When the cache is over an hour old, a
ring starts a detached `pythonw quotes.py pick --pool checkin --all --out <cache>`,
and the *next* ring picks it up by mtime. There is never a `RunWait`: the bell
lives in the always-on Parent process, where a Python start would freeze every
hotkey.

## Traps

- **Stale code.** The server holds itself and `local_viewer.py` in memory. After
  editing either, run `py Scripts/newtab/newtab_server.py restart`. Page files need
  no restart; the open tab reloads itself on its next return to view.
- **Vimium only hints real controls.** The first Reading and Watching rows were
  `<li>`s with click listeners: they got no hint. Anything clickable must be an
  `<a href>`, checkbox, `<label>` or `<button>`, and must never be hidden. Vote
  buttons are dim, not invisible.
- **Never pass a URL through `MAINFUN.bat`.** cmd reads `&` as a separator and
  expands `%...%`. The server runs `AutoHotkey64.exe MAINFUNCTIONS.ahk` directly.
- **Read a POST body before refusing it.** A keep-alive connection leaves an unread
  body in the socket, and the next request parses from it (a 501). Fixed in the
  shared shell.
- **pythonw has no stdout.** `affinity.py` calls `sys.stdout.reconfigure()` on
  import, so every windowless `quotes.py` launch died silently until the guard at
  the top of `quotes.py`.
- **A tag pool is slow** (~7 s, because it expands inherited tags). Surfaces select
  by `entry_types` or group (~0.25 s).
- **An empty pool exits 1.** `fetch_pool` treats that as zero rows, not a failure.
- **An unreadable `storage.json` stops the server** rather than being emptied on
  the first click.
- **`panel_order` is retired** (2026-09-16). `panel_layout` replaced it, and the
  first drag removes the old key.
- **Hidden tabs get no frames.** A background Chrome tab and the Claude browser
  pane get no animation frames and no ResizeObserver callbacks. So the page
  uses timers, not `requestAnimationFrame`, to reveal the board and handle
  resizes, and the height hold looks broken in the pane when it isn't. Test
  layout with `py Scripts/newtab/check_layout.py`, which runs headless against
  the live server and answers every POST itself, so it never writes.

## Phases

1. ✅ **Local, same look, plus Reading and Watching.** Pinned tab repointed, repo
   archived.
2. ✅ **To-dos and Rotation** from `projects.json` and the completion log.
3. ✅ **Check-ins in the quote store.** 39 check-ins, including the 7 reworded ones
   approved 2026-09-15. The bell reads the pool.
4. ✅ **Votes, weighted draw, collapse.**
5. 🟡 **Grow the Quotes panel to 25–100.** 16 lines after round 1, 42 after
   round 2 (both 2026-09-15). New drafts go through the same review.
6. ⬜ **Voice**: a `newtab` context; "check N" / "done <template>" as API calls.
7. ✅ **Stable layout** (raised and built 2026-09-16). Every redraw changes the
   text, so panel heights change and the layout reshuffles each time she comes
   back. Either fix panel sizes and fit the text inside, or make the packing
   move less. The current packing is an old hack, so look at it fresh.
   She still wants panels to size to their content. What she doesn't want is
   small size changes moving things, so she can learn where things are.

   **Diagnosis, 2026-09-16.**
   - `#board` uses CSS multi-column (`column-count: 4`). The browser balances
     column heights and flows panels top-to-bottom, then left-to-right, so a
     panel's column depends on the heights of every panel before it.
   - At 1920 px wide, 8 reloads produced two layouts, alternating 4 times.
     - Layout A uses all 4 columns.
     - Layout B moves Watching under Reading. That shifts Rotation, Pacing,
       Check-In and Quotes one column left and leaves column 4 empty.
   - The trigger is small. Check-In's height ranged 519–641 px and Quotes
     269–350 px, depending on which lines were drawn.
   - Collapsing a panel changes its height by hundreds of px, so it reshuffles
     the same way.
   - Panels also paint empty, then fill as each fetch returns, so the page
     reflows while it loads.

   **Decisions (Jamie, 2026-09-16).** The rule is that she owns the column and
   the content owns the height.
   - **Explicit columns.** The saved layout is a list of columns, each an ordered
     list of panel ids; it replaces `panel_order`. Only her drag moves a panel
     sideways. A height change nudges only the panels below it in the same column.
   - **Grow now, shrink reluctantly.**
     - A panel grows at once, so text is never clipped.
     - A redraw she didn't cause that comes out shorter keeps the old height,
       as bottom padding, unless it loses more than max(60 px, a quarter of
       the panel).
     - She approved "about one row". Built with one row first, but the numbers
       said no: a drawn row is 32–72 px, and Check-In's natural height runs
       539–641 px, so Quotes would still have moved on most redraws. With a
       quarter, Check-In settles at its tallest draw and holds; in 12 headless
       redraws, Quotes moved twice early on and then not again.
     - Anything she does inside a panel (pointer, key, input) frees it for 4 s,
       so her own changes, like collapsing it or ticking a to-do, land at once.
     - This is kept in memory, not saved. A full reload starts from natural
       heights.
   - **Hide the board until every panel's first load has landed.**
   - **Seed from today's 4-column layout A.**
     - Column 1: Programming, House, Reading
     - Column 2: Watching, Rotation
     - Column 3: Pacing
     - Column 4: Check-In, Quotes
   - **Narrow windows: fold, don't rearrange.**
     - She is often at half screen. At about 1280 px wide, 3 of today's 320 px
       columns fit.
     - She chose 3 columns at half screen over 4 squished ones.
     - At 3 columns, column 4 stacks under column 3.
     - At 2 columns, columns 1+2 stack into the left column and 3+4 into the
       right, so what's on the left at full screen stays on the left.
     - At 1 column, all four columns stack in order.
   - Not chosen: sticky auto-balancing, which is still unpredictable;
     fill-to-height pools; a separate layout per width.

   **Built, 2026-09-16** (`web/app.js`, `web/index.html`).
   - `#board` is a flex row of `.column` elements.
   - Each panel carries `data-col`, its layout column. `arrangeColumns()`
     moves panels (never rebuilds them) into the on-screen columns using
     `FOLDS`.
   - `saveLayout()` reads the columns back in document order.
   - A drop lands in the column under the pointer, before the first panel
     whose middle is below it, and takes that neighbour's `data-col`. So a drop
     at half screen saves the right layout column, and the gaps and the space
     under a short column are drop spots too.
   - The column count is `floor((board + gap) / (--g-col-width + gap))`,
     capped at 4: 1920 → 4, 1280 → 3, 960 → 2.
   - The board has `class="loading"` (`visibility: hidden`) until every live
     panel's first load resolves, or 2.5 s passes.
   - `Scripts/newtab/check_layout.py` passes all 6 sections.

   **Aside, same day.** The old GitHub page was open in Chrome again: the
   `testing_testing` context matched its URL at 12:58 and several times after.
   `RepointNewTabPinnedTab` moved it (URL verified) at 16:42. Two stale pointers
   were also fixed:
   - `INIDATA/VoiceChoices/sites.json` "testing testing" pointed at GitHub. It
     now points at `http://127.0.0.1:8293/`, which Caster picks up on its next
     reload.
   - `INIDATA/Contexts/testing_testing.json` had `url_contains` set to the
     GitHub host. It is now `127.0.0.1:8293`, applied by `ReloadWithNotice`.

## Quotes panel review, 2026-09-15: what she kept, and what it says

**Approved (12):**

- **Moved into `reminders` (now `remember`) from her store**, reworded as approved:
  - *You cannot bully a seed into growing.*
  - *Your brain doesn't even have arms…*
  - *Whatever I am, let it be enough.*
  - *I am happy, but later I won't be…* (Babish)
  - *Life is very short. I think it is cool to love stuff.* (Brennan Lee Mulligan)
- **Copied from book highlights**, with each highlight left in place:
  - *When action grows unprofitable, gather information; when information grows unprofitable, sleep.* (Le Guin)
  - *Parts are little inner beings…* (Schwartz)
  - *Like the sun, the Self…* (Schwartz)
- **New:**
  - *You don't have to finish today for today to count.*
  - *Stopping mid-thought is safe…*
  - *Momentum is kind, but it isn't in charge.*
  - *Working code can wait five minutes for a body that needs them.*

**Declined (9):**

- *Expectations are like fine pottery* (Sanderson)
- *The bug is not a verdict on you*
- *Curiosity outlasts frustration*
- *Hard problems get smaller after sleep*
- *Confusion is what learning feels like from the inside*
- *Every system you've built started as something you didn't know how to do*
- *Your worth isn't measured in commits*
- *You're allowed to be proud of the tools you've made*
- *The insight often arrives the moment you look away*

**The pattern, for the next round of drafts.** Two kinds of line landed:

- **Permission to stop, pace and come back:** finish today, stopping mid-thought,
  momentum not in charge, the body before the code.
- **Perspective on her own mind:** parts keeping her safe, the Self obscured, the
  brain with no arms.

What was declined is **pep talk about the work itself**: reassurance about bugs,
confusion, commits, and pride in tools. Write toward the first two kinds, not
the last.

## Quotes panel review, round 2, 2026-09-15

She asked for more in the kept style plus some variety. 32 drafts, 26 approved,
all added to `reminders` (now `remember`) as manual quotes (42 lines in total).

**Approved by style:**

- **Stopping and pacing:** allowed to stop while it's going well · nothing set
  down forgets how to be picked up · tired is information · "later" is a real
  place · leave the door open and walk away · stopping on time · enough for now.
- **Her own mind:** "hurry" never on time · overwhelm as a lot of you showing up ·
  the worried part can come along without steering · loud because it cares.
- **Playful:** the to-do list has no legs · *You are a very clever animal, and
  animals need rest* (her edit: the draft said "water").
- **Nature:** rivers go around · the garden takes the winter off · tides don't
  apologize · water is patient.
- **Warmth:** the people who love you aren't waiting for the finished version ·
  joy isn't a reward for finishing · you don't have to earn a rest.
- **Questions:** urgent or just loud · what you'd say to a friend · what would
  make the next hour gentler.
- **Very short:** go gently · soft eyes, long view · unhurried is a speed.

**Declined (6):** a thought is a visitor, not a landlord · something in you is
already calm · snacks are a legitimate strategy · nobody wished they'd skipped
the fun part · winter isn't the tree failing · loving something is never wasted.

**What this adds to the pattern.** The variety landed: nature images, questions
and very short lines all worked. The misses were the more familiar,
poster-shaped lines (visitor/landlord, already calm) and the ones that lean on a
negative to reassure (the tree *failing*, *never* wasted). Joke lines work when
they point at rest (animals need rest) more than at treats or fun.

## Backlog (the original wish list)

- [x] Check-off tasks that reset counters → Rotation panel
- [ ] User profiles with Google integration
- [ ] Image debugging workflow
- [ ] General settings page
- [~] Panel for current YouTube/tabs/activities → Watching panel built; tabs/activities open
- [ ] Make it work from anywhere (at least speech) → phase 6
- [ ] Poem/quote of the day dynamically pulled from external sources
- [ ] Weather panel
- [ ] Panels as easy-to-add options across boards
- [x] Recurring event/task panel with reset-on-check → Rotation panel
- [x] Portable to-do list sourced from elsewhere → projects.json
- [ ] Nested checklists (click one to reveal sub-tasks) → projects.json already nests; the page shows one level
