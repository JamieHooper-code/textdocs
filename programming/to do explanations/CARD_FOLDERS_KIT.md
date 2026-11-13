---
tags: [programming, design-doc, local-viewer, card-kit, music, food, ahk, caster]
created: 2026-11-12
updated: 2026-11-12
status: built 2026-11-12 — Music Folders moved onto it, Food is the second system
related: ["[[MUSIC_FOLDERS]]", "[[FOOD_SYSTEM]]", "[[LOCAL_APP_HUB]]", "[[POEM_PRINT_AND_DESIGN]]"]
---

# The card-folders kit — one engine for every "folders of image cards" page

Jamie, 2026-11-12: *"If there is a world where we should have a shared backend
between this and the spot folder system, we should strongly consider that,
because it feels like we're going to end up building a lot of parallel systems
that could drift … a bunch of folders full of thumbnails that I click through …
Make sure everything is nice and baked in, and only really the data model kind
of plugs into it."*

So Music Folders' page — nested folders of image cards, numbered for voice,
hearts, edit mode with Undo, sort / group, views — was lifted into a shared kit.
Music runs on it unchanged, Food is the second system, and the next one is one
command.

## The split

| Shared (written once) | Per system (its data model) |
|---|---|
| folders that nest, custom tile order | what a card IS (`CardSpec.entry_key`, `fav_key`) |
| filing (the same thing in many folders, "already here" isn't an error) | what's stamped on a card for drawing (`annotate`) |
| custom card order, sort, group, sub-folder display (tiles / rows / mixed) | extra sorts, groups, folder options |
| views (what two folders share), descriptions, auto-favorite folders | smart folders (`smart_view`, `smart_cards`) |
| edit mode: numbered ✕, several numbers = one batch, Undo | item pages: what's on them, their buttons (`item_view`, `item_act`, `item_pick`) |
| hearts as affinity (+1 a press, below zero buries) | what a card click does (`open_entry`) |
| item pages: photo, facts, sections, numbered buttons, numbered pickers | its look (CSS tokens) and its own words |
| every control numbered, the kit's pins, keys, voice action words, live refresh | |
| "Add to a folder…", the voice "grab" mode | |
| process control (serve / state / stop / restart), the pages header | |

This keeps the rule `local_viewer.py` already had: share the **shape**, never
force one content model on everything. Each system's store keeps its own data
(music.json for Music, `data["items"]` for Food); the kit round-trips keys it
doesn't know.

## Files

- `Scripts/viewers/card_store.py` — the engine: `CardSpec` (the hooks) and `CardStore` (folders, entries, order, views, trash/undo, hearts, `view()`), plus the shared CLI (`tree | resolve | create | list | option | undo | describe | flag`).
- `Scripts/viewers/card_server.py` — `CardSite`: the page shell, `/folder/<id>`, `/item/<id>`, `/sys/<file>`, `/api/view`, `/api/item`, `/api/item-act`, `/api/pick`, `/api/resolve`, `/api/folder-choices`, `/api/open`, every generic POST op, and process control.
- `Scripts/viewers/kit/cards.js`, `cards.css`, `cards_page.html` — the page. A system's plugin calls `Cards.use({...})`; the hook list is at the top of cards.js.
- `Helpers/CardPages.ahk` — `CardPageOpen(app, words)`, `CardPageEdit / Undo / EditDone / Up`, and one `Open<Name>` line per system.
- `INIDATA/Contexts/card_pages.json` — parent of every card page (title ends `· <System> · Cards - Pages`), so the shared voice commands reach all of them.
- `Scripts/codebase_tools/new_card_system.py` — makes a new one.

## Pinned numbers (the same on every card page)

15 search · 20–21 breadcrumb · 25 Edit · 26 Undo · 28 New folder · 29 Folder options · 30 Add… · 31 Sort · 32 Group · 33 Filter · 34 a system's own folder option · 35 Sub-folders · 36 Ask Claude · 37 Description · 39 the folder's / item's heart.
A system pins its own in what's left: Music 11–14, 16–19, 22–24, 27, 38, 40–50; Food's item pages 11–14, 16–19, 22–24, 27, 38, 40–41.

## Voice

On any card page (context `card_pages`): **edit folder**, **done editing**, **undo remove**, **folder up**, a number, several numbers, and the action words **favorite / open / menu / delete N** and **start grab**. A system adds its own words in its plugin (Music: play, skip; Food: had, made, hide, keep).

## Making a new card system

```
py Scripts/codebase_tools/new_card_system.py Movies --port 8306 --item movie
```

It writes `Scripts/movies/` (store with a starter item model, server, plugin JS and CSS), an `apps.json` row (`"cards": true`), the `movies_page` context under `card_pages`, `OpenMovies` in CardPages.ahk, and the voice rows "open movies" / "movies <words>". Then: `ReloadWithNotice` (the context) and one Caster restart (the voice rows). `--undo` takes it all out again. Tested end to end 2026-11-12 with a throwaway system.

## How Music moved over (2026-11-12)

`music_folders.py` keeps every name the other music modules import, bound to the engine (`MusicSpec` holds the music identity: album = artist|title, the play-count sorts, source / artist groups, `source_pref`). The views before and after were compared folder by folder: identical. The old 2,400-line page became the shared kit plus `music_folders_page.js` (recommendations, moods, search, the player bar) and `.css`. The music tests (40) and the kit + Food tests (16, `test_card_kit_and_food.py`) all pass.
