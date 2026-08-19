---
tags: [programming, books, kindle, automation, ahk, caster, listnav, uia, design-doc, tagging, media]
---

# Anna's Archive pipeline — one `jump N`, book on the Kindle

**Authoritative doc.** Read before touching `Helpers/AnnasArchiveFunctions.ahk`,
`Scripts/anna_download_watch.py`, `Scripts/anna_metadata.py`,
`INIDATA/Contexts/anna_s_archive_results.json`, or the tag-provenance rules below.

Related: [[MEDIA_SYSTEM]] · [[COMPLETION_LOG]] · [[QUOTES_SYSTEM]] ·
`INIDATA/Contexts/README.md` (the ListNav profile system) ·
`skills/ahk-functions/references/uia-clicking.md`

---

## What it does

Say a search (`book <title>` → the existing `searches.json` entry), then **`jump N`**
on the results page. That single command runs the whole chain:

```
jump N ──▶ click result N in a NEW TAB          (search page survives)
       ──▶ wait for the detail page              (slow-server link = ready signal)
       ──▶ "Read more…" → capture ~28 metadata fields → sidecar (md5-keyed)
       ──▶ arm the detached watcher              (pythonw, outlives this process)
       ──▶ Slow Partner Server #1
       ──▶ wait out the site's countdown         (~17s, NOT circumvented)
       ──▶ "Download now" → confirm Save As
       ──▶ MAINFUN process EXITS (~40s total)
              ⋮ watcher runs on alone
       ──▶ file lands in E:\Downloads            (waits up to 5 min)
       ──▶ kindle_import.py   → library + auto genre tags
       ──▶ anna_metadata.py   → merge the captured record + promote tags
       ──▶ send_books_to_kindle.py → Kindle
       ──▶ tooltips at each stage
```

Verified end to end: ~16s from `jump 1` to "imported and sent" on a fast server.

## The pieces

| Layer | File | Owns |
|---|---|---|
| Entry / row click | `INIDATA/Contexts/anna_s_archive_results.json` | ListNav profile: which links are rows, new-tab click, the post-click hook. **No code.** |
| Click-through | `Helpers/AnnasArchiveFunctions.ahk` | `AnnaGrabBook` (the whole flow), `AnnaDownloadBook` (server → countdown → download → Save As), `_AA_CaptureMetadata` (the sidecar) |
| Detached tail | `Scripts/anna_download_watch.py` | wait-for-file → import → metadata → send, + on-screen tooltips |
| Metadata | `Scripts/anna_metadata.py` | parse the captured record, merge into `library.json`, promote tags, `--survey` |
| Existing, reused | `kindle_import.py` · `send_books_to_kindle.py` · `BookManagerMenu.ahk` | ingest, delivery, review queue — **the back half already existed** |

## Design decisions (the why)

**The profile carries the flow, not the engine.** `jump N` is the generic ListNav
verb shared with Instagram/Teams/Gmail/Spotify. Nothing site-specific went into
`ListNavFunctions.ahk`: the profile names `post_click_function: "AnnaGrabBook"`
and the existing hook invokes it. Blast radius zero — a profile without the field
no-ops. (The hook previously fired only on the grid path; it now fires on the
enumeration path too, which is a fix, not a special case.)

**Height alone identifies a row.** Every result is one tall link (809×111 compact,
96×165 covers) and every other link on the page is ≤37px. So
`row_selector: {Type: Hyperlink, min_height: 100}` — no width filter, which is why
the SAME profile works on the compact search view AND the cover-grid list view, and
survives window resizing. Don't add a width filter.

**Capture BEFORE download.** The metadata panel only exists on the detail page;
clicking a download link navigates away and it's gone forever. So capture runs
first, writes a sidecar, and the watcher joins on it later. **The md5 is the join
key** — it appears both in the panel (`MD5 f9dc…`) and inside AA's filename
(`-- f9dc579c… --`). No guessing.

**The tail is detached.** A 5-minute download must not hold an AHK dispatcher open.
`pythonw` + `Run(..., "Hide")`, mirroring `_BMArmWatcher`. The watcher is armed
BEFORE the download starts: its "new file" cutoff is set when it launches, so
arming afterwards would race a fast download and miss it.

**Readiness is polled, never slept.** The slow-server link's existence IS "the
detail page loaded". The download link's appearance IS "the countdown is served".
No fixed `Sleep` anywhere. The site's ~17s wait is served in full — nothing here
circumvents a rate limit.

**Completion is a real test, not a timer.** A file counts as arrived only once it
carries a book extension AND its size has been stable for 3 consecutive polls
(Chrome's in-flight `.tmp`/`.crdownload` never carry one). Three, not two: a
stalled transfer can look stable for a moment — the same easing-plateau lesson
`_LN_FollowAutoScroll` learned.

## Gotchas that cost real time

**Tag provenance is a CORRECTNESS rule, not bookkeeping.** Every writer owns exactly
one `src` and rewrites only its own:

| `src` | Writer | Rewritten by |
|---|---|---|
| `auto` | `kindle_import` genre lookups | `--retag` regenerates these |
| `annas` | `anna_metadata.py` | nobody else |
| `manual` | Jamie | `book-set-tags` |

These tags were originally written as `auto` — which collided, because
`retag_books` keeps only what it doesn't own and regenerates the rest. **A routine
`--retag` silently deleted every Anna's-Archive tag in the library.** Both filters
now keep everything they don't own (`!= "auto"` / `!= "manual"`).

**`--retag --no-net` strips tags.** `genre_tags` skips the Google Books/OpenLibrary
lookup entirely without the network, so a no-net retag blanks every book whose tags
came from there. Always run `--retag` with the network.

**Key parsing: longest match first.** Panel entries are ONE flat string with no
delimiter (`"Filesize 305904"`), and several keys are prefixes of others:
`Z-Library` vs `Z-Library Source Date`, `Nexus/STC` vs `Nexus/STC Source Updated
Date`, `Libgen.li File` vs `Libgen.li fiction_id`. Splitting on the first space
writes `key="Z-Library", value="Source Date 2023-03-09"` **and reports success**.
Match against `KNOWN_KEYS` longest-first. Values are always LISTS — a record with
three source collections genuinely has three `Filepath`s and three `IPFS CID`s.

**Never invent vocabulary.** `canon_tag()` returns unknown tags *unchanged* (so they
stay addressable), so its output is NOT proof a tag exists — gate on
`get_record()`. Without that, `thrillers` gets auto-created next to `thriller` and
quietly fragments a curated vocabulary. Unmapped categories are preserved as
`annas.unmapped_tags` and surface as **suggested** in the tag picker.

**Unknown keys are preserved, never dropped.** AA pages differ by source collection,
so an unknown key is expected. Everything unparsed lands verbatim in
`annas.unparsed`. `py Scripts/anna_metadata.py --survey` reads across every stored
record and ranks the missing keys by evidence — that's how `KNOWN_KEYS` grows
(from 2 records it already ranked `OCLC` ×61, `Open Library` ×9, `EBSCOhost` ×5).

**AHK→Python handoff must be `UTF-8-RAW`.** A BOM breaks `json.load` on the far side.

**`&` in a URL.** Only the *query* crosses the MAINFUN bridge — the template stays
in `searches.json` on the AHK side. Don't pass a full `&`-laden URL through a
`.bat` argument.

**Source metadata is often junk.** Search result #1 is not the best-catalogued
record. One had the title repeated in the author slot and no year/publisher, which
filed it under a title-named folder. The parser was correct; the record wasn't.
`clean_author` was also fixed here: `"Doyle, Sir Arthur Conan"` fell through both
branches and returned a bare `"Doyle"` (only 1 of 158 author folders had ever hit
this).

## Where the data lives

- Sidecars: `%TEMP%\anna_meta\<md5>.json` (deleted on successful merge)
- Books: `E:\Media\Books\<author>\library.json` → the book's `annas` block
- Downloads: `E:\Downloads` (what `kindle_import.py` scans — Chrome already
  defaults there, so no path plumbing)
- Alias backlog: `INIDATA/tag_alias_backlog.json` (see below)

## Reading the logs

The whole pipeline is one greppable story:

```bash
grep "Anna/" ahk_event.log          # grab · meta · download · saveas · watch
```

## The tagging backlog (`Scripts/tag_backlog.py`)

Tagging splits into two problems that look alike and are not. **Mechanical**
(`thrillers`→`thriller`) a rule derives — already handled by the alias index.
**Semantic** (`political`→`theory`) NO rule derives; it's a fact about Jamie's
taxonomy. Every failure here came from asking the deterministic layer to answer
semantic questions.

So: collect unknown tags, let the local model PROPOSE from the existing vocabulary,
Jamie decides in batches. `scan` / `propose` / `list` / `accept` / `accept-all` /
`promote` / `reject`. Writes go through `clog book-tag-alias` (aliases AND folds
across the library) or `quotes.py vocab-add` — **never hand-edit `media_tags.json`**.

**The guard that makes the LLM safe:** it picks from a CLOSED LIST and any answer
not in it is discarded, not trusted. It cannot hallucinate `subjects` into the
taxonomy because `subjects` isn't on the menu. Task `tag_alias` in
`local_llm/tasks.json`, same path as `quote_tag`/`quote_facet`. On the first real
run it declined 9/10 and every decline was correct — conservative by design
("prefer none over a weak guess"), and a `none` is the signal to `promote` a tag
rather than alias it (that's how `cosmere` (28 books) and `tolkien` (7) became
vocabulary instead of being folded into `fantasy`).

Aliases are permanent, so the backlog SHRINKS as the vocabulary grows —
amortisation, not a treadmill. Trigger a review on **backlog size** (~15 pending),
not book count.

## Gotchas (fixed, keep an eye out)

- **`[Author]` brackets in filenames broke the send (fixed 2026-07-28).** AA
  filenames are `Title -- Author [Author] -- … -- Anna's Archive.epub`. The auto
  send path (`send_books_to_kindle.py --send-files-from`) located each file with
  `base.rglob(name)` — but `rglob`/`glob` treat `[...]` (and `*`, `?`) as pattern
  metacharacters, so a bracketed name matched **nothing**, `send_specific` found
  no target, and it exited **0 with "none found to resend"** — the pipeline
  logged `send: ok` and popped a "Sent to Kindle" tooltip while emailing nothing.
  Symptom: book imports + tags fine, is stamped in `library.json`, but never
  arrives on the Kindle and never appears in `.sent_manifest.json`.
  Fix: `_find_by_name()` does an exact-name walk (no glob) instead of
  `rglob(name)`; applied in both `send_specific` and `stamp_library_name`.
  Non-bracket names (e.g. "Passing -- Nella Larsen -- …") were unaffected, which
  is why it looked intermittent. If a send silently no-ops, check for glob
  metacharacters in the filename first.

## Batch retry — when a whole session's downloads fail at once

Added 2026-07-30, after the external drive holding `E:\Downloads` dropped off USB
mid-session. Nine click-throughs all succeeded — every Save As confirmed — and
every file went nowhere, because Chrome could not write to a disk that was
returning `WinError 483` (fatal device hardware error). The pipeline was never
at fault, so the fix is simply to replay it.

| Half | File | Owns |
|---|---|---|
| Click-through | `AnnaRetryDownloadsFromList(listPath)` in `Helpers/AnnasArchiveFunctions.ahk` | Replay N detail pages from a UTF-8 list of URLs, one per line (default `%TEMP%\anna_retry_urls.txt`) |
| Back half | `Scripts/anna_batch_import.py` | Wait for the batch to land, then import ONCE, merge metadata per md5, send in one call |

```bash
MAINFUN.bat AnnaRetryDownloadsFromList "%TEMP%\anna_retry_urls.txt"
py Scripts/anna_batch_import.py --expect 9
```

**Why a batch needs its own back half — the watcher does not compose.**
`anna_download_watch.py` is one-book-per-process: it returns the *first* file
that stabilises, imports it, sends it, exits. Each instance shells
`kindle_import.py`, which writes `library.json`. That is correct for the
interactive `jump N` flow, where books arrive minutes apart. Fire nine within
two minutes and you get nine concurrent writers to the same JSON file. So the
batch path arms **no** watcher at all: download everything, then run the
importer once, sequentially. `AnnaRetryDownloadsFromList` deliberately omits
the `_AA_ArmWatcher()` call that `AnnaGrabBook` makes — that omission is the
entire difference between the two functions, and it is load-bearing.

**Recovering the list after a failure.** The md5 sidecars in
`%TEMP%\anna_meta\` survive a failed download — capture happens before the
click, so a session that downloaded nothing still leaves nine records. Rebuild
the retry list from them rather than from Chrome's download history:

```python
urls = [f"https://annas-archive.gl/md5/{json.load(open(f))['md5']}"
        for f in glob.glob(os.path.join(os.environ['TEMP'], 'anna_meta', '*.json'))]
```

Prefer the sidecar's `md5` over its `url` field — `url` comes from
`ChromeCurrentUrl()` at capture time and can lag a redirect, whereas the md5 is
read straight off the detail page's `AA Record ID` row.

**AA serves ONE download at a time — and a refused download looks exactly like
the drive failure.** The first batch run fired all eight click-throughs
back-to-back. Books 1 and 2 landed; 3, 4 and 5 were refused because the
previous transfer was still running, and the run never reached 6-8. The
dangerous part is the symptom: the slow-server link is found, the countdown is
served, "Download now" is clicked, the Save As is confirmed, and the log reads
`ok - download confirmed` — and no file ever appears. That is byte-for-byte the
same signature as writing to a dead disk, so it is **not** diagnosable from the
Anna/ log alone. Always confirm against the actual file landing in
`E:\Downloads`.

`_AA_WaitForDownloadsIdle()` is the fix: between books, block until Chrome has
no `.crdownload`/`.tmp` in the downloads root. Absence of both is "idle".
It sleeps 2s first — checking immediately reads "idle" on a transfer that has
not started yet. Never remove the wait to make a batch faster; the speed is not
real, the downloads just fail silently.

**New tab, not in-place navigate.** The retry opens each detail page with
`OpenChromeTabs`. After a download the tab has moved to the partner-server host,
so there is no reliable same-host tab for `OpenOrNavigateChromeTab` to reuse —
and Chrome's UIA exposes tab *names* but not URLs, so host matching is not
dependable here anyway.

- **An oversized book failed at SMTP instead of up front (fixed 2026-07-30).**
  `find_new_books` has always split its results into `new` and `oversized`
  (`max_bytes`, default 24MB), but the `--send-files-from` path did not: it went
  straight to `batch_books()`, which GROUPS by size and cannot split a single
  file that is over the cap on its own. So one 173MB book (*The Book of
  Wilding*, a heavily illustrated title) became its own batch and sailed into
  SMTP, where Gmail answered `552 5.3.4 message exceeded size limits` — an error
  naming neither the book nor the real problem, arriving at the very end of a
  download → import → tag pipeline that had otherwise fully succeeded.
  Fix: `send_specific` filters oversized targets **before opening SMTP**, prints
  each one with its actual size vs the cap, and stamps it.
  Note the ceilings — Gmail attachments cap at 25MB and Amazon's Send-to-Kindle
  address at 50MB, so a book this size can never be e-mailed by any route; it
  needs the Send to Kindle **web uploader** (up to 200MB) or USB.

- **`send_status` gained a third value, so the "sent" test had to tighten.**
  `_stamp_library_entry` set `sent_to_kindle = (status != "failed")`, which was
  equivalent while `sent`/`failed` were the only states. Adding `oversized`
  broke that: a book that was never even attempted would have been stamped as
  delivered. Now `(status == "sent")` — only a real send counts.

- **The oversized queue is durable, and both doors write it.** Books too big to
  e-mail carry `send_status: "oversized"` in their `library.json`, stamped from
  the scan path AND the resend path (one store, whichever door you came
  through). `py Scripts/send_books_to_kindle.py --list-oversized` reads that
  stamp — so the list is what is still *owed to the Kindle*, not merely
  everything currently over the cap — and prints `MB<TAB>title<TAB>filename` for
  a menu to consume.

## The three ways a big book went missing (fixed 2026-08-06)

A session of large nature books exposed three separate bugs that all presented
the same way: *the book is on disk, and the system says everything is fine.*

**1. The watcher gave up on slow downloads.** `--timeout` was a fixed 420s total
budget, which cannot distinguish "this is a 100MB book still arriving" from
"this download is dead". A 101MB encyclopedia finished at 16:08:44, ninety
seconds after its watcher quit at 16:07:05 — so nothing imported it, and the
only evidence was a file sitting in `E:\Downloads` looking perfectly downloaded.
Fix: **waiting is bounded by PROGRESS, not elapsed time.** Any growth in an
in-flight `.crdownload` resets the clock (`inflight_sizes()`); the watcher gives
up only after `--timeout` seconds with *nothing moving*, backed by a `--max-wait`
hard cap (default 90 min) against a transfer that trickles forever. `--timeout`
is now an IDLE limit — do not read it as a total budget.

**2. "Sent to Kindle" was announced for books that were never sent.**
`send_specific` returned `None`, so `main()` exited 0 whether it e-mailed nine
books, found none of them, or refused an oversized one. The watcher saw exit 0,
logged `send: ok`, and popped a *"Sent to Kindle"* tooltip for a 68MB book that
went nowhere — the send stage took 0 seconds, which is the tell. Fix: explicit
exit codes `SEND_OK=0` / `SEND_FAILED=1` / `SEND_OVERSIZED=3`, and callers now
branch on them to say the true thing ("TOO BIG to e-mail — Book Manager > Too
big to e-mail"). **Silence is not success**: a stage that delivers nothing must
never exit 0.

**3. Two watchers imported and sent the same book.** Every `jump N` arms its own
watcher, and all of them poll the same directory for "a new book", so two alive
at once both claim whichever file lands first — one book was imported twice and
e-mailed twice within 15 seconds. Fix 1 makes overlap the norm rather than the
exception, so `claim()` now gates the back half: an `O_CREAT|O_EXCL` marker in
`%TEMP%\anna_claims\` lets the filesystem pick the winner, with no check-then-take
window. Claims older than 3h are treated as abandoned so a crashed watcher
cannot strand a book permanently.

**Trap when touching these:** `run()` in both `anna_download_watch.py` and
`anna_batch_import.py` returns an **exit code**, not a bool. `if not run(...)`
now reads as *"if it worked"* — every call site must compare to `0` explicitly.
That inversion is silent and would flip the entire pipeline's error handling.

## Known gaps / next

- **Row-action `N.M` → Edit tags** still opens the old two-pane picker; drill-ins
  use the new control. Real fix = `layout: "checklist"` opt on
  `_PersistentLoopPickGui` (see [[COMPLETION_LOG]] polish items).
- **`KNOWN_KEYS` is short.** `--survey` already ranks the candidates: `OCLC`,
  `Open Library`, `EBSCOhost … Subject`, `ISBN-10/13`, `LCC`, `ASIN`. Note
  `Open Library Subject` and `EBSCOhost … Subject` are better tag sources than the
  EPUB subjects the importer currently uses.
- **9 backlog items** pending Jamie's call (`body image`, `chinese`, …).
- **The standalone Book Manager** still exists under More; the Reading node
  duplicates its day job. Retire once the node has proven out.
- The pipeline takes the FIRST slow server; no fallback if it's down.
