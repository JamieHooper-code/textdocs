---
tags: [programming, journal, chatgpt, reader, ahk, design, todo]
created: 2026-10-15
status: built 2026-10-15 (§ 6 design, § 7 what was built and verified)
related: ["[[JOURNAL_QUESTION_LOOP]]", "[[JOURNAL_SYSTEM]]", "[[READING_ROOM_BOOKS]]", "[[QUESTIONS_SYSTEM]]"]
---

# Replying to a ChatGPT chat from the reading room

**Goal:** answer a ChatGPT conversation without going back to the ChatGPT tab,
starting from the reading room's `chat` view (`grab read` → `chat:<entry id>`).
Jamie called this a nice-to-have, not a top priority.

**Today's loop:** read in the reader → jump to ChatGPT → type a reply → `grab chat`
(or `grab read`) again.

**Recommendation in one line:** keep the real ChatGPT tab as the only thing that
*sends*. Our side gives her a box to write in and does the jump for her:
**write → paste and send in the tab (about a second) → back to the reader → fetch
the reply through the API we already use → append it to the entry → the reader
reloads on the new page.** Don't build a direct send to ChatGPT's backend.

---

## 1. Can the backend we already read from also SEND?

**Technically yes. In practice, no, not for us.**

What we read today (`Scripts/journal/chatgpt_api.py`) is
`GET /backend-api/conversation/<id>` with a bearer token from
`/api/auth/session`. That works because reads are protected only by the token,
plus Cloudflare's "does this look like a browser" header check, which
`BROWSER_HEADERS` gets past.

Sending is a different endpoint (`POST /backend-api/conversation`, now
`/backend-api/f/conversation` in recent web builds) and it is guarded on purpose.
Public reverse-engineering projects describe the guards like this. **I have not
checked any of it against Jamie's account**, because a test would send a real
message:

| Guard | What it is | Why it hurts us |
|---|---|---|
| `sentinel/chat-requirements` | a POST before every send that returns a one-time token | one more round trip; easy on its own |
| proof-of-work token | a SHA3 hash puzzle over a fake browser-fingerprint array | has to be re-solved whenever OpenAI changes the recipe |
| Turnstile token | Cloudflare's bot check, produced by running obfuscated JS | can't be done from plain Python without running their VM code |
| SSE stream | the reply streams back as server-sent events | doable, but more code |

Getting past the proof-of-work and Turnstile means **defeating an anti-bot control
on purpose.** That is exactly the line OpenAI's terms draw ("automated or
programmatic" extraction, getting around protective measures). It's also the line
I won't cross in code. And it would break every time OpenAI changes the recipe,
which projects that rely on it report as every few weeks.

The read path we already have is grey too: it's an undocumented endpoint, used by
the web app's own token. But it's read-only, low-volume and uses her own session.
It falls back to the scrape when it breaks. A send path has no such fallback,
because a half-working send can double-post or post to the wrong chat.

**Verdict: don't build the backend send.**

## 2. The options

### A. UI relay: reply box → the real tab sends it (RECOMMENDED)

Every hard piece already exists:

| Step | Existing piece |
|---|---|
| bring up *that* conversation and prove it's the right one | `_JQFocusConversation(chatId)` (JournalQuestions.ahk). It waits until the front tab's address really is `/c/<id>`. "Typing into the wrong conversation is the failure this whole check exists to prevent." |
| type multi-line text without sending early | `_ChatGPTTypeMultiline` (handles the "newline = Enter" trap) |
| focus the composer, then send | `SendToChatGPT(text, Map("paste", false, "submit", true, "newChat", false))` |
| conversation id for an entry | the entry's `chat_url` (`set-chat-url`; `AskAIAboutEntry` links it at the source) |
| read the reply back | `ChatGptApiFetch(url)`, which is structured and complete, with no scraping |
| file only the NEW turns | `append-chat`, which already treats a re-grab as a superset and adds only unmatched turns |
| show it | `OpenJournalChatInReader(id)` / reader `/api/reload` |

New code would be:
1. **`ReplyToJournalChat(entryId, textFile)`** (JournalCapture.ahk). Focus the
   conversation, type, send, and go back to the window she was in. Then **poll
   `ChatGptApiFetch` every ~3s** until the last turn is the assistant's and has
   stayed the same across two polls. (Optionally parse `message.status ==
   "finished_successfully"` in `chatgpt_api.py`, which currently ignores it.)
   Then `_JAppendChat` and reload the reader. Runs windowless for up to a few
   minutes, so it needs `MainfunKeepAlive`. Add `MainfunClaim("chatreply:"
   id)` so a second reply to the same chat replaces the wait instead of racing it.
2. **A box to write in.** See B and C below. That choice is the real decision.
3. **Reader refresh on arrival.** The server is told to reload and jump to the
   last exchange. Until then the page shows a quiet "waiting for ChatGPT…" line.

**The honest cost:** the ChatGPT tab has to be in front for the one to two seconds
it takes to type and press Enter. Typing needs the active window, so it will flash
up. Mitigation: a ChatGPT window parked on another monitor (or a background
desktop), used only for relaying, so her reader never moves. The wait for the
reply is all API and happens off screen.

**Risks:** fragile only where `grab answers` already is (ChatGPT's composer
markup). Auth is her own browser session, so nothing new. ToS: this is her typing
into her own tab, just automated keystrokes, the same as voice dictation and
`grab answers` today. It's the lowest risk of the options.

### B. Reply box in the reader page

A textarea under the last exchange. "Send" POSTs `/api/chatreply` (a write, so it
goes in POST with the server's own Origin, per the local_viewer origin guard). The
server writes the text to a temp file and runs `MAINFUN.bat ReplyToJournalChat <id> <file>`.

- **+** never leaves the page. The reply appears where she's reading.
- **−** a browser textarea isn't the house writing box. No line numbers, no
  `draft_key` crash-proofing, no `tags:` markup, no voice rules scoped to
  `Writing box:`. Dragon dictation into a Chrome textarea works but is the weaker
  surface.
- **−** a draft lives only in the page until sent (save it to `localStorage` per
  entry as a minimum).
- Needs a `read_server.py` restart after the edit (stale-code trap).

### C. Writing box from the reader or the Miller (voice "reply chat")

Say "reply chat" in the reader (or use a row on the entry's Miller page). That
opens `WritingBox("window", …)` titled `Writing box: reply to <chat title>` with
`draft_key` = the entry id. Ctrl+Enter sends through A.

- **+** her real writing surface: crash-proof drafts, dictation, line numbers.
- **+** the Miller row gets it for free in every context, not just the reader.
- **+** no reader-page JS. Smallest build.
- **−** a window over the reader instead of inline.

**B and C can share A** and aren't exclusive. C first is the cheapest useful step.
B can come later if she wants it inline.

### D. The official OpenAI API (platform.openai.com)

The platform API **cannot reach web chats at all.** Its conversations are separate
objects with no access to her ChatGPT history, memory or custom instructions. The
only way to "continue" a web chat would be to replay our saved transcript as
context and file the answer in the journal. The chat would fork: the reply would
exist in her journal but never in ChatGPT.

- **+** fully headless, no tab, no focus, stable and documented, ToS-clean.
- **−** **costs money per message**, billed separately from a ChatGPT
  subscription. A 40k-character chat is ~10k input tokens *every* turn.
- **−** the conversation splits in two. ChatGPT's own memory and history never see
  it, and `grab chat` can no longer be the source of truth.
- **−** a new key to store (in INIDATA, gitignored, like the session token).

It's worth it only if she wants a **local "keep talking about this entry"** mode
that is deliberately separate from ChatGPT. That's a different feature from "reply
to this chat".

### E. Backend send (section 1): no.

## 3. Recommendation

1. Build **A + C**: `ReplyToJournalChat` plus a "reply chat" writing box,
   reachable from the reader (voice, scoped to the reader's `chat` view) and from
   the entry's Miller page.
2. Add **B** (an inline box in the reader) only if C feels like too much of a
   context switch in use.
3. Park **D** as its own idea ("talk about this entry locally"), not as a send
   path.
4. Never **E**.

Rough size of A + C: one AHK function of ~120 lines, a voice phrase, a Miller row,
and a small `chatgpt_api.py` tweak for completion status. It's on the scale of
`GrabAnswers`, which is the closest relative: it already focuses the linked
conversation and pastes into it. The only new ideas are pressing Enter for her and
waiting for the answer.

## 4. Open questions for Jamie

- Where should the writing happen: writing box (C), inline in the reader (B), or both?
- Is a ~1-second flash of the ChatGPT tab acceptable? Or should the relay use a
  parked ChatGPT window on another monitor?
- Should it **send** on Ctrl+Enter, or stop at "pasted, press Enter" the way `grab
  answers` does today? Auto-send is the whole point, but it's also the one
  irreversible step.
- When the reply lands, should the reader jump to it automatically, or show a
  quiet "new reply" note?
- Is D (a paid, local, forked conversation) wanted at all?

## 5. Traps to remember if this gets built

- **Never send to a conversation we haven't verified.** Reuse
  `_JQFocusConversation`'s address check. Don't write a looser one.
- **Trailing newline = Enter** in ChatGPT's composer (`_ChatGPTTypeMultiline`).
  Strip it, or the message goes before the box is done.
- **Don't trust "the last turn is assistant" on the first poll.** The stream is
  still writing. Wait for it to be stable or `finished_successfully`, or the
  journal files half a reply. `append-chat`'s superset matching would then add the
  full reply as a *second* turn, because the text differs.
- **Writes to the reader server are POST** with its own Origin. A GET returns 405,
  logged as `refused` in `ahk_event.log`.
- Keep the token-refresh path as it is. A stale token mid-wait is the routine
  10-day event (exit 3), not a failure.

---

## 6. DECIDED 2026-10-15: "show typing", the docked split (supersedes B vs C)

Jamie's decisions. This replaces the "writing box (C) vs inline box (B)" choice
in section 2. Option A (the real tab does the sending) stays the send path.

### What it is
**"show typing" / "show types"**, said anywhere in the reading room (a chat, poem,
journal entry or book), docks the REAL writing box over the right half of the
reader window. The reader lays its text out in the left half. It's the same box
with every feature (own process, `draft_key`, line numbers, markup, the
`Writing box` voice rules), docked, not rebuilt in HTML. It follows the reader
window when it moves or resizes (`WinEvents_On` location-change, not a poll).
**"hide typing" / "hide types"** undocks it, keeps the draft, and puts the
reader back to its full two-page spread.

### What Ctrl+Enter does
- **Chat view:** paste the reply into the linked conversation
  (`_JQFocusConversation` + `SendToChatGPT(..., submit:false)`) and **stop at
  "pasted"**. Jamie presses Enter herself. The ChatGPT tab flashing up is fine,
  so no parked window is needed.
- **Anything else (poem, book, entry):** it's **"journal this"** for the open
  item (JournalSources, reader `/api/state`), in the docked box.
- **Plain "journal this"** (not via show typing) stays exactly as it is today:
  the normal box, never docked.

### The wait (Jamie: "make sure it waits until the reply is fully finished")
1. Wait for HER message to appear in the conversation via the API. She sends it,
   not us.
2. Then wait for the assistant turn after it to be `finished_successfully` (add
   status parsing to `chatgpt_api.py`) AND unchanged across two polls.
3. Only then `append-chat`. Grabbing early stores half a reply, and the full one
   later lands as a second turn, because `append-chat` matches on text.
4. Never sent within a few minutes = give up quietly, keep the draft.
`MainfunKeepAlive` for the wait, `MainfunClaim("chatreply:" id)`.

### When the reply lands
The reader jumps to the new exchange **and the box vanishes** (draft cleared,
since it was sent), giving the full two-page reading view back.

### Page turning with focus in the box
- New functions **`ReaderPageNext` / `ReaderPagePrev`** turn a PAGE (not an item:
  `ReaderNext/Prev` already skip whole items). Jamie makes the Stream Deck
  buttons herself. These are what they call.
- Inside the docked box, **`>` and `<` are hotkeys** for those. Jamie only ever
  dictates those characters, so losing them as typed characters in that box is
  fine. Note: Dragon/Wispr dictating "greater than" into the docked box will turn
  the page rather than type it. That's accepted.
- MECHANISM: page turns live only in the page's JS today (`forward()`/`back()`).
  Add a server-side turn request (seq number on `/api/state`) that the page's
  ~0.5s state poll acts on. Do NOT ControlSend arrows into Chrome. It's unreliable
  for the render widget.
- TRAP: the page skips `reportView` unless `document.hasFocus()`, so two tabs
  don't fight. With focus in the box, the visible span would go stale after a
  turn and "grab 3 to 5" would number the OLD page. A remotely requested turn must
  report the view regardless of focus.

### Voice commands with the box in front
The docked box's title carries BOTH words, e.g.
`Writing box: ReadingRoom docked — <item title>`, so it is still a
`typing_box`/`Writing box` (markup rules, typing surface) AND a reading-room
surface. CATCH: `reader_commands.py` is registered with `executable="chrome",
title="ReadingRoom"`, and the title alone won't match an AutoHotkey64.exe window.
Those rules (plus `reading_commands.py` and the reading_room rows in
`generic_materializer_reading_room_commands.py`) need a `function_context` that
accepts Chrome+ReadingRoom OR the docked-box title. All their actions already go
through the server (`_ReadApi`), so only the scope changes.

### Focus after a grab
A grab can open the quote form or review editor, which take focus. When it closes,
focus returns to the docked box (the draft was never at risk: separate process).
Decided by Claude (unanswered question 7). Without it, the next keystroke lands
somewhere unexpected.

### Build order
1. Split + docking + `ReaderPageNext/Prev` + `>`/`<` + the view-report fix + the
   voice scope.
2. Chat reply: paste, the finished-reply wait, append, jump, box vanishes.
3. Journal-this mode for non-chat items.

---

## 7. BUILT 2026-10-15

All three stages of § 6 are in and verified live, except the one step that needs
a real message sent (see "Not verified").

### Where it lives
| Piece | File |
|---|---|
| show / hide, the docked box, `>` `<`, the reply wait | `AutoHotkey/Helpers/ReaderTyping.ahk` |
| `ReaderPageNext` / `ReaderPagePrev` (one PAGE, for Stream Deck) | same file |
| dock flag, page-turn queue, `chat_url` on the item | `Scripts/reader/read_server.py` (`POST /api/dock`, `POST /api/turn`, `DOCK` / `TURNSEQ` in the page) |
| reply finished? | `Scripts/journal/chatgpt_api.py` `tail_status` -> meta `tail_role` / `tail_status` (also on `ChatGptApiFetch`) |
| journal save shared with "journal this" | `JournalCapture.ahk` `_JournalSaveFromBox` (split out of `AddJournalEntry`) |
| box hooks | `SingleFieldSkipableInputGui.ahk` opts `on_show`, `submit_hook` |
| "is the reader in front" incl. the box | `ReaderFunctions.ahk` `ReadingRoomInFront()` (used by grab routing, "make grab", "grab music") |
| voice | `rules/reader_commands.py` "show typing / types", "hide typing / types"; scope = `window_context_helpers.reading_room_or_docked`; the reading_room context's `caster_scope.function_context` makes the generated materializer use it too (`_gen_context_materializers.py` now supports `function_context`) |
| tests | `test_chatgpt_api.py` (tail status), `Helpers/Tests/test_reader_typing.ahk` (which prompt is hers) |

### How it behaves
- The box is OWNED by the Chrome window: no Alt+Tab entry of its own, hides when
  Chrome minimises, hides while another tab is in front of the reader and comes
  back when the reader tab does. It sits over the right half of the PAGE AREA
  (the largest visible `Chrome_RenderWidgetHostHWND`), not over the toolbar.
- Esc = "hide typing" (draft kept, no clipboard rescue). "show typing" again
  restores the draft for that item (chat: per entry; else per item).
- Chat Ctrl+Enter pastes and stops; the box stays up with the draft until the
  reply is FILED, then vanishes and the reader opens on the newest reply.
- Waits: 10 minutes for her message to appear, then 5 for the reply to finish
  (`finished_successfully` on an assistant tail AND unchanged across two polls).
  Gives up quietly with a tooltip; the draft stays.

### Traps found while building
- **Re-activating an AHK Edit selects ALL its text.** Measured (caret 5-5 ->
  0-5 after Chrome and back). The next keystroke would replace the whole draft,
  and this box is re-activated constantly. The docked box saves the caret on
  WM_ACTIVATE-out and restores it on the way in. Other writing boxes probably
  have the same hazard -- worth checking.
- **Chrome has many render widgets**, most hidden; `...HWND1` was a hidden 44px
  one after a maximise. Pick the largest visible one. On a maximise Chrome
  re-lays the page area AFTER the frame moves, hence a second placement pass.
- **The docked box is its own context (typing_box)**, so anything that asks
  `IsForegroundContext("reading_room")` thinks the reader is gone. Grabs did --
  "grab 3 to 5" would have gone to the generic OCR engine. Ask
  `ReadingRoomInFront()` instead.
- **`ChatGptApiFetch`'s result is not `_JAppendChat`'s shape** (no `model`,
  `url`); the wait builds the capture Map GrabChatToJournal builds.
- **Testing from a Claude session:** windows never take focus (by design,
  `_GuiNoFocusSteal`), and AHK hotkeys ignore keys another AHK script sends at
  SendLevel 0. Neither is a bug in the box.

### Verified live
Dock/undock and the one-column left-half layout; remote page turns with focus
in the box (view span reported, so "make grab" numbered the visible page, 26
sentences, focus stayed in the box); `>` turns the page and is not typed; the
box follows a moved/restored/maximised reader; focus returns to the box when a
form window closes; "reader commands" and "hide typing" recognised with the box
in front (dragon_mimic); hide keeps the draft, show restores it; Ctrl+Enter
pasted into the right conversation and stopped (test text cleared, nothing
sent); the wait run against the conversation's existing last message found it,
waited for a finished stable reply, ran append-chat (nothing new) and reopened
the reader on the newest reply; journal mode opens with the same header/source
"journal this" gives.

### Not verified
- A REAL send: her message appearing after she presses Enter, then a reply
  streaming in. The logic is unit-tested and dry-run against a finished
  exchange, but the first real reply is the real test.
- Saving a journal entry from the docked box (would have written a test entry
  into her journal and used up a series number). It is the same
  `_JournalSaveFromBox` plain "journal this" runs.
- Focus on open when said by voice (Claude-launched windows never take focus).


### Added later on 2026-10-15
- **F11** in the docked box = the reader's fullscreen (the box re-docks full
  height). **Escape belongs to the box**, not Wispr (Jamie: stop Wispr with the
  Delete binding): out of fullscreen if the reader is in it, otherwise put the box
  away. Both verified live. Not verified: Escape while Wispr is actually
  recording (the box's hook is installed after Wispr's, so it should see the key
  first).
- Known pre-existing gap: a reader tab Chrome has suspended is titled "Reading
  Room - Pages" (the page's own "... · ReadingRoom" title is set by its script),
  so `_ReadShowWindow` cannot find it by title and "show typing" says the reader
  isn't open.

---

## 8. NEXT: the writing box inside Chrome (decided 2026-10-15)

Jamie, after using the docked version: two windows pretending to be one is
clunky (F11, Escape, tabs, getting in and out). Decision: build the writing box
INSIDE Chrome as a shared component any local page can use (reader, new tab,
later others), aimed at eventually replacing the AHK box. Keep the docked
version working until then.

**The make-or-break test passed (dragon_mimic, 2026-10-15).** Test page:
`Scripts/viewers/kit/writing-box-test.html`, served by every local server at
`/kit/writing-box-test.html`.
- Plain `<textarea>`: "select hello hello" and "select quiet morning" selected
  exactly those words; "correct hello hello" selected them and opened Dragon's
  Correction Menu with alternatives; "correct river" showed Dragon's numbered
  markers over both occurrences. "cancel" closed the menu cleanly.
- `contenteditable`: "select ..." worked too.
- "insert after / insert before" did nothing in either -- most likely shadowed
  in Chrome by the Caster rule `insert <robot_text>` (Gpt And Opening rules,
  exe chrome, title "-", which matches nearly every Chrome tab). To check.
- Still to confirm by Jamie's real voice (mimic runs the same engine, but her
  mic path and the Dragon Web Extension state are what she'll actually use).

**Shape, so it cannot drift:**
- ONE component in `Scripts/viewers/kit/` (like the hub header): one script tag
  puts a writing box on any local page.
- ONE backend: the page saves nothing itself. Ctrl+Enter POSTs to the page's own
  server (`/kit/...` in local_viewer.py, so every server has it), which calls the
  same save paths as the AHK box (journal.py via `_JournalSaveFromBox`'s
  equivalent, markup.py for `{tags}` etc). Drafts go to the same
  `INIDATA\Drafts\<key>.txt` + `.history.txt` files and format.
- Voice: the page title carries "Writing box" so the existing markup rules see
  it; commands that act on the box (go to line, mark) dispatch on which kind is
  in front.
- Free in a browser textarea: real undo, line breaks (the AHK box's break repair
  exists only because a Win32 Edit mishandles pasted LFs). Needs a JS twin: line
  numbers + go-to-line, the column-width grip.
- Reused as-is from the docked build: the chat-reply wait, `tail_status`,
  `_JournalSaveFromBox`, `ReadingRoomInFront()`.

### 8b. BUILT 2026-10-15 (evening) -- the in-page box is now the default
`reader.typing_surface` = `web` (default) | `window` (the AHK docked box, kept).

| Piece | Where |
|---|---|
| the box itself (textarea, line-number gutter, go-to-line, drafts, Ctrl+Enter / Escape / claimed keys) | `Scripts/viewers/kit/writing-box.js` -- `KitWritingBox.mount(container, cfg)`; any local page can use it |
| drafts, same files + format as the AHK box | `local_viewer.py` `draft_read/save/clear`, `GET/POST /kit/draft?key=` (every local server) |
| reader: config in, submit out | `read_server.py` `POST /api/typing` (JSON body from AHK), `POST /api/typing/submit` (text body -> `MAINFUN ReaderTypingSubmit`), `typing` / `typing_end` in the state; page `syncTyping()` mounts / unmounts in the right half |
| AHK | `ReaderTyping.ahk`: `_RTPlan` (shared by both surfaces), `_RTShowInPage`, `ReaderTypingSubmit`; `LocalViewer_PostBody` |

Behaviour: "show typing" -> the page lays out in the left half and the box opens
on the right with focus in it; the tab title gains "· Writing box" (so tag /
rate / mark voice rules work). `>` `<` turn the page in JS. Escape: out of full
screen first, else put the box away (draft kept); from the page with nothing
else open, Escape also puts it away. Coming back to the tab puts the caret back
in the box. Ctrl+Enter: chat -> paste + wait (box closes with `reason=done`
when the reply is filed); journal -> `_JournalSaveFromBox` (box closes only if
it saved).

Verified live: mount + layout; the AHK box's existing draft opened in the web
box (shared files); Dragon "select quiet river" inside the box; `>` turned the
page without typing; Ctrl+Enter pasted into the right conversation and stopped
(cleared, nothing sent); Escape from the page put it away; journal mode header +
series stem. Not verified: a real send/reply; a journal save from it (would
write an entry); Escape while Wispr is recording; voice go-to-line (the JS
`goToLine` exists, nothing routes "go 70" to it yet).

Next candidates: route "go 70" to the in-page box; put the box on the new tab
page; then retire the AHK docked surface once she's happy.

### 8c. Prefills + the AHK box's look (2026-10-18)
- **Prefills are generic.** `_RTPlan` first tries each function named in
  `_RTPrefillers()` (ReaderTyping.ahk), `fn(st, n) -> plan Map or ""`; the first
  that applies wins, else the plain chat-reply / journal box. A new kind of
  prefill = one function + one name in that list. The box side is generic too:
  `caretQuestion` / `caretToQuestion(n)` in writing-box.js.
- **First prefill: your entry's questions.** On her own entry page in a chat
  view (`...#entry`), when the body has question rounds, "show types" opens the
  box on JUST the rounds (every question with its `A:` blank), caret on the first
  unanswered; **"show types N"** puts it on question N (the number she sees), and
  saying it again with the box open moves the caret. Ctrl+Enter =
  `journal.py questions-section-save`: body = her writing above (untouched) +
  the edited section, under one lock; refuses (exit 3) if her writing above
  changed, (exit 4) if the questions changed since the box opened. Section +
  caret come from `journal_questions.questions_tail` (tests:
  `test_questions_tail.py`). Draft key carries the section fingerprint, so an
  old draft is never offered over newer questions.
- **Looks like the AHK writing box:** the box takes the AHK GUIs' current theme
  from `INIDATA/Theme/gui_theme.json` via `GET /kit/gui-theme` (local_viewer
  `gui_theme()`), so it matches and follows a theme change; Segoe UI at the
  theme's point size.
- Verified live: "show types 3" (mimic) opened both rounds with the caret on 3's
  `A:`; "show types 5" moved it in the open box; an unchanged Ctrl+Enter saved
  and left the entry byte-identical (same fingerprints).

### 8d. Arrow keys between answers (2026-10-18)
- **What she asked for:** in the questions box, Up/Down behave like a normal
  text box inside an answer; Down on an answer's LAST line goes to the start of
  the next answer (just after its `A: `), Up on its FIRST line to the end of the
  previous one. Question lines are reachable only by clicking.
- **A generic box option:** `answerNav: true` in `KitWritingBox.mount`
  (writing-box.js, `answerArrow` / `answerSpans`). The reader turns it on for
  `mode === 'questions'` in `syncTyping()`. The keys hint now says
  "↑↓ move between answers" (`_RTPrefillQuestions`).
- **The browser moves first, the box only corrects.** Whether the caret is on
  an answer's last VISUAL row depends on wrapping, which only the browser knows.
  So the arrow is never blocked: its own move runs, and only if that carried the
  caret OUT of the answer is the caret put where it belongs (`setTimeout 0`).
  Wrapped rows, and blank lines between paragraphs of one answer, behave like
  any text box.
- **What an answer is:** found in the text on every press, so typing or a
  restored draft is never stale. It runs from its `A:` line to the last
  non-blank line before the next question, `## ` round heading or `---` rule.
  A question with no `A:` line within three lines is skipped. Down on the last
  answer and Up on the first stay put rather than reaching a question. Shift/
  Ctrl+arrows and a selection are left alone.
- After a hop the box scrolls only as far as it must (the next question and its
  `A:` line going down, the caret's line going up).
- Verified in the built-in browser with real key presses: first on a test page
  (a wrapped answer moved row by row, a blank line inside an answer was reached,
  the hop crossed a rule and a round heading, the ends stayed put), then on a
  real entry through "show types" (Down x5 went 1 to 6, Up came back, and it
  scrolled to keep the caret on screen). Not yet tried by Jamie with Dragon.
