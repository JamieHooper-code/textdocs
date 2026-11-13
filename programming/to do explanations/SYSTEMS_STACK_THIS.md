# Systems: "stack this"

Tags: #systems #stacks #todo #claude #voice #design

**Built 2026-11-06.** A to-do list per part of the setup, filled from wherever she is, that Claude brings up when it works on that part.

## The problem

Using the music library, the reader or the torrent pipeline, she notices small things that are wrong. Each one either went on the main stack, onto its own stack, got done immediately, or got forgotten. Stacks were barely used and the to-do lists not at all, because nothing put the right list in front of the right moment.

## What she says

| Phrase | What happens |
|---|---|
| **stack this** | Typing box with dictation on, aimed at the system for WHERE SHE IS. The headline says "Onto: Music Library" and where it was said from. Ctrl+Enter files it. |
| **stack this dump** | The same, plus a full make dump (screenshot + UIA + manifest) filed with the item. |
| **stack &lt;system&gt;** | The box aimed at that system, from anywhere ("stack reader" while on YouTube). |
| **fix &lt;system&gt;** | A new Claude Code session, prompt typed in (New chat + paste, no trust dialog): work through that system's open items. "fix sort" sorts and triages the inbox. |
| **fix this** | The same, for the system where she is, picked like "stack this". In a Claude chat it's the system in the chat's title ("Reader: …"). |
| **fix recent** | A new session on the item she stacked last: claim it and start. Still in Sort → sort and triage first, then that item. |
| **fix next** | Done with a chat, want to keep going: a new session running "send program", told the title of the chat she fired it from so it can point out what follows on. Busy systems still sorted last by claims. |
| **fix program** | A new session running plain "send program", with nothing about the chat you were in. |
| **send project** / **fix project** | Her mood for a bigger build: a chat lists the Projects lane, she picks one, and it decides the home with her (new system, subsystem, or a large item in an existing one) before building. |
| **stack backlog** / **fix backlog** | Things she doesn't want shown: out of every plan; only when she asks. |
| **open systems** / **open systems page** / **open systems miller** | Every system and its list: the Miller, or the same thing as a Chrome page (:8302/systems/). |

In the box: **Alt+1–9** picks another candidate (this place's system first, then the bigger systems it sits inside: Music Folders → Music Library → Macro). **Alt+0** picks any system.

**First time somewhere no system claims**, the Question window asks once: a new system (name, what she'll say, inside which system, what "here" means: this context, this website/local page, these windows) or an existing one. After that, "stack this" there just opens.

## The data

No new store. A **system** is a workspace or project in the projects store (`INIDATA/Projects/projects.json`) with a `system` block:

- `spoken`: what "stack ___" / "fix ___" hear
- `contexts`: context tokens that route here (`INIDATA/Contexts/*.json`)
- `match.url` / `match.title`: regexes for places with no context of their own
- `code`: file globs. Editing one means Claude is working on this system.
- `folder`: where "fix" opens Claude (defaults to the AutoHotkey folder)
- `about`, `docs`

**Items** are ordinary to-dos under it, plus `captured` (context chain, title, url, what was open via the make dump source functions, the selection, the dump folder) and `size`. Sizes are **never asked of her**: Claude rates them (tiny / small / medium / big) when it offers them.

**Routing**, best first: a url/title pattern, then (our own AHK windows only) a title whose lead IS a system's name or spoken word ("Writing box: New poem — Thu" → Writing Box, no rule needed), then the deepest context token a system claims, then in a Claude window the systems that chat has been editing (and its folder), then the system whose code drew the window, then each candidate's parent systems.

**Rules stay general (2026-11-13).** The first-time question offers the title's stable lead first ("Every Writing box window"), the whole title only as "Only windows titled: …". Adding a title rule drops the narrower ones it covers and skips one that's already covered (`add_pattern`). Chrome titles never name-match: they're page-authored ("Voice - Messages" is not Voice commands).

**It learns from her picks.** The box remembers its first guess (`captured.guessed`, "" when it had to ask). When she files onto a different system from the same place a second time, that place routes there from then on, and the tooltip says so ("From now on: Every Gratitude box window → Poetry"). Never for Sort or a lane (that's "decide later"), never for "stack <name>" said from elsewhere. `learn_route` in systems.py.

**The box says why.** The headline reads "Onto: Writing Box · Every Writing box window". Every reason is in plain words: a route's label, "window named Poem", "its code drew this window (X.ahk)", "you named it", "holds Writing Box" for a parent.

**Seeing and removing routes.** The Systems page and Miller have "Where 'stack this' goes" (every route, grouped by system) and each system's Set up → Routes. Enter on a route asks "Remove this route? (Esc keeps it)"; the writer is `systems.py remove-route <sys> <kind> <value>`. `systems.py routes` lists them as JSON.

**Every miss is logged for review.** No automatic weekly check: the data piles up for a periodic review with Claude. A miss is a stack filed onto something other than the first guess (the `add` history line carries `missed`, `guessed`, `place`, title, url), or an item later moved off the system it was stacked onto. `systems.py route-log [--since DATE]` prints them grouped by place, plus every learned route; the "Routing misses" shelf shows the same.

**Code ownership: an exact file beats a folder.** A system that names a file outright (Writing Box: `Helpers/Gui/WritingBox.ahk`) wins over one that owns its folder by wildcard (Miller menus: `Helpers/Gui/*`).

## The log snapshot

Each item can carry what the system was doing when she stacked it, saved in its folder (`INIDATA/Systems/dumps/<item>/`, gitignored) and **never put in front of Claude**. The brief shows one line ("logs: <folder>"), and Claude opens it only when an item looks like a bug.

- `ahk_event.txt`: the AHK event log from 10 minutes before "stack this" (the capture file's time) until it was filed. About 250 lines.
- `voice_commands.txt`: the last 50 voice commands Caster recognised (rdescript lines only, no dictated text). The Dragon log has no timestamps, so it's a count, not a window.

When it's taken is a per-system setting (Set up → Log snapshot, `systems.py set-logs <sys> here|always|never`):
- **here** (default): only when said from inside the system, so "stack reader" from YouTube skips it.
- **always**: systems you're always "in", like the mic. Voice, Numpad and Macro are set to always.
- **never**

Why keep it at all: both logs are 4 MB+ and scroll, so the moment something went wrong is buried within hours.

## Claude's side

`~/.claude/hooks/systems_hook.py`:

- **A prompt names a system** with open items (first time in a chat). "Names" means it's the subject: a multi-word name anywhere ("music library"), but a single word only after the/my/fix/stack/open/work on, or before system/page/setup. "The most recent Dragon commands" is a passing mention and doesn't count (`systems.py mentions`, shared with `match-text`). Then: its list goes into Claude's context. At the START of the answer, Claude gives design considerations and offers related items to do in tandem.
- **An edit lands in a system's code** (first time in a chat): recorded for that chat (this is what "stack this" in a Claude window resolves to), and Claude is told to offer the items at WRAP-UP, never mid-task. Umbrella systems (ones holding other systems, like Macro) are recorded but never announced.

Claude sizes with `systems.py size`, offers items numbered for numpad replies, and ticks them off with `systems.py done` (which logs to the completion log).

## "send programming"

Also "send program" (an alias). In a chat that is already about a system (talked about it, or touched its code), Claude focuses on that system instead, via `plan --focus <system>`, and the plan says what its own pick would have been. A send (`ctl sends`, group plan) that hands Claude a standing prompt: run `systems.py plan`, then offer **triage first** (batches she answers with one number: done / drop / move / keep), then **two to four options** for the session, smallest-and-most-useful first. Change nothing until she answers.

`systems.py plan` is read live. The **focus** is the system she has STACKED onto most in the last 3 days, then the last week, else the biggest pile. Only items carrying a "stack this" capture count as recent, so bulk-filed items never look hot. The output has the focus system's full list (each item cut to about 300 characters, with a pointer to the full text), the other systems ranked, every "Looks done" item, and old stacks still holding items.

## Triage is stored, not redone (2026-11-08)

Claude triages each item **once** and the engine keeps it, in `item.triage`, written with `systems.py triage-batch`:
- verdict (keep / drop / done / move + where)
- reason
- **score** 0-100: how worth doing it is now
- **rank** inside its system
- **cluster**: a short key shared by items that fold into one session
- notes: the files it touches and the approach
- a fingerprint of the item's text

An edit to the text makes it stale; a move to another system clears it so it's ranked fresh there. Proposals stay **proposed** until she answers (`done` / `drop` / `move` / `keep`). Unanswered ones are re-asked next time, never re-thought.

**Master priority** (`systems.py priority`, in every plan): batches, not items. Items sharing a cluster in one system go together. A batch scores its best item plus 5 for each tiny/small member that rides along. A member far below the best (under 60% of its score) that isn't cheap stands alone. So 1, 3 and 4 beat 2 when they fold together, but 55 doesn't ride along with 1.

**Recency** (her rule: "the most recent stacked items first; if I stacked something 3 months ago and still haven't done it, we'll get to it when we get to it"). Every score is weighted by when she stacked it:
- the last week: full weight
- sliding down to half at 3 months or more
- bulk-filed items, which have no "stack this" capture: 60%

**Answering it (built 2026-11-08).** Two shelves at the top of the systems page and Miller (`systems_library.py`):
- **Priority**: the master list as batches, a divider per batch, Claude's call + score + reason on each row, and "Keep all N" when a batch has more than one unanswered.
- **Waiting on you**: only unanswered proposals, grouped by what Claude says (already done / drop / move to X / keep), each group with "Accept all N" so one answer settles it.
- Answers go through `systems.py accept <item...>` (take Claude's call as it stands, one or many) or done / drop / move / keep; each writes a history `answer` line with matched / overridden. Page clicks log as `who: page`.
- **Voice** on the page: "accept 3", "keep 2 and 5", "scrap 4", "done 1" (and "start scrap" modes). The word is **scrap**, not drop: "drop [N]" is a global navigation command. An item panel also has "Agree: …" (25) and "Keep it" (26).
- **Miller keys** on those two shelves: N.0 drop, N.1 keep, N.3 done, N.4 accept Claude's call (move = open the item).
- Not done: retiring `StackTriage.ahk`; it still serves the leftover habit / tech stacks.

**Sort** (top level, "stack sort" / "stack inbox"): an inbox nothing routes to. Every "send program" files it first. It's also the first choice when "stack this" lands somewhere no system claims.

**History** (`INIDATA/Systems/history.jsonl`, gitignored; `systems.py history`): one line per write, saying who did it (voice / menu / claude). It also records what "stack this" offered and which one she picked, every plan run, and whether her answer matched Claude's proposal. This is the data for tuning routing, ranking and triage.

## "send finished" (alias "send done for today")

The end of a chat she doesn't want to keep open. Claude:
1. Documents only if something changed how things work. Nothing to add means nothing added.
2. Files usually 0 to 5 genuine follow-ups (features, gaps, bugs, loose ends; never "verify" chores, never duplicates) with `systems.py stack-batch`, fully triaged while the chat still knows the files.
3. Reports them numbered so she can drop any.

It never commits: she does her own bulk commits (2026-11-08).

Those items carry `captured.from_chat`, the session and its transcript. They count as freshly stacked. "Fix this one" points the new chat at the transcript, and **Back to the chat it came from** (`SystemResumeChat`) reopens the original.

## "send fresh" (alias "send stacked") (2026-11-11)

Her loop: work on something, go use the UI, stack 7 or 8 bugs and wants, come back to the chat. "send program" is the wrong tool for that (it plans across every system), and the chat's old picture of the list said "empty". `systems.py fresh` lists what she stacked on **this chat's systems** (its title, claims, edits, what it was told about) since it last looked (`seen_at` in the session record; the first time, since the chat started), sized, plus a one-line list of anything stacked elsewhere in the last few hours. The send then: **tiny/small → claim and just do them**; **medium/big → a short design sketch each, numbered**; no full triage.

The hook also notices on its own: on any prompt, if new items landed on the chat's systems since it last looked, one line tells Claude its picture is stale and to run `fresh`.

## Several chats at once: claims (2026-11-09)

She opens several chats in a row and says "send program" in each, picking in one before opening the next. Each chat used to offer the same top items. Now the moment she picks, the chat runs `systems.py claim <items> --title "<System>: <thing>"` and renames itself to that title. Claims live in `INIDATA/Systems/working.json` (`systems_working.py`), and the hook re-stamps them on every prompt and edit in that chat, so their age means "last active":

- **fresh** (under 12 h): IN PROGRESS in the plan, left out of the options.
- **stalled** (12 h to 3 days): offered, tagged, ranked a bit lower.
- **old** (over 3 days): offered normally, "worth a revisit". A claim never means finished.

The claimed items and their cluster are left out. A fresh claim also makes the rest of that system **BUSY**: still offered, but **last**, with "another chat is on <system>". Big, wide systems where several chats at once is normal are marked **several chats at once** (`set-parallel`, or Set up → Chats at once) and are never busy: Computer, Macro, House and Personal so far.

A plan run that finds untriaged items stamps "triage in progress" for 20 min (refreshed by `triage-batch`, cleared by the chat's claim), so a second chat skips triage instead of redoing it. "send finished" runs `release --note`, which clears the claims and leaves "touched <date>, not finished" on anything still open.

**A write-up is not the goal.** An item asking for options or a write-up stays open once the write-up exists; the next step is building it.

**Follow through on features (2026-11-11).** Chats in stack mode were treating a medium/big item like a quick fix: half-doing it, then offering something unrelated. A picked feature is now the session's job: built, working, checked, rough edges handled, and the chat asks her before moving on. Tiny bugs stay fix-and-next. Wording lives in the program and fresh sends, the hook's wrap-up text (it only offers the list once the work is actually done), and the CLAUDE.md Systems section.

## Lanes, sizes and what chats see (2026-11-12)

**Who decides what's shown.** No background AI does triage. The Claude chat she's in does it when she says "send program": it reads `systems.py plan` and writes a score from 0 to 100 per item with `triage-batch`. That's the chat's normal usage, nothing extra. Item titles come from a local model on her GPU (`projects.py _gen_title`, Ollama), which is free. The ranking itself is plain arithmetic: score, lowered with age (half at 3 months), grouped by shared code, with claimed items held back. "send program" offers the top 2 to 4, so anything below about #5 was never offered. The Cooking build sat at #17 that way, because chats score big builds low on "worth doing *now*".

**Sizes:** tiny / small / medium / large / project. "big" is still accepted and read as large; the 24 stored "big" items were renamed. No time estimates, ever.

**Lanes:** two systems with `lane` set (`set-lane`) whose items are kept out of the plan, the priority list, triage and "send fresh":
- **Projects** (`p-91b2894e`): big builds. A project-sized item moves there by itself, remembering its `home`. Every plan names them in one line, and "send project" / "fix project" offers them. When she picks one, the chat decides its home with her (a new system, a subsystem, or a large item in an existing one).
- **Backlog** (`p-1747c0ce`): only shown when she asks. `backlog <ids>` moves items there and `restore <id>` puts one back where it came from. The plan lists **BACKLOG CANDIDATES** (60+ days old and score 25 or below) for her to approve as one batch; chats never move them on their own.

**WAITING A WHILE:** each plan names the best-scored item that's 3+ weeks old and below the top of the list, so a good old idea comes back up now and then.

**Quieter hook (scatterbrain fix):** the full list goes in only at the start of a chat (the first 2 prompts, nothing claimed yet). Edits get a one-line background note instead of the list. The "new items stacked" notice says it's not a request. Every prompt asks chats to round out a finished feature (show it, offer what's next for *it*, ask if it feels finished) before going back to the list.

## The fold (2026-11-06)

The old stacks became systems: poem → **Poetry**, tab → **New tab page**, graphical → **Miller menus**, log → **Completion log**, Journal → **Journal**, numpad → **Numpad**, dragon → **Voice**. The Atlantic stack's items moved into the **Atlantic** workspace, which is a system now. "Spotify / Music catalog" became **Music Library**, and "Media system" became **Media catalog**. New: **Reader**, **Torrents**, **YouTube**. The **Macro** workspace is the umbrella.

**Second batch (same day, from a week of context time + three weeks of the programming log):** Messages, Claude app, Dictation & mic, Windows & desktops, Pedals & bindings (Numpad folded in), Local pages kit, Events, Meditation, Chat room (inside Journal), plus Chrome extension and Reminders from the old lists. Programming became **Computer**. Every open item was sized by Claude; ones that look already built carry a "Looks done:" note and show on the **Looks done already** shelf for her to confirm.

Left as plain stacks for review: macros (still the "stack that" target), habit, lockout, reading, research, tech. Backup before the fold: `INIDATA/Projects/projects.before_systems_2026-11-06.json`.

## Files

- Engine (the one writer): `AutoHotkey/Scripts/systems/systems.py`; claims `systems_working.py`; triage + history `systems_triage.py`
- Miller + page spec: `AutoHotkey/Scripts/systems/systems_library.py` (feed kit, see [[POEM_PRINT_AND_DESIGN]] § The library)
- AHK: `Helpers/SystemFunctions.ahk`, `Helpers/SystemsMenu.ahk`, `Scripts/SystemsViewer.ahk`
- Voice: `caster/rules/system_commands.py` (`<system_slot>` from `INIDATA/VoiceChoices/system_slots.json`)
- Hook: `~/.claude/hooks/systems_hook.py`; per-chat record `~/.claude/context/systems_sessions.json`

Related: [[STACKS_SYSTEM]], [[ALWAYS_ON_CONTEXT_DETECTOR]], [[LOCAL_APP_HUB]]
