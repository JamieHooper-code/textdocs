---
tags: [programming, design-doc, food, cooking, recipes, card-kit, local-viewer]
created: 2026-11-12
updated: 2026-11-12
status: v1 built 2026-11-12 — page, folders, smart folders, picker, light log, photos, Copy Me That import
related: ["[[CARD_FOLDERS_KIT]]", "[[MUSIC_FOLDERS]]", "[[LOCAL_APP_HUB]]"]
---

# Food — what to eat and what to cook, as photo cards

Built on the shared [[CARD_FOLDERS_KIT]], like her music folders. Page on
`http://127.0.0.1:8305`; "open food", "food <words>" ("food quick", "food
chana masala", "food curries"). System "Food" in her to-do lists.

## Why it's shaped like this

From her own words (2026-10-26, 2026-11-12): food anxiety and indecision, a
physical disability that means eating the same things a lot, sometimes bored of
them and sometimes comforted by them, and forgetting the classics (PB and J,
pita and hummus). So:

- **Image cards in folders** she makes: Curries & Stews, Fermenting, Easy Comfort, Healthy Comfort, Overnight Oats. Clicking a card opens the food: photo, recipe, notes, flavours.
- **Going in already knowing**: smart folders at the top — *What to eat* (a dashboard), *Haven't had in a while*, *Quick and easy*, *Big batch next*, *What do I need?*, *Want to try*, *Favorites*, *Everything*.
- **Effort on every food**: no cooking / a little / some cooking / a project. "Quick and easy" and the picker respect it, for low-energy days.
- **Not a food log.** Four presses only, when a nudge comes up or after cooking: *Had it recently*, *Made a batch*, *Hide for a while* (2 weeks to 3 months), *Keep showing me this*. Each food has its own "remind me after" gap (defaults: 1 month for a food, 6 weeks for a recipe or batch dish, 3 weeks for an oats flavour).
- **Overnight oats as a family**: the family's page lists its flavours, each its own card, and "Oats you haven't had lately" rotates them so the same one doesn't repeat. "Add a flavour" (23) adds hers.
- **Real photos only.** `food_photos.py` takes them from Wikimedia Commons (every file is a real, free-licensed photo with a named author; the credit shows on the food's page). Never AI-made. "Change photo" (16) shows ~16 numbered candidates; a flavour looks for oats photos first, then its own ingredients, and no photo is used for two foods. Imported recipes keep their own recipe photo.
- **Hearts** like music: more presses = more loved; below zero buries it (never suggested).

## The picker — "What do I need?"

Three numbered questions: *Familiar or new?* · *How much energy?* (barely any / a little / I can cook) · *Comfort or healthy?* Each answer is a button; with all three answered the cards below are what fits, favourites and overdue ones first. The answers live in the address (`pick~familiar.none.comfort`), so back and voice numbers just work.

## Nudges

- The *What to eat* dashboard's first row and the *Haven't had in a while* folder.
- The new tab page: one "haven't had X lately" a day (`/api/nudge` on :8305, `food_store.nudge()`), linking to that food.

## Copy Me That

Only she can download the export: on copymethat.com, **More → Download Recipes → HTML → Download** (a .zip with recipes.html and an images folder). Then say **"import copy me that"** — it finds the newest export in Downloads (`food_import_cmt.py --find`). Every recipe comes in with ingredients, steps, notes, servings, its source link and its photo; Copy Me That's categories become folders inside *From Copy Me That* (move them wherever); starred ones get a heart; ones she never made are tagged "to try". Re-importing updates the recipe text and never touches what she set here (effort, notes, gaps, hearts, folders).

Copying changes back to Copy Me That (it has no API: it would be the Chrome extension driving the site) is the one piece not built yet.

## Files

- `Scripts/food/food_store.py` — the data model: `FoodSpec`, items, the log, smart folders, the picker, item pages and their buttons, the CLI (`seed | items | add-food | had | made | hide | keep | unhide | due | nudge` + the kit's).
- `Scripts/food/food_server.py` — the CardSite on :8305 (+ `/photo/<file>`, `/api/nudge`).
- `Scripts/food/food_page.js`, `food_page.css` — Food's words (had / made / hide / keep N), its card lines and badges, the picker's questions, the warm look.
- `Scripts/food/food_photos.py` — Commons search, pick, download (`find`, `fill`).
- `Scripts/food/food_import_cmt.py` — the Copy Me That import.
- `Scripts/food/food_seed.py` — her first folders and foods.
- Data: `INIDATA/Food/food.json`; photos `C:\Users\jamie\FoodData\photos`.
- `Helpers/CardPages.ahk` — `OpenFood`, `FoodImportCopyMeThat`.

## Item page numbers

11 Had it · 12 Made a batch · 13 Hide for a while · 14 Keep showing · 16 Change photo · 17 Notes · 18 Effort · 19 Remind me after · 22 Add to a folder · 23 Add a flavour · 24 Big batch? · 27 Rename · 38 Tags · 40 Open the recipe · 41 Delete · 39 its heart.

## Later

- Copy Me That write-back (the extension driving the site).
- Folders for fermenting projects with dates (started / ready by).
