---
tags: [design, architecture, web, viewers, newtab, settings]
related: ["[[NEW_TAB_PAGE]]", "[[MEDITATION_PAGE]]", "[[SETTINGS_SYSTEM]]", "[[ART_SYSTEM]]", "[[REMINDERS_SYSTEM]]"]
status: built (v1) — 2026-10-06
updated: 2026-10-06
---

# The local app hub — one header over every Chrome page

**Status: BUILT.** Every local page (new tab, meditation, reader, art, birds,
tarot, alarm clock, reminders) carries the same header: a bar of every page.
**Alt+`** shows or hides it.

## How it behaves (Jamie's choices, 2026-10-06)

- **Open by default.** Hiding it (✕ or Alt+`) shows a small **◇ pages** tab
  in its place.
- **Each page remembers its own open/shut** (localStorage; every server is
  its own origin, so the memory is per page for free).
- **The reader starts shut**, and its tab is a near-invisible **◇** at the bottom
  left, right after the reader's ⚙ (`data-header="collapsed"
  data-pill-after="#gear"`).
- **Fullscreen hides the whole header**, tab included: the page's own
  fullscreen, or Chrome's F11 (seen as a window the exact size of the screen).
  Opening it by hand in that visit overrides this.
- **From the new tab page, every link opens a new tab** (`data-links="newtab"`).
  Ctrl/Shift flips that. When the target's server has to start first, the tab
  is opened blank inside the click and pointed at the page once it's up. A tab
  opened after the server answers would be eaten by the popup blocker.

## Why not one server

Jamie's worry was that separate servers made a shared navigation impossible.
They don't. The pages already follow one address shape
(`localhost:<port>/<thing>`), so moving between them is just a link. What they
lacked was a **list of what exists** and **a header that reads it**. Merging the
servers would have bought a page load's worth of speed and cost:

- one crash takes every page down;
- editing any page restarts all of them (each server holds its code in memory
  — the stale-code trap);
- the reader (5,000 lines) would swamp the rest.

**This is a default, not a rule (Jamie, 2026-10-19):** "I really don't care
about" a crash taking more than one page down. A new page may live as another
route on an existing server when it reuses that server's machinery. The Writing
page is the first one: it is `/write` on the new tab server, because it is the
new tab's panels with different content ([[WRITING_PAGE]]). Choose by what the
page reuses, not by isolation. A big, independent page like the reader still
gets its own server. `apps.json` rows may share a port and differ by `path`.

If one address ever matters, the new tab server can **proxy**
(`/meditate/` → :8295) later without rewriting a page. Not built; not needed
unless switching ever feels slow.

## The three pieces

| Piece | Where | What |
|---|---|---|
| Registry | `INIDATA/apps.json` | one row per page: id, label, icon, port, path, server script (+ `python: "anki"` for birds, `wait` for slow starters). Order = header order. **Adding a page = one row.** |
| Kit (server half) | `Scripts/viewers/local_viewer.py` § THE KIT | `kit_get` / `kit_post`: `/kit/<file>`, `/kit/apps` (registry + live status, pinged in parallel), `/kit/settings`, `POST /kit/launch?app=`. `ViewerApp` wires these in automatically; the reader and art call them from their own `do_GET`/`do_POST`. |
| Kit (page half) | `Scripts/viewers/kit/kit.js` | ONE script tag. The header, `Kit.openSettings(system)`, `Kit.post/get/toast`. `data-header="pinned"` (new tab) / `"off"`. |

### Every server serves /kit/ itself

So a page never makes a cross-origin request. A header fetched from the new tab
server would need CORS, and would disappear whenever that one server was down.

### Starting a server that's down

A header link to a page whose server is down does **not** fail. The page asks its own
server to `POST /kit/launch?app=<id>`. That server starts
`<python> <script> serve` (windowless, detached), waits until it answers, and
hands back the URL. Ctrl/middle-click opens it in a new tab.

Launching is a **POST to the page's own origin**, never a GET like
`/go/<app>`. Starting a process is a write, and the origin rule ("GET reads only;
writes need the server's own Origin") stays one sentence long only if nothing
is excused from it.

### The web settings panel

`Kit.openSettings("meditation")` renders the **same Settings-system schema** a
Miller's ⚙ Settings row renders. So `settings.py declare` gives a page its settings
with no UI code, the same deal Millers already had. An enum is a row of
buttons, never a text box (pick-from-a-list rule). `settings.py`'s
`--source command` options now run with `CREATE_NO_WINDOW`, because the panel
calls it from a `pythonw` server and a console child of a windowless parent
gets its own console window.

### Jump labels and the title marker (2026-10-14)

Saying "jump 7", or just "7", clicks the control labelled 7 on any local page.

- **A page opts controls in with `data-jump`.** That attribute is all a page
  ever does. `data-jump="1"` pins a number. `data-jump-group` on a container
  numbers its items together, so the new tab page reads panel by panel instead
  of across columns. `data-jump-pos="corner|badge|left"` picks where the label
  sits. Leave the "..." buttons and settings unmarked: Vimium covers those.
- **Only what's on screen is numbered**, and numbers are recalculated once
  scrolling stops, so a folder of 300 albums still reads 1-30.
- **Labels:** dark rounded labels, 14px bold. They sit in a fixed overlay layer
  below the hub header. The numbers are never added inside a page's own markup,
  with one opt-in exception below.
- **Inline numbers (2026-10-18):** `data-jump-pos="inline"` draws no floating
  label. Kit writes `data-jump-n="7"` onto the element (and `data-jump-hit` for
  450ms when it's said), and the page styles the number itself. The new tab page
  uses it: each row's `›` turns into its number in the panel's accent colour, and
  ↻ reads "↻ 22". `jump_labels` off removes the numbers and the `›` comes back.
  Use it on any page that has a gutter of its own; the overlay stays the default.
- **Settings:** the shared `pages` settings system (`jump_labels` on/off,
  `jump_size` 12-20 px, `jump_opacity` 40-100 %). It is reached from the hub
  bar's ⚙. `Kit.openSettings(<any system>)` also adds it as an "All pages"
  section to the bottom of every page's own panel, and those rows save to
  `pages`, not to the page's system.
- **Which pages have numbers:** new tab, music folders, timers, reminders, the poem studio (print + carousel),
  (section nav + check-in buttons) and meditation. Meditation packs are pinned
  to the number the card already shows. Its tracks are unmarked, because they
  already show their series number. Health has no controls, so nothing is
  numbered there.
- **Title marker:** kit.js makes every title end " - Pages" and puts the suffix
  back when a page retitles itself. Caster and the context detector tell "one of
  my pages" apart by title alone. The shared Caster rule is
  `rules/local_pages_commands.py` (bare `<n>`; the reader is left out because it
  reads typed digits itself). The shared context is
  `INIDATA/Contexts/local_pages.json`, and every page's context now has it as
  its parent. A page's own rule still scopes on its own title.
- **Delivery is keystrokes, not HTTP.** `LocalPageJump(n)`
  (`Helpers/LocalViewer.ahk`) sends Ctrl+Alt+Shift+F9, the digits, then Enter.
  It is reached from ListNav's `nth_click_function` and from the bare-number
  rule. On the chord, kit parks focus in an off-screen input, because Vimium
  would otherwise read the digits as a count. A window-capture listener also
  hides the keys from the page. Going this way meant no server changes and no
  restarts.

## Rules for the next page

- Put `<script src="/kit/kit.js" defer></script>` in its `<head>`, and add a row
  to `apps.json`.
- Build it on `ViewerApp` and the kit routes come free. On a hand-rolled
  handler, call `lv.kit_get` before your routing in `do_GET` and `lv.kit_post`
  right after the origin check in `do_POST`.
- **Every control is a real `<a href>` or `<button>`.** Jamie uses a link-hint
  extension. For the same reason there's **no shadow DOM** in the kit: hint
  extensions don't reliably reach inside one, so kit styles are plain and
  prefixed `kit-`.
- **Mark its everyday controls with `data-jump`** (see Jump labels above). Give
  its context `"parent": "local_pages"`. Never anchor a title regex at
  `- Google Chrome$`: the marker comes before that.
- **The pinned header is `position: fixed` plus body padding, never in flow.**
  In flow, it became a flex item beside the new tab page's columns, because that
  page's `<body>` is a flex row.

## Not done / later

- "Watching" has no page of its own yet (it's a new tab panel), so it's not in
  the header.
- Proofread (:8765) is left out. The poem studio (:8296) joined 2026-10-18: header
  link opens the carousel (`/carousel/` = the last poem), both its pages load kit.js
  and number their buttons (option grids pin 1..N). The alarm clock is in, but its own page shows the header only
  after its server next restarts. It holds live timers, so it wasn't restarted.
- Moving existing pages' own CSS onto shared tokens. Only the header was added
  to them.

Tests: `Scripts/codebase_tools/tests/test_meditation_page.py` (kit file
serving, unknown-app refusal, registry well-formed: unique ids and ports, every
script exists).
