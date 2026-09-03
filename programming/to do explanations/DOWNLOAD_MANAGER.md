---
tags: [programming, downloads, media, ahk, python, miller, design-doc, books, tv, movies, music]
---

# Download manager — did everything I started actually finish?

**Authoritative doc.** Read before touching `Scripts/downloads/downloads.py`,
`Helpers/DownloadsMenu.ahk`, or adding a new producer.

Related: [[ANNAS_ARCHIVE_PIPELINE]] · [[MEDIA_SYSTEM]] · [[COMPLETION_LOG]]

---

## The problem

Downloads happen in several unrelated subsystems — Anna's Archive books via
aria2, TV/film/music via qBittorrent, more later — and **none of them could
answer "did everything I started actually finish?"**

Each knew only its own live state, and only while it was running:

- **The Anna queue actively DELETED finished rows** when it drained (*"finished
  rows are history, not state"* — my own comment, and wrong). A book could fail
  and leave no trace at all.
- **`acquire_media_log.jsonl` does not fill the gap.** It is a *filing* log: it
  records moves that already succeeded. A download that never completed never
  appears in it, by construction.
- **AcquireMenu's "Downloading" list is live qBittorrent state**, with no
  history at all.

So there was nowhere that *"this started and never finished"* was written down —
which is exactly the thing that goes unnoticed, because nothing is on screen to
notice.

## The shape

An **append-only event log**: `INIDATA/downloads.jsonl`, one line per state
change, never rewritten. Current state is **derived** by folding the events.

```json
{"at":"2026-09-01T09:50:08","source":"anna","kind":"book",
 "id":"c7e49af174931d9f205f0e10c394b1db","title":"A Psalm for the Wild-Built",
 "state":"active","path":"E:\\Downloads\\A psalm … .epub"}
```

**Append-only is the entire point.** A store that is updated in place can lose
the row you are looking for — which is literally what happened — while a log
that is only ever appended to cannot. It also means a crash mid-write costs at
most the last line, and that a corrupt line is skippable rather than fatal (one
bad line must not hide the whole history and reintroduce the blind spot).

**`id` is stable per item** — the md5 for a book, the infohash for a torrent —
so repeated events fold onto one row instead of piling up. For books that is the
*same* key the metadata merge and the provenance log already use, so the manager
joins to both without inventing a new identity.

**Rows are keyed per `(source, id)`**, so two subsystems can never overwrite each
other.

**The first event's timestamp survives as `started`.** An item that never
finished still shows when it began, which is the question actually being asked
of a lost download.

## States are STAGES, not pass/fail

A download is a chain, and **the most useful thing the log can tell you is where
it stopped**. "It failed" is not actionable; "it downloaded fine and then the
Kindle send failed" is — the file is on disk, the library has it, and only the
last step needs redoing.

```
queued -> active -> downloaded -> imported -> sent          (books)
queued -> active ----------------------------> filed        (torrents)
```

| State | Means | Bucket |
|---|---|---|
| `queued` `active` | in flight | still owes work |
| **`downloaded`** | **file arrived, nothing imported it** | **still owes work** |
| `imported` | in the library, not e-mailed | done |
| `sent` | delivered to the Kindle | done |
| `filed` | torrent moved into the TV/film/music library | done |
| `oversized` | in the library, **cannot** be e-mailed | done |
| `skipped` | looked at it, left it alone | neither |
| `failed` | stopped, with the `stage` it stopped at | failed |

**`downloaded` is deliberately a LIVE state.** A row sitting there is the
sneakiest outcome of the lot: the file is in `E:\Downloads` at full size and
looks completely fine, and no library has it. Nothing on screen used to say so.
It gets its own menu section for that reason.

**`oversized` is a success, not a failure.** It imported fine and simply cannot
be e-mailed (Gmail's 25 MB cap) — it needs the USB or web uploader, not a retry.
Filing it as failed would send Jamie chasing a bug that does not exist.

**`skipped` is neither.** "Looked at it and left it alone" — a torrent that
turned out not to be music — is a real answer to "what happened to that?", and
must not be dressed up as either outcome.

**`stage` is separate from `state` and from `note`.** The state says the chain
stopped; the stage says how much of it succeeded (`download` / `import` /
`metadata` / `send`); the note is the raw error. `describe()` leads with the
stage, because "where did it get to" is the question actually being asked.

**A stale note must not haunt a finished row.** Only truthy fields overwrite
during the fold, so an earlier stage's note survives — `unpacking` hanging off a
row that finished ten minutes ago reads exactly like a stuck job. Notes are
suppressed on `DONE_STATES`.

## History vs progress

**Progress is deliberately NOT in the log.** A percentage is a property of a
transfer happening right now, not an event; writing progress lines would swamp
the log to record something stale a second later.

So: **history comes from the log, progress is read from the source** — the
part-file on disk for aria2 (it writes straight to the target name, so its size
*is* the progress), the qBittorrent WebUI for torrents — and only for items
still marked live.

## The pieces

| Layer | File | Owns |
|---|---|---|
| Engine | `Scripts/downloads/downloads.py` | `record()`, `fold()`, live progress, the CLI |
| Store | `INIDATA/downloads.jsonl` | append-only events (version-controlled) |
| UI | `Helpers/DownloadsMenu.ahk` + `Scripts/DownloadsViewer.ahk` | `open downloads` |
| Producer — books | `Scripts/anna_fetch.py` (`track()`) + `anna_download_watch.py` (`_track()`) | queued / active / downloaded / imported / sent / oversized / failed+stage |
| Producer — torrents | `Scripts/acquire/pipeline.py` (`_track_job`, hung off `set_job`) | queued / active / filed / failed / skipped |

## The menu

`open downloads` / `MAINFUN.bat OpenDownloads`:

```
Still unfinished (3)              started but never reported finishing
Downloaded, never imported (1)    the file is on disk, the library does not have it
Failed (1)                        shows which STEP it died at
Too big to e-mail (1)             in the library - needs USB or the web uploader
Books · TV · Films · Albums
Everything (8)                    newest first
↻ Refresh
```

**The first four rows are the alarm; everything below is browsing.** "Still
unfinished" is first because an item there started and never reported an ending.
"Downloaded, never imported" gets its own row despite being a subset of it,
because it is the one that LOOKS fine — full-size file on disk, no library
entry — and would otherwise hide among the in-flight rows.

`MAINFUN.bat ShowDownloadSummary` answers the same question as a tooltip without
opening anything.

## Retry (row action `2`)

What "retry" means depends on where the chain stopped, so the action branches:

| State | Retry does |
|---|---|
| `downloaded` | re-runs **just the import** (`anna_download_watch.py --complete <path>`) — the file is already on disk; re-downloading it would be silly |
| `failed` (anna) | **reopens the book page**, from which one `jump` runs the whole thing again |
| anything else | says there is nothing sensible to retry, rather than pretending |

A failed download's url is dead by then (partner links expire after 2h), which
is why the failed path reopens the page rather than re-queueing the url. The
book page is rebuilt from the row with nothing extra stored — **the row's `id`
IS the md5**, and that is what the page is keyed by.

**The menu reads `list --json`, not the formatted line.** It used to parse the
human-facing output, which has no id in it — and a retry needs the id to know
WHICH book. Parsing a display string for machine fields was going to rot anyway.
Every field read goes through `_DlGet`, because the engine omits empty fields to
keep the log small and a missing key is a THROW in AHK v2: a menu that dies on
the first row with no `stage` is worse than one that shows "".

## Adding a producer

Two lines. Import the module and call `record()` on each state change:

```python
sys.path.insert(0, str(AHK_BASE / "Scripts" / "downloads"))
import downloads as _dl
_dl.record(source="acquire", kind="tv", item_id=infohash,
           title=name, state="done", dest=str(target))
```

Rules for a producer:

- **Guard the import.** Every pipeline here predates the manager and must still
  run if it is ever missing. `anna_fetch.track()` is the reference.
- **Never let tracking break a download.** `record()` returns `False` rather than
  raising, and callers ignore the result. Tracking is the least important thing
  in any of these pipelines.
- **Use a stable id**, not the title. Titles get cleaned, truncated and
  re-derived; an md5 or infohash does not.
- **Kinds are a closed set** (`book · tv · movie · album · other`). A typo'd kind
  would silently create a category no menu section shows, so unknown kinds are
  coerced to `other` rather than accepted.
- **Hook the chokepoint, not the call sites.** The torrent producer hangs off
  `set_job()` — the one place a job's state changes — so a state added later is
  tracked for free instead of being silently missed. Look for the equivalent
  before instrumenting a dozen places.

## Gotchas

**The Anna queue keeps its own history now too.** `QUEUE_HISTORY_LIMIT` caps it,
and `status --history` shows it. That is deliberately redundant with this log:
the queue's copy is operational (it is what the worker reads), this log is the
durable record.

**Keeping queue history exposed a latent bug.** The de-dup test was
`state != "done"`, which — once failed rows stopped being deleted — would have
treated a FAILED row as still queued and silently refused to retry the very book
most likely to need it. It now tests `state in LIVE_STATES`.

**The menu shells the engine once per build and caches it** for the life of the
viewer process. A Miller previews the highlighted row eagerly, so a subprocess
per preview would stutter the UI — the same reason `SettingsStore` reads its
JSON natively. `↻ Refresh` re-reads.

## Voice

`open downloads` opens this menu. It used to open the downloads FOLDER, via the
generic `open <directory>` Choice — that entry was renamed to
**`downloads folder`**, so:

| Say | Get |
|---|---|
| `open downloads` | this manager |
| `open downloads folder` | `E:\Downloads` in Explorer |

The rename lives in `INIDATA/VoiceChoices/directories.json`; the manager's phrase
is a generic-store row (`voice_wizard.py assign`). **`voice_index check` will
report a stale COLLISION until the grammar dump is refreshed** — it reads the
last `make catalog` dump, which still lists the old `downloads` key.
