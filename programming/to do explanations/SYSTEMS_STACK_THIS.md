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
| **fix &lt;system&gt;** | A new Claude Code session, prompt typed in: work through that system's open items. |
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

**Routing**, best first: a url/title pattern, then the deepest context token a system claims, then in a Claude window the systems that chat has been editing (and its folder), then each candidate's parent systems.

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

## The fold (2026-11-06)

The old stacks became systems: poem → **Poetry**, tab → **New tab page**, graphical → **Miller menus**, log → **Completion log**, Journal → **Journal**, numpad → **Numpad**, dragon → **Voice**. The Atlantic stack's items moved into the **Atlantic** workspace, which is a system now. "Spotify / Music catalog" became **Music Library**, and "Media system" became **Media catalog**. New: **Reader**, **Torrents**, **YouTube**. The **Macro** workspace is the umbrella.

**Second batch (same day, from a week of context time + three weeks of the programming log):** Messages, Claude app, Dictation & mic, Windows & desktops, Pedals & bindings (Numpad folded in), Local pages kit, Events, Meditation, Chat room (inside Journal), plus Chrome extension and Reminders from the old lists. Programming became **Computer**. Every open item was sized by Claude; ones that look already built carry a "Looks done:" note and show on the **Looks done already** shelf for her to confirm.

Left as plain stacks for review: macros (still the "stack that" target), habit, lockout, reading, research, tech. Backup before the fold: `INIDATA/Projects/projects.before_systems_2026-11-06.json`.

## Files

- Engine (the one writer): `AutoHotkey/Scripts/systems/systems.py`
- Miller + page spec: `AutoHotkey/Scripts/systems/systems_library.py` (feed kit, see [[POEM_PRINT_AND_DESIGN]] § The library)
- AHK: `Helpers/SystemFunctions.ahk`, `Helpers/SystemsMenu.ahk`, `Scripts/SystemsViewer.ahk`
- Voice: `caster/rules/system_commands.py` (`<system_slot>` from `INIDATA/VoiceChoices/system_slots.json`)
- Hook: `~/.claude/hooks/systems_hook.py`; per-chat record `~/.claude/context/systems_sessions.json`

Related: [[STACKS_SYSTEM]], [[ALWAYS_ON_CONTEXT_DETECTOR]], [[LOCAL_APP_HUB]]
