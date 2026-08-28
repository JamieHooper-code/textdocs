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
       ──▶ the next slow partner server in turn   (#1, #2, #3, #4, #1, …)
       ──▶ wait out the site's countdown         (~17s, NOT circumvented)
       ──▶ read the printed url → aria2 queue → import on completion
            └─ (no url on the page? click "Download now" — fallback only)
       ──▶ MAINFUN process EXITS (~40s total)   tail is detached either way
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
| Fetch + queue | `Scripts/anna_fetch.py` | aria2 hand-off, the one-at-a-time queue, link-expiry checks |
| Click-through host | `Scripts/AnnaGrabHost.ahk` | own process, so a tooltip cannot kill a grab |
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

## Server rotation — taking them in turn

`jump N` used to hand every book to **Slow Partner Server #1**. One server carried
the whole night's downloading, and a waitlist there stalled everything behind it.
The servers now go **in turn** — #1, #2, #3, #4, #1, … — one per download.

**The pool is read off the page, not hardcoded.** AA sorts its slow servers into
two tiers and says which is which in the row text:

```
Slow Partner Server #1 (slightly faster but with waitlist)    <- rotate these
Slow Partner Server #5 (no waitlist, but can be very slow)    <- skip these
```

`_AA_PoolFromRows` keeps a row that mentions a waitlist and is *not* the
"no waitlist" tier. Matching the LABEL rather than the numbers 1–4 means the
rotation follows the site: if AA moves a server between tiers, the pool moves
with it instead of silently going stale on a number that used to be right.
`_AA_DefaultServerPool()` (`[1,2,3,4]`) is only the fallback for when the list
cannot be read at all.

**"no waitlist" contains "waitlist".** A naive `InStr(name, "waitlist")` sweeps
the very-slow tier straight into the rotation, and the only symptom is downloads
that take an age — which reads as a bad night on AA, not a bug. Both halves of
the test are pinned against the site's real wording in
`Helpers/Tests/test_anna_server_rotation.ahk`.

**The FAST tier is a membership feature.** `Fast Partner Server #1–14` sit in the
same list and are never eligible — clicking one without a membership downloads
nothing at all.

**The cursor is a file, because there is nowhere else.** Every MAINFUN call is a
fresh process, so anything in memory dies with the download that set it:
`INIDATA/anna_server_cursor.json` holds `{"last_server": N, "updated_utc": …}`.
It stores the server NUMBER, not an index into the pool, and `_AA_NextInPool`
advances by POSITION — so a pool that is not `1..N` still steps one and wraps,
and a cursor pointing at a server that has left the tier restarts cleanly rather
than sticking. A corrupt or missing cursor restarts the rotation; losing our
place is not worth refusing to fetch the book.

**It advances on SELECTION, not on success.** The cursor is written before the
click-through runs, so a jammed server hands the next book to the one after it
instead of being retried forever.

**Forcing one server:** `AnnaDownloadBook(6)` (or `MAINFUN.bat AnnaDownloadBook 6`)
uses exactly that server and leaves the cursor alone. `0` / no argument = rotate,
which is what `AnnaGrabBook` and the batch retry both do.
`MAINFUN.bat AnnaShowServerRotation` reports the pool, the last server used and
the next one, without downloading anything — run it on a detail page.

## Every page prints the url — take it

Once the wait is served, **every partner page prints the download url as
copyable text**. AA uses two layouts and both carry it:

```
server #1:  [link] "📚 Download now"        server #2:  "To download, copy this URL
            [link] "Download with short filename"                and then paste it in your
            [button] copy                                        browser's URL bar:"
            [text]  https://wbsg8v.xyz/….epub                  [button] copy
                                                                 [text]  http://45.3.63.28:6060/….epub
```

So there is no "which shape did this server give us" question: **the url is the
path on every server**, and the "Download now" link is a fallback for a page
that somehow has one and no url.

**This shipped backwards, and it was silent.** The first version checked for the
link first and returned on it — so server #1, whose page carries *both*, was
reported as having no url, and its books went down the Chrome path with no
retry, no resume and no queue slot. Nothing errored; the pipeline just used the
worse half of itself for some servers. The ordering is now pinned by
`_AA_ReadyFromSignals`, which exists purely so the precedence rule can be tested
(`test_anna_download_choices.ahk`).

**Why the url beats the link even where both exist:** a link hands the transfer
to Chrome, which cannot retry a 502, cannot resume a dropped transfer, cannot
tell us when it finished, and — most importantly — cannot be queued. Since AA
allows one download at a time across all servers, everything has to go through
one queue, and only a url can.

**Hosts differ per server** (`https://wbsg8v.xyz` on #1, `http://45.3.63.28:6060`
on #2 — different scheme, too). That is exactly why the one-at-a-time rule is
global rather than per-host: four books from four servers are still four
sequential downloads.

**We read the url, we do not press "copy".** The page has its own copy button and
using it would be the obvious move, but the clipboard is not ours to spend —
Jamie's named clipboard slots live there (`CopyPasteManager`). The full url is
already exposed as the text node's UIA `Name`.

**Take the LONG url.** Where a page prints two, the second is offered as *"Use
this URL to get a shorter filename when saving"*:

```
…/A%20psalm%20…%20--%20c7e49af174931d9f205f0e10c394b1db%20--%20Anna’s%20Archive.epub    (439 chars)
…/annas-arch-c7e49af17493.epub                                                    (154 chars)
```

The metadata merge joins the captured sidecar to the imported file on the
**32-hex md5 in the filename** (`MD5_RE` in `anna_download_watch.py`). The short
name carries only the first **12** characters, so taking it would silently skip
the merge — the book lands stripped of everything the detail page knew about it,
with nothing but a `no md5 in filename` line in the log.
`_AA_PickDownloadUrl` prefers the md5-bearing url regardless of page order.

**What separates a url from prose.** `^https?://\S+$` plus a book extension at
the end. Both halves earn their keep: the page also prints the file's library
PATH, which ends in `.epub` but is not a url, and the site's own links are urls
but do not end in a book extension. The whitespace-free test matters too — AA
percent-encodes, and a "url" with a real space in it is prose that happens to
end in `.epub`.

## Fetching with aria2 — the queue

Chrome downloads the file on the "Download now" path. On the printed-url path it
goes to **aria2** instead (`Scripts/anna_fetch.py`), which is where the pipeline
stops being flaky.

**The partner servers really are unreliable, and this is measured, not felt.**
Probing one live url over five minutes returned, in order:

```
429 Too Many Requests
200 OK
502 Bad Gateway
502 Bad Gateway
200 OK          ← curl --retry got through; nothing else changed
```

Chrome has no answer to that. A failed transfer is simply a file that never
appears, and the watcher times out saying nothing useful. aria2 retries and
**resumes**, so a 502 costs ten seconds instead of a book. The server sends
`Accept-Ranges: bytes` with a stable `ETag`, so resume genuinely works.

**Completion stops being a guess.** The watcher decides a download has finished
when the file's size has not changed for three polls. aria2 exits `0` when the
file is complete. That is the whole reason the back half moved into
`anna_download_watch.import_and_send()`, reachable as `--complete <path>`: there
is now ONE import chain entered two ways — by the watcher's guess (Chrome) or by
aria2's fact.

### One at a time, account-wide

Anna's Archive allows **one concurrent download across all partner servers**, so
`anna_fetch.py` is a queue with exactly one worker — not a pool. Firing a second
transfer does not go faster, it gets refused.

The worker is **self-electing**: whoever enqueues first takes a lockfile and
drains the whole queue; later callers see the lock, add their item and exit, and
the running worker picks it up. Nothing stays running once the queue is empty.
That is deliberate — it means there is no daemon to babysit, and **no way for
stale code to keep serving after an edit**, which is the trap the mailwatch
daemon documents at length. One-at-a-time is structural (one worker, awaiting
each transfer), not a setting anyone can get wrong.

The lock steals only from an owner that is really gone: pid alive **and** a
python image name, the same two-part check the mailwatch pid-reuse bug forced.

### Links die after two hours, and the queue knows

A partner url carries its own expiry in the path — `/s/<unix-ts>/` — always
exactly 2h after the page was served. Measured on two separate pages, and an
expired url answers **403** where a live one answers **200**.

This matters *because* of the queue: a book waiting behind several others can be
holding a dead link by the time its turn comes. So the queue stores the expiry
and refuses to start a transfer whose url is already dead, saying

> Anna: link expired while queued — re-run jump N for …

rather than letting aria2 report a bare 403, which reads like "the server
refused us" instead of "mint a fresh link". Queue depth is therefore
self-limiting rather than silently lossy.

### One connection, deliberately

aria2 can split a file across 16 connections. **We use one.** This exists to
survive flakiness and to resume, not to go faster and not to get around AA's
throttling — and 16 parallel requests is exactly what earns the 429 above. The
countdown is still served in full, in the browser, upstream of this script.

### Filenames must be the ones Chrome would have written

`chrome_style_name()` reproduces Chrome's sanitising rather than inventing a
scheme: Windows-illegal characters become underscores (AA's real titles contain
colons — "A psalm for the wild-built **:** Monk & robot series"), and an
over-long name truncates **from the end**, keeping the extension.

That is not cosmetic. `kindle_import.looks_annas()` recognises a book by AA's
naming convention, and `anna_metadata.py` joins the captured sidecar to the
imported file on the **md5 inside the filename** — which sits before the trailing
"Anna's Archive", so end-truncation preserves it. There is a book in the library
ending `-- Anna's Arch.epub` that shows Chrome doing exactly this. Get the naming
wrong and books import with no metadata and no tags, which looks like a bad AA
record rather than a naming bug. Pinned in
`Scripts/codebase_tools/tests/test_anna_fetch.py`.

### Where each path leaves the file

`AnnaDownloadBook(server, importAfter)` arms the correct tail **after** it knows
which shape the server offered, and the two must not be crossed:

| Path | Transfer | Completion signal | Tail |
|---|---|---|---|
| printed url (**normal**) | aria2, queued | aria2 exit code | `anna_fetch.py` calls `import_and_send` itself |
| "Download now" link (**fallback**) | Chrome + Save As | size stable for 3 polls | `anna_download_watch.py` armed **before** the click |

**Never arm the polling watcher on the aria2 path.** aria2 writes straight to the
final filename, so a watcher looking for "a book file whose size stopped
changing" can mistake a stalled transfer for a finished one and import a
half-written book. `AnnaGrabBook` no longer pre-arms for that reason.

`importAfter` is **false** for the batch retry and for a bare manual call, so
neither changed behaviour: the batch still downloads the whole list and files it
in one pass afterwards (`anna_batch_import.py`). `_AA_WaitForDownloadsIdle` now
also counts `*.aria2` control files — without that the batch reads "idle" the
moment it switches to the aria2 path and fires the next book straight into AA's
one-at-a-time limit.

### Operating it

```
MAINFUN.bat AnnaShowQueue          what is downloading and what is behind it
py Scripts\anna_fetch.py status     the same, as text
```

The queue is the one part of this pipeline with no visible surface: a book
sitting behind three others for twenty minutes looks, from outside, exactly like
a book that silently failed. Hence `AnnaShowQueue`.

Install: `winget install aria2.aria2`. `anna_fetch.aria2c()` resolves the binary
from PATH first, then winget's shim and package directories — because winget's
own install says "restart your shell to use the new value", and the always-on
AHK processes were started long before that.

## The tooltip that killed the scrape

**Symptom:** start a run of `jump N`s, and partway through it just stops. No
error, no failure tooltip, nothing in the log. Often right around when an
*earlier* book's "Sent to Kindle" tooltip appeared.

**Cause:** `MAINFUNCTIONS.ahk` line 6 is `#SingleInstance Force`. Every
`MAINFUN.bat` call starts a fresh MAINFUNCTIONS process, so Force means **any
new command terminates the command already running**. That is invisible almost
always, because commands finish in milliseconds — but the Anna click-through
sits for 20–120 seconds waiting out a countdown, and this pipeline's own
detached watcher announces every finished download by shelling
`MAINFUN.bat ShowTooltip`. So the pipeline was killing itself: book 3's
completion tooltip terminated book 5's click-through. There is no log line
because the process was gone before it could write one.

Anything that fires a tooltip on a timer would do the same, so this was never
really an Anna bug — Anna is just the only thing slow enough to notice.

**Fix:** the slow half moved into its own script, `Scripts/AnnaGrabHost.ahk`.
`#SingleInstance` groups by SCRIPT, so MAINFUNCTIONS instances can replace each
other all day without touching it. `AnnaGrabBook()` is now a thin launcher that
starts the host and returns; `AnnaGrabPipeline()` is the work. Same shape as
`KindleGrabHost.ahk`.

**The one-at-a-time gate lives in the LAUNCHER, not the host.** Two
click-throughs cannot overlap safely — both drive Chrome's *active* window
through UIA, so a second grab switching tabs makes the first read the wrong
page. The launcher therefore REFUSES a second grab (with a tooltip saying so)
rather than `Force` silently killing the one in progress: a refused grab costs
another `jump N`, a killed one wastes a countdown already served. Lockfile is
`%TEMP%\anna_grab_host.lock`, released on every exit path including the error
handler, with a 4-minute staleness guard so a crashed host cannot lock her out.

**Still on the old footing:** `AnnaRetryDownloadsFromList` runs its loop inside
MAINFUNCTIONS and is vulnerable to exactly the same kill. It should move behind
the host too.

## Server health — benching one that is broken

Servers fail, and they do not fail evenly. On 2026-08-26 server **#4** was used
twice and failed **both times, instantly**, with `bad HTTP response header`,
while #1/#2/#3 succeeded every time that morning. Nothing recorded *which*
server failed, so the pattern was invisible and #4 kept its turn in the
rotation.

`INIDATA/anna_server_health.json` now records every attempt per server, and
**two consecutive failures bench that server for an hour**.

- **Two, not one.** A single 502 is normal weather on these hosts and aria2
  retries through it; benching on one would pull healthy servers out.
- **Consecutive.** A success clears the streak — otherwise two unlucky failures
  a week apart would bench a working server.
- **A success un-benches immediately**, so a server fixed at the other end comes
  back on its own. Nobody has to remember to un-bench anything.
- **A bench is a preference, not a prohibition.** If *every* server is benched
  the rotation ignores the bench rather than refusing to download — a pipeline
  that silently stops for an hour is worse than a slow one.

Which server a url came from is known only to the AHK side that clicked it, so
it rides along on the queue item (`--server N`). Failures the AHK side sees
itself — a countdown never served, a missing slow-server link — report through
`anna_fetch.py report N fail`, because a server that *hangs* is the worst kind
and must count too. **Python is the only writer** (atomic replace); AHK reads
the file natively on the hot path.

```
MAINFUN.bat AnnaShowServerHealth     per-server history + any benches
py Scripts\anna_fetch.py health
```

## Provenance — so the library can be re-derived

`INIDATA/anna_provenance.jsonl`, append-only, one line per grab:

```json
{"at":"...","md5":"...","detail_url":"https://annas-archive.gl/md5/<hash>",
 "book_title":"...","search_title":"Federici, Silvia - Search - Anna's Archive",
 "search_query":"Federici, Silvia"}
```

`library.json` can be rebuilt from the files on disk, but **which search turned
a book up, and which AA record it is, exists nowhere on the file itself**. Lose
the library and that context is gone. This lives in `INIDATA` — version
controlled and mirrored — rather than beside the media on `E:`, precisely so it
survives losing the drive.

It records the **book page, never the download page**: download urls carry a
two-hour expiry token and are dead almost immediately, while `/md5/<hash>` is
permanent.

The search title is read off Chrome's **other tab**. `jump N` opens the result
in a new tab, so the search page is still sitting there, and Chrome exposes
every tab's title in the UIA tree — no switching, nothing disturbed. Names go
through the shared `ChromeCleanTabName` to strip Chrome's
" - Memory usage - 171 MB" / " - Pinned" decorations.

Written from `_AA_CaptureMetadata`, which is the only moment where the book
page, the md5 and the originating search tab all exist at once. Never fatal: a
provenance failure must not cost the book she is actually trying to get.

## After the url is taken

The download page has done its job the moment its url is queued, so it closes
(`Ctrl+W`) and drops her back on the search results, ready for the next
`jump N`. Gated on the foreground context rather than fired blind — a stray
Ctrl+W closes whatever tab is in front.

If the download later fails, `anna_fetch.py` **reopens the book's md5 page** so
the retry is one `jump` away rather than a fresh search. The download url is
dead by definition at that point; the book page never expires.

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
- Rotation spreads the load, but there is still no FALLBACK: if the chosen
  server's countdown never completes, that book fails and the next one simply
  gets the next server. A retry against the following server would cost another
  full countdown, which is why it is not automatic.
