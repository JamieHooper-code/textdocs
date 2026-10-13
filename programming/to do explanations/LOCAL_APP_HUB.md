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
- **The pinned header is `position: fixed` plus body padding, never in flow.**
  In flow, it became a flex item beside the new tab page's columns, because that
  page's `<body>` is a flex row.

## Not done / later

- "Watching" has no page of its own yet (it's a new tab panel), so it's not in
  the header.
- Proofread (:8765) and the poem studio (:8296) are left out (the studio needs
  a poem to open). The alarm clock is in, but its own page shows the header only
  after its server next restarts. It holds live timers, so it wasn't restarted.
- Moving existing pages' own CSS onto shared tokens. Only the header was added
  to them.

Tests: `Scripts/codebase_tools/tests/test_meditation_page.py` (kit file
serving, unknown-app refusal, registry well-formed: unique ids and ports, every
script exists).
