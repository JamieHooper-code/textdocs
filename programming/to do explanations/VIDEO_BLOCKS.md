---
tags: [ahk, chrome-extension, video, youtube, design-doc, voice]
related: ["[[CHROME_EXTENSION_IDEAS]]", "[[CHROME_BRIDGE_EVENTS]]", "[[GLOBAL_CLICK_HINTS]]"]
status: BUILT 2026-11-12. v5 same day: boxes past the picture edge, auto blocks start soft + grown, blob frost. v4: panel moved to a numbered, voice-named kit page (Vimium works). v3: side panel on the touchscreen, soft frost, make block never grabs a covered cam, "make frost" = finder. v2: edit keys pass AHK + Vimium (focus trap), Shift grow / Ctrl shrink, video>playlist>channel defaults, block/frost save <name>, run records, joint-rect detector. Live retest pending.
created: 2026-11-12
---

# Video blocks: cover a streamer's face cam

Jamie watches streams and playthroughs (Robert Rambles' WoW series first) and
would rather not see the streamer's face cam. "make block" finds the cam and
covers it; the box is remembered for that channel and comes back by itself.

## Under YouTube's controls (2026-11-12)

On YouTube the blocks layer lives INSIDE #movie_player (its own stacking layer)
at z 12: above the video container (10), below the spinner (18), the dark
gradients (24-25) and the title bar / control bar (58-59), so pausing or
hovering shows YouTube's controls ON TOP of the blocks (seen in the built-in
browser: control bar, progress dot and captions all over a black box).
#movie_player has no transform, so position:fixed still means the viewport.
While editing / picking / the make-frost cover is up, the layer moves back to
the top of the page so outlines and numbers are never hidden. Other sites:
always on top (ensureLayer / playerBox in blocks.js).

## How it fits together

```
"make block" -> MakeBlock (Helpers/VideoBlocks.ahk)
   -> bridge op block.frame      (blocks.js: the video frame as a JPEG data URL)
   -> Scripts/video_blocks/detect_cam.py   (OpenCV: face -> the cam's border)
   -> bridge op block.propose    (dashed box in edit mode; Enter saves)
```

- **Boxes are fractions of the picture,** not screen pixels: the overlay reads
  the `<video>`'s box, undoes object-fit letterboxing, and places each block on
  the picture. Fullscreen, theater, a small window or the other monitor all
  land on the same spot. In fullscreen the layer is moved inside the
  fullscreen element so it stays visible.
- **Presets per scope:** YouTube channel (`yt:@handle`, read from the owner
  link once the watch page shows the new video) or site (`site:host`). Each
  preset is named by where it sits: bottom left, top right, left, middle...
  Saving a box at a spot that already has a preset overwrites it (he moved or
  resized the cam). `on` = what was up last time; it's re-applied on every
  video from that channel.
- **"chat"** is a special preset: hides YouTube's live-chat panel (CSS), no box.
- **Styles:** black, blur (backdrop-filter), icon (the channel's avatar), and
  pictures: files in `ChromeExtensions/macro-bridge/block_art/` listed in its
  `index.json` (e.g. a Cataclysm logo for WoW).
- **Storage:** `chrome.storage.local` key `blocks` (read on every page load, no
  round trip). Every save emits topic `blocks`; `consumers/blocks_consumer.py`
  mirrors it to `INIDATA/video_blocks.json` (read-only mirror).

## Defaults: video > playlist > channel (2026-11-12)

Presets (named boxes) belong to the channel/site. Which presets come up on
their own is a DEFAULT, kept at three levels that override going down: this
video's default beats this playlist's, which beats the channel's. The most
specific level that has one is "the level in effect"; Enter, block off,
cycle and toggles write there. `block default <level>` copies what's up to a
level, `block forget <level>` drops a video/playlist override.

## Voice

| Say | Does |
|---|---|
| make block | detect the cam, propose a box. Cam already covered (or no cam found) = a fresh box in the least-used spot, never a grab of the existing one |
| block fix | the cam moved: detect again; a saved spot that matches is switched to with no editing, a new spot is proposed |
| make frost | find still overlays across the video; numbered candidates, digits choose, 0 = all, Enter frosts them (presets "frost <spot>"). A hand-made frost box = make block, then S |
| block panel | the side panel on the touchscreen (below) |
| block text / block text find | frost blocks: a small heavier blur over each line of text inside (finds it the first time; "find" looks again). See "Text spots" below |
| block save <name> / frost save <name> | name the box that's up and save it (frost save also frosts it) |
| block load <name> | toggle a named box |
| block bottom left / top right / ... / chat | toggle a saved spot (chat = hide YouTube live chat) |
| block cycle | one box at a time through this channel's presets |
| block grab / block grab two | numbers over every block that's up (drawn above them); a number (key, "block grab two", or "pick two" on the panel) edits that one. One block up = edits it straight away |
| edit block | the box that's up back into edit mode |
| block black / frost / blur / icon | change the style of the box that's up |
| block default video / playlist / channel | what's up becomes that level's default |
| block forget video / playlist | drop that override |
| unblock | off on this page; comes back next video |
| block off | off at the level in effect |

Rule: `caster/rules/video_blocks_commands.py` (Chrome). Diagnostic: `MAINFUN.bat ShowBlockState` (shows every level).

## Edit mode keys (ONLY while editing; the page owns the keyboard until Enter/Esc)

arrows move 0.5% · Shift+arrow pushes THAT side out (grow) · Ctrl+arrow pulls
the OPPOSITE side in (shrink: Ctrl+Up = bottom edge up) · F/G scale 6% ·
1 3 7 9 corners · S / Shift+S style · Tab next box · Delete remove ·
T text blur on/off (Shift+T find again) · ; ' text blur · { } text reach ·
Enter save · Esc cancel. The chip shows the last key it received.

**Why keys used to vanish (2026-11-12, her first live try):** AHK's YouTube
Left/Right bindings (`YoutubeBackTenSeconds` / `YoutubeSkipForwardTen`) took
the arrows before Chrome saw them, and Vimium took G and S (its listener runs
before a document_idle content script). Fix: in edit mode focus sits in an
invisible `<input aria-label="video block edit">` in the LIGHT DOM (a closed
shadow root hides it from UIA). `IsFocusInTextField()` is then true, so every
binding with guard `not_in_textbox` (both YouTube arrows have it now) lets the
key through, and Vimium and YouTube both ignore keys typed into a text box.
blocks.js's window capture listener swallows them so the input never fills.

## Soft frost (2026-11-12, v2: progressive)

Every box has SOFTNESS 0-100 and STRENGTH 1-10 (frost blur, scaled to the
picture height). Frost defaults to soft 45 / strength 5; black stays crisp
unless softened. Edit keys `[` `]` softness, `-` `=` strength; the panel has
softer / sharper / more / less frost.

v1 faded ONE uniform blur's OPACITY with two straight-line gradients
multiplied together. She saw "a perfect box within another perfect box": the
sharp picture ghosting through a sharp-edged blur, with rectangular falloff.
v2 ramps the BLUR AMOUNT itself: 5 stacked layers (FROST_LAYERS), light blur
reaching nearly to the edges, heavy blur only in the centre, each faded in an
OVAL that TOUCHES THE SIDES (radial-gradient, ellipse closest-side) so it
fades out inside the box: a blob in the square, corners clear. (farthest-corner,
through the corners, ran past the sides: the light layer was full-on at the box
edge and got cut off in straight lines -- the square she still saw at soft
100.) Fine SVG fractal-noise GRAIN on top, same blob; plus a faint tint on the same oval. Softness =
how much of the box is ramp (under 5 = one plain uniform layer). The layers
are CHILDREN of the box, which must carry no filter / opacity / mask, or it
becomes their backdrop root and they blur nothing. Black (and pictures) soften
with the same oval opacity fade. 7 layers made the preview pane time out
rendering; 5 is the setting. In edit mode the dashed outline is its own layer
(`.ol`) so the fade can't hide it.

## Boxes past the edge, and soft auto defaults (2026-11-12)

**A box may hang off the picture, up to half of itself** (`fit`: x in
[-w/2, 1-w/2], size up to 1.5). Boxes draw inside a `.clip` exactly the size
of the picture, so the off-picture part is cut away; the edit outline is not
clipped, so she sees where the box really ends. Why: with soft blobs, a box
stuck inside the frame can never put its thick middle on a screen corner --
her blobs jammed against the left/bottom/top edges.

**Auto-detected blocks start the way she tunes them by hand** (`AUTO` +
`autoBox` in blocks.js): frost, very soft, bigger than the thing on every side
(`grow` x its size), and wherever the thing touches the picture's edge the box
goes PAST that edge (`past` x its size).
- cam (make block, block fix): soft 100, frost 7, grow 0.15, past 0.35 -- from
  her own Robert Rambles cam block (about +10%, soft 100, frost 7).
- overlay (make frost picks): soft 80, frost 4, grow 0.2, past 0.45 -- her goal
  bar was +40% wide, soft 50.
- block fix keeps the moved block's own look (`takeLook`); a re-found saved
  spot keeps its own. The run record's `proposed` is still the RAW detection,
  so the detector is tuned on what it found, not on the grown box.
Tested on the dump frame: the cam corner frosted right into the screen corner;
a goal-bar pick became a soft smudge reaching the top-left corner.

## Stacking, her measured defaults, pictures (2026-11-12)

**Defaults are HER tuning, measured** ("that looks really freaking perfect"):
`AUTO` in blocks.js has `long` / `short` axis amounts. Cam: +12% on the open
sides, 14% past the edges it touches, soft 80, frost 10. Overlay: +36% along
its length and 23% past its edge, only +5% across, soft 50, frost 1 --
overlays stretch along their length. Keep new tuning in that one place.

**v2 defaults (later 2026-11-12, "that should be the new default"):** her
"bottom left solid" on Robert Rambles became the cam default: a SNUG SOLID
box (`AUTO.cam`: +5% on the open sides, 7% past the screen edges, soft 0) with
edge fuzz 4, and every picture starts at size 52%, brightness 90%, 70% solid
(`PIC` defaults). Which picture stays per channel (new blocks start black).
WoW videos start the solid cam box WITH a picture (her call): Burning
Crusade logo for a Burning Crusade / TBC video, the Cataclysm W for any other
World of Warcraft video, judged from the title + YouTube's Game tag
(wowPicture / camBox in blocks.js); everything else starts black.
Her earlier soft frost blob is now `AUTO.camFrost`, the SECOND layer: "make
block" on a covered cam adds the other look (solid if it's frosted, frost
blob if it's solid; both = a fresh box). Tested: a fresh make block landed
within 0.15% of her box.

**Stacking at one spot:** "make block" on a cam that's ALREADY covered adds a
SOLID black box exactly on the detected cam (named "<spot> solid"), so a soft
frost blob and a crisp black box can live on the same cam and be toggled
separately. If both exist, the next is a fresh box in a free spot.
`uniqueNames()` on save: two boxes that would share a spot name get "<spot> 2"
instead of overwriting each other. Her Robert Rambles channel now has
"bottom left" (frost blob), "bottom left solid" (black, exact), and
"frost top left" (goal bar).

**Pictures:** block_art/ holds real WoW art from Wowpedia (wowpedia.fandom.com):
Alliance and Horde crests, shaman crest, WoW Classic logo, the Cataclysm "W",
a cut-out murloc and the blue Time-Lost murloc -- full-size transparent PNGs,
shrunk to 900px max (sources per file in index.json `_source`). To fetch
more: the MediaWiki API (action=query&prop=imageinfo&iiprop=url) gives the
file URL; add format=original or Fandom serves WebP. The first set (56px
Wowhead game icons blown up) looked terrible -- never use game icons for this.
warcraft.wiki.gg refuses scripts (bot check); Wowpedia does not. A picture shows on black, 92%
of the box. Switching: `P` while editing / "next picture" / "block picture"
step through pictures only; `S` steps through everything; "block black" /
"block frost" jump straight back. Panel: 29 next picture (editing), 87 next
picture (details; its label shows which one).

**Edge fuzz** (fuzz 0-10, per block, rides in PIC): on a solid (black / picture) box the black is its own layer blurred by fuzz x 2px at 1080p, so the edges fade inward AND outward with rounded corners, while a picture on top stays crisp. Keys , and . while editing; panel 19 fuzzier edges / 20 crisper edges (words fuzzier, crisper, more fuzz, less fuzz). Different from softness, which makes an oval blob.

**Picture settings, per block** (`PIC` in blocks.js, saved with the preset):
picSize % of the box (10-100, default 92), picBright % (10-150, default 100),
picOpacity % (5-100, default 100; lower = fainter, the black shows through),
picGray % (0 or 100). Panel section **Picture** (shown when the block being
edited or selected shows a picture or the channel icon; its title shows the
values): 42 picture smaller / 43 bigger (10% steps), 47 dimmer / 48 brighter,
49 fainter / 50 more solid, 44 grayscale (toggle). Words: "picture smaller",
"picture bigger", "dimmer", "brighter", "fainter", "less faint", "grayscale".
The panel updates its own copy on each press, so quick repeat presses add up
(they used to all start from the last refresh and count once).
Burning Crusade logos added (wow-burning-crusade = stacked, for a squarish
box; wow-burning-crusade-wide = for a long overlay).

## The side panel ("block panel", 2026-11-12)

A LOCAL KIT PAGE, `127.0.0.1:8293/blocks` (new tab server; files in
`Scripts/video_blocks/panel/`), in its own normal Chrome window titled
"Video Blocks - Pages", FULL SCREEN (maximized) on the touchscreen
(secondary, portrait 1080x1920) monitor by `OpenBlockPanel`. Layout: what is
playing and the voice commands at the TOP (to read), every button at the
BOTTOM (to reach).

**Why a local page and not an extension page** (v1 was
`chrome-extension://.../blocks_panel.html`): Chrome lets no extension script
another extension's pages, so Vimium link hints, her DOM click hints and kit's
numbers could never reach it. As a kit page it gets all three, plus the kit
title marker that voice rules scope on. (A popup-type window drops the
" - Google Chrome" title suffix the local-pages rule needs, so it's a normal
window.)

**A voice commands section** at the bottom lists the common commands (spec key `voice`, display only).

**Every control is numbered and named from ONE list,**
`Scripts/video_blocks/panel/panel_spec.json`: pinned kit number (1-99), label,
words. The page builds its buttons from it; `caster rules/
video_blocks_panel_commands.py` (scoped to the panel's title) reads the same
file, so each word presses that number the way a spoken number does (kit chord
Ctrl+Alt+Shift+F9, digits, Enter). Bare numbers come free from
local_pages_commands.py. Edit the spec, not the page or the rule.
- 1-9 make block / block fix / make frost / plain box / edit / unblock /
  block off / live chat / block grab (the global block phrases keep working: see routing)
- 11-14 move, 15-18 grow a side, 21-24 shrink a side, 25-28 smaller / bigger /
  style / next box, 31-34 corners, 35-38 softer / sharper / more / less frost,
  39 remove box, 40 cancel, 41 save ("save box", "keep it")
- 45 all/none, 51-59 "pick one".."pick nine" (make frost candidates; 40/41
  are cancel / frost them there too)
- 61-69 a block row's on/off ("toggle one"), 71-79 its details ("open one");
  81-86 black / frost / icon / edit it / rename / delete
- 91-95 default for this video / playlist / channel, forget video / playlist
Words left out because they're taken: plain "save" (Ctrl+S), "cancel" (global
Escape -- which the panel passes to the video while editing, so it cancels
anyway), "done" (health check-in), "spot ..." (Spotify).

**How it reaches the video:** the page has no chrome.*. blocks.js runs on it
too and, on that page only, relays `window.postMessage` calls ("panel relay")
to the service worker: kind `blocks.relay` (a block.* op) and `blocks.run`
(MakeBlock / BlockFix -> topic `blocks.do` -> blocks_consumer.py ->
`Scripts\MAINFUN.bat <fn> "<video tab title>"`). An orphaned relay (after an
extension reload) stays silent, or its stale answer beats the live one.

**BLOCKS TARGET** (background.js): the video tab the panel is for = the last
tab she was on in a normal window that blocks.js has seen a VIDEO in (kind
`blocks.video`), kept in storage.session; with nothing stored, the video tab
with the newest `lastAccessed`. And any block.* op the bridge would deliver to
the panel page itself (a voice "make block" said with the panel in front)
goes to the target instead. Diagnostic: bridge op `ext.blockstarget`.
A video that starts PLAYING claims the panel too (blocks.js 'play' listener ->
`blocks.video {playing}`), as does one already playing when the extension
reloads. Bug 2026-11-12: after a reload every tab reported at once and the
parallel read-modify-writes of `blocksVideoTabs` lost entries (her Robert
Rambles tab never made the list, so the panel stuck on a Numinosa video);
`noteVideoTab` now runs one at a time on a promise chain.

Tested 2026-11-12 in her Chrome: panel opened on the touchscreen, showed
Robert Rambles + the saved "bottom left" block; mimicked "open one" (a word)
opened its details, "seventy one" (a bare number) closed them; a block.state
aimed at the panel window landed on the video tab. Not yet seen: Vimium's
hints on it (they're real <button>s, so they should).

## Every detection is kept

`MacroData\video_blocks\runs\<yyyyMMdd-HHmmss>\`: frame.jpg, detect.png
(face + proposal), result.json. When she saves (Enter or block/frost save),
`blocks_consumer.py` adds final.json and result.png (green = proposed, blue
= saved) and a line in `MacroData\video_blocks\runs.jsonl` with how far each
side moved (% of frame). That file is the tuning record.

## make frost: still-overlay finder (blocks.js, 2026-11-12)

A stream overlay is the SAME across the whole video while everything else
changes. `findOverlays` seeks to 8 points spread across the video (picture
covered by a frosted "cover" while it jumps; position and play state restored
after; ~5s on a 6-hour VOD), grabs each at 384px wide, and keeps pixels that
agree with their own median (+-10) in at least 70% of frames AND sit on a
sharp edge of the median picture. Those grow into blobs (1.2% dilation),
neighbours merge, and blobs are ranked by edge pixels x nearness to the frame
edge; max 9, 2.5% x 1.5% minimum, 25% area maximum.
- "Most frames", not all: the goal bar's fill and dollar amount change over
  hours, and streams have break screens. Requiring every frame missed it.
- Watching a few seconds (v1) failed: when the character stands still the
  whole game is still and everything merges into one blob.
- Live video with under 2 minutes seekable: watches 8 frames 0.7s apart.
- Game HUD (WoW minimap) is just as still, so these are candidates she picks,
  never applied on their own. The face cam moves, so it is never offered.
- Tested: the Robert Rambles VOD (goal bar found exactly, plus the minimap and
  a left-edge UI strip); a fixture with a moving background and two fixed
  overlays (both found exactly; pick 1 + Enter saved "frost top left").

## Text spots inside a frost block ("block text", 2026-11-12)

Her idea: a light frost big enough to cover the whole goal bar overlay still
let the caption above it and the "$0 / $150" labels under it read through its
faded edges, and making the frost heavier or bigger blurred far more game than
needed. So a frost block can find the **lines of text inside it** and put a
small, heavier blur over each one, fading out over a margin. The block itself
stays light.

- **Per block settings** (in `PIC`, so they save, copy and `block.set` like the
  picture ones): `textOn` 0/1, `textBlur` 1-10 (same scale as frost strength,
  default 5), `textPad` 0-150 (how far each patch reaches past its text, as %
  of the text's height; default 40). The spots are stored once on the block
  as `textSpots: [[x, y, w, h], ...]` in picture fractions, so moving the box
  doesn't move them; a spot whose middle ends up outside the box is skipped.
- **Finder** (`findTextSpots`): the same seek-and-sample as make frost (8
  frames, still pixels on sharp edges of the median), at 768px wide, inside
  the block only. First the **rules** come out: long row runs (25% of the box
  wide, little gaps bridged) and tall column runs. Those are a bar's outline,
  and without this step they tied the caption, bar and labels into one piece
  (the first try found nothing because that one piece was too tall). The rule
  pieces wide enough are **bars**. The rest is text: letters smeared sideways
  into words and lines, not up/down, so stacked lines stay apart. Text on a
  bar's rows joins the bar, pieces on the same line within 2% merge, pieces
  inside another merge.
- **Drawing**: each spot is a `.tp` child of the box with its own backdrop blur
  and a two-gradient mask (intersect) that fades across the margin. Hard frost
  (softness 0) now draws its blur on a child layer too: a box with its own
  backdrop filter would be the patches' backdrop root and they'd blur nothing.
- **Robert Rambles result**: 4 spots: the "Help Support..." caption, the bar
  with its amount, "$0" and "$150". With text on, all four are unreadable
  under her frost top left (soft 50, frost 1); off, the caption and labels
  showed through.
- Ways to use it: voice "block text" (toggle; finds the first time) and "block
  text find" (look again); edit keys T (Shift+T finds again), `;` `'` blur,
  `{` `}` reach; panel section "Text in the frost" (96 on/off, 97 look for
  text, 88/89 heavier/lighter, 98/99 bigger/smaller), shown when a frost block
  is selected or being edited. Debug: `block.text {name, debug: 1}` returns
  the region's median with still edges in red, plus the pieces.

## The detector (detect_cam.py)

1. Haar cascades (frontal, alt2, profile, mirrored profile). OpenCV must be the
   **4.x** line: 5.0 moved `CascadeClassifier` out of the main package.
2. Sides near the frame edge snap to it (face within ~1.2 face widths).
3. The other sides are chosen TOGETHER: score every candidate rectangle by how
   much each free side looks like one straight border (mean |gradient| x
   fraction above a low bar, measured between the rectangle's own corners,
   middle 80%), minus a penalty when a border carries on PAST a corner (the
   top border running past a candidate right side = that side is a line
   inside the cam). Two free sides (a corner cam) = exhaustive search.
   Searches start right at the face edge: his head nearly touches the cam top.
   History: a fixed threshold + "first line outward" (v1) missed faint borders
   on a bright game and stopped on a doorframe inside the cam.
4. No face: the biggest corner rectangle whose inner horizontal edge runs from
   the frame side and whose inner vertical edge runs to the frame top/bottom.

Tested 2026-11-12: dump frame (cam 0-18.75% x, 75.7-100% y) -> 0-18.75%, 75.6%;
her live loading-screen frame (0-18.85%, 75.6%) -> 0-18.75%, 75.6% (v1 gave
0-15.5%, 68%); no-face corner path 0-18.85%. `--debug out.png` draws what it found;
`%TEMP%\video_blocks\last_detect.png` is the last live run.

## Not built / ideas

- Follow-the-cam polling: decided against (2026-11-12): "block fix" on demand instead.
- Therapy self-view hiding: set aside for now (a DOM selector preset would do it).
- A Stream Deck button for block cycle.
- frost find: BUILT (see above).
