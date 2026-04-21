# Handoff: Obsidian Vault — Next Steps

**Written 2026-04-20.** Session before this one stood up the vault, retargeted SaveCurrentLink at it, built the persistent tag-picker Gui, migrated SavedLinks, and added voice commands. Read `reference_obsidian_vault.md` for the full architecture before changing anything.

## Quick context

- **Vault** at `C:\Users\jamie\Desktop\Important\ObsidianVault\` — 8 migrated category notes + Welcome.md.
- **Capture flow:** "save link \<category\>" pops the tag-picker Gui (filter Edit + ListView + Added panel + PgDn save / Esc cancel). Edit mode kicks in if the URL is already saved.
- **Generic Gui** at `Helpers\PersistentLoopGui.ahk` — reuse, don't reimplement.
- **Tag scanner** at `Scripts\vault_tags_scan.py` — strips frontmatter + nested code blocks.
- **Old `SavedLinks\*.txt` are frozen archives.** Don't write to them. They're a safety net only.

## Loose ends to wrap before new work

1. **Enable Dataview**: Obsidian → Settings → Community plugins → toggle ON. Plugin files already on disk at `<vault>\.obsidian\plugins\dataview\`. One click and Welcome.md's example queries start rendering.
2. **Local `named_mainfun_action_with_args` shims** in 6 rule files (web_opener, windows, spotify_global, lockout, google_voice, link_saver) can be deleted after the next Dragon restart. Upstream `ahk_mainfun.py` is already fixed; the shims defend against the cached sys.modules version. See `reference_voice_add_pattern.md` for the file list.
3. **Unparsed entries** in `PoliticalLinks.md` (and possibly others) — migration left a couple of orphaned chunks at the bottom of that file in a code block. Worth a quick pass: either fold them into bullets or delete.

## Headline task: retroactive auto-tagging

The capture flow now requires Jamie to add tags for new saves, but the 16+ migrated bullets are tagged only by category (frontmatter `#resource #political #books`). To make the vault truly findable for cross-cutting queries like "books about black queer thought," every bullet needs topic tags too.

**Three approaches, ranked by what to try first:**

### Approach 1: Interactive Claude-assisted (RECOMMENDED START)
Build a script that walks every bullet in the vault, proposes 1–4 tags based on title + (optionally) page content, and asks Jamie to approve / edit / skip. Each round writes back to disk so progress isn't lost.

- **Cost:** mostly time. Jamie reviews each bullet once.
- **Quality:** high — Jamie's eye on every tag.
- **Build effort:** small. ~150 lines of Python: walk markdown, parse bullets, prompt user (CLI or simple Gui), append tags inline.
- **Suggested tool:** `Scripts\auto_tag_bullets.py`. Read all `*.md`, parse bullets via the same regex `LinksaverToVault.ahk` uses, present each in a TUI/Gui with a tag-picker similar to `_PersistentLoopPickGui` (could even shell into Obsidian's existing tag autocomplete via the URL scheme). PgDn confirms, → next bullet.

### Approach 2: Rule-based with keyword maps
Build a YAML/JSON map of keyword patterns → tags (e.g. `"queer|trans|gay" → #queer-theory`, `"marx|marxis|capital" → #marxism`). Run it across all bullets, append suggested tags WITHOUT Jamie's review.

- **Cost:** none after build.
- **Quality:** mediocre. False positives (a book ABOUT marxism vs. a book by Marx) and false negatives (subtle topics not in the map).
- **Build effort:** small but iterative — the map needs many rounds of refinement.
- **Use case:** good as a *first pass* before Approach 1 — Jamie just confirms/adjusts the suggestions, much faster than starting from scratch.

### Approach 3: Anthropic API (`claude-sonnet-4-6` or `claude-haiku-4-5`)
For each bullet, call Claude with `{title, url, existing_tags}` and ask for 1–4 topic tags from a constrained vocabulary (the existing tag set + room for new ones).

- **Cost:** small but real. ~16 bullets × maybe 200 input tokens = trivial. Could re-run for new saves automatically as a "tag suggester" mode.
- **Quality:** high if prompted well; risk of hallucinated tags.
- **Build effort:** small — single prompt template, JSON output mode, one batch call (or use Batch API for cost). Use the `claude-api` skill — it'll handle prompt caching + model versioning.
- **Use case:** the long-term winner. Build it ONCE, then it can run on every new save in the background and surface a "suggested tags" list in the picker Gui as a bonus column.

**Suggested order:** Approach 2 to seed → Approach 1 for review → graduate to Approach 3 once the tag vocabulary is mature.

## Other porting candidates

Resources scattered across `TEXTDOCS/` that could become vault notes:

| Source | Worth it? | Notes |
|---|---|---|
| `TEXTDOCS/programming/stacks/*.txt` | Maybe | Stack files are short-lived clipboard buffers. Probably stay as-is; Stack system has its own voice flow. |
| `TEXTDOCS/programming/Quest Log - To Do Programming.txt` | No | Active to-do list, edited heavily, doesn't fit the "resource" model. |
| Google Docs: Thoughts, Journal, Lyrical | No | Long-form writing, edited in GDocs collaboratively. Keep there. |
| Google Docs: Helpers (Anxiety / Phrases / Mantras), Intentions, Highlights, Recipes, To Read, To Listen | **Yes, eventually** | Pure reference content. Could mirror to vault notes for offline + Dataview-queryable + tag-cross-cutting. Use the Google Workspace MCP (see `reference_google_workspace_mcp.md`) to fetch them as markdown. |
| `TEXTDOCS/programming/voice commands current.txt` | No | Living doc, frequent updates, fits as plain text. |
| Notepad++ files (friends, resources) | Maybe | "resources" might be a vault candidate if it's link-shaped. "friends" probably stays. |
| Browser bookmarks | **Yes** | Jamie's existing bookmarks (Chrome) are scattered. A one-shot import via Chrome's bookmark JSON → vault notes (one per folder?) would centralize a chunk of historical signal. |

Suggested first port: **the Helpers Google Docs** (Anxiety, Phrases, Mantras, Intentions). Tight scope, clear value (offline + searchable), and the Google Workspace MCP makes the fetch trivial. Builds the precedent of vault notes that aren't "lists of links."

## Tooling backlog

Concrete scripts worth building (all live under `AutoHotkey\Scripts\` and shell-callable from voice or Claude):

| Script | What it does | Effort |
|---|---|---|
| `vault_tag_rename.py old new` | Rename a tag across all `*.md` (inline + frontmatter). Idempotent, atomic-write per file. Voice command "tag rename". | Small |
| `vault_tag_report.py` | Print tag usage stats, untagged-bullet count, orphan tags (used once). Pipe to a vault note `_TagReport.md` Jamie can open. | Small |
| `vault_link_check.py` | HEAD-check every URL in the vault, flag 4xx/5xx/timeouts. Output a `_DeadLinks.md` note. | Medium (concurrent HTTP) |
| `vault_title_refresh.py <url>` or `--all` | Re-fetch the live page title for one (or all) bullets, update in-place. For "this title is stale" cleanup. | Medium |
| `vault_search_voice.py <query>` | Voice "search vault X" → opens Obsidian's search prefilled via `obsidian://search?query=X`. Wire as voice command. | Tiny |
| `vault_browse.py` | TUI/Gui browse: pick a tag → see all bullets with that tag → pick one → opens in Obsidian. Solves the "I never go back and look" problem. | Medium |
| `clip_to_vault.py` | Pair with the Obsidian Web Clipper extension (or a custom one) to capture full page content (not just URL+title) into a vault note. | Medium |
| `vault_daily_note.py` | Voice "daily note" → opens / creates today's `Daily/YYYY-MM-DD.md` with a template. Builds on Obsidian's Daily Notes core plugin. | Small |

## Obsidian features worth exploring

Sorted by likely Jamie-value:

1. **Dataview dashboards** — once enabled, build named queries: "Reading Queue" (`LIST FROM #toread`), "Recent saves" (`LIST FROM #resource SORT file.mtime DESC LIMIT 20`), "Untagged bullets" (a custom query). These become *living indexes* she actually visits.
2. **Templates plugin** (core) — for new category notes, daily journal, etc. Eliminates repetitive frontmatter typing.
3. **Daily Notes plugin** (core) — date-stamped scratch notes that link to whatever she captured that day. Great for "what was I looking at last Thursday".
4. **Mobile** — vault syncs via any folder-sync (iCloud, Dropbox, Syncthing). Free Obsidian mobile app. Would let Jamie save links from her phone. Trade-off: setup is fiddly.
5. **Web Clipper** — official Obsidian extension. Captures full page content (text, images, sometimes formatted) into the vault. Beats title+URL bullets when the page itself is the resource.
6. **Canvas** — visual board for sticky-note-style relationships between notes. Useful for Jamie's poetry / project planning, less so for resource lists.
7. **Bases** (Obsidian's new database feature) — table-style views of notes with structured fields. Could replace some of what Notion offers without leaving the vault. Worth a tire-kick once she has 100+ tagged bullets.
8. **Graph view** — visual map of all backlinks. More fun than useful unless the vault gets dense.

## Voice command extensions worth thinking about

- "search vault \<dictation\>" — opens Obsidian search prefilled.
- "browse \<tag\>" — open a generated note listing all bullets with that tag (via Dataview or generated via script).
- "daily note" / "today" — open / create today's daily note in Obsidian.
- "tag rename \<old\> \<new\>" — runs `vault_tag_rename.py`.
- "open \<category\>" — shortcut to open a specific vault note (already partially possible via `show link`).

## Pattern reminders for the next Claude

- **Persistent picker Gui:** anything "build a list of N items with autocomplete" → use `_PersistentLoopPickGui(opts)` from `Helpers\PersistentLoopGui.ahk`. Don't write a new one.
- **Tag normalization:** kebab-case via `_ObsidianKebabCase` (and Python equivalent). One canonical form everywhere.
- **Caster reload:** any rule edit needs a `# _reload_marker:` content delta — see `feedback_caster_reload_touch.md`.
- **Voice-add JSON pattern:** if you're migrating a hardcoded Choice dict, follow the 5-edit template in `reference_voice_add_pattern.md`. 12 done, the pattern is stable.
- **AHK Gui layout:** absolute coords, not `ys`/`xs` Section refs. Section anchoring grabbed wrong y on a previous bug.
- **Dragon restarts are forbidden** (see `feedback_never_restart_dragon.md`). Don't fire `RestartDragon`. Mention reboots if needed; Jamie decides.

## What this session decided NOT to do

- **Not Notion.** API flakiness fights voice workflows; Obsidian fits Jamie's local-files style better.
- **Not Logseq.** Same shape as Obsidian but less polished plugin ecosystem; Obsidian won the toss.
- **Not voice-dictation tags.** The `save link \<cat\> \<tags\>` variant was tried and rejected — Jamie wants the persistent Gui flow, one tag at a time.
- **Not auto-update bullet titles in edit mode.** Saved title is preserved (protects manual edits). Browser title only used for fresh saves.
- **Not pre-create empty `<vault>\<Category>.md` at "link add" time.** First save creates the file on demand.
