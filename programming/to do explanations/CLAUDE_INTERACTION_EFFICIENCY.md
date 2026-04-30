---
tags: [claude, voice, streamdeck, accessibility, project]
---

# Claude Interaction Efficiency — minimize keystrokes & voice load

Project to drive Claude Code interactions toward Stream Deck + numpad input,
minimizing keystrokes and voice load (elbow disability + voice fatigue). Work
the lowest-friction surfaces first; backlog the bigger redesigns.

## Current state — what's built

### Numpad reply shorthand
- Spec: `1.-2.+3.4.*7` style. Bare `N` = yes, `-N` = no, `+N` = discuss,
  `*N` = comment follows, `/N` = skip. Special codes: 0/50/69/77/88/99.
- Primer file: [[ClaudePrompts/numbers.txt]]
  (`AutoHotkey/INIDATA/ClaudePrompts/numbers.txt`).
- Memory rule: [[feedback_numpad_reply_format]] — Claude parses these on
  sight and formats multi-question turns as a flat global-numbered list
  (1-99, no letters for sub-questions, footer with full legend).
- Voice: `send numbers` pastes the primer in fresh sessions.

### Send-prompt mechanics
- `SendClaudePrompt(name)` in
  `AutoHotkey/Helpers/ClaudeFunctions.ahk` pastes named .txt prompts.
- `AddSendPrompt` (in `VoiceConfigManager.ahk`) is the proper add path —
  creates the .txt with frontmatter, updates `send_prompts.json`, offers a
  Deck B button.
- Per-prompt frontmatter: every file begins with
  ```
  ---
  auto_send: true
  ---
  ```
  - `auto_send: true` → paste + Enter (submit).
  - `auto_send: false` → paste + Shift+Enter (paragraph break, no submit;
    cursor stays in the box for follow-up dictation).
- Dictation Box guard: when `natspeak.exe` is foreground, paste only — no
  Enter, no Transfer. Jamie reviews and submits manually.
- Existing prompts: `audit, clipboard, debug, document, goahead, handoff,
  learn_testing, memory, next, numbers, plan, question`.

### Stream Deck wiring
- Deck B `GECK2 CLAUDE` page hosts most send-prompt buttons (tap = send,
  hold = edit). Deck A has a subset.
- Wiring is symmetric: every SD prompt button maps to a `send_prompts.json`
  entry, so every button has a voice command equivalent.

## Open TODOs

### Add new send prompts (4)
Create `tighter`, `more`, `why`, `cite`. Suggested contents:
- `tighter` → "Tighter. Fewer words, less hedging. Cut anything that isn't load-bearing."
- `more` → "Expand on your last point with more detail."
- `why` → "Why? Walk me through the reasoning and tradeoffs you considered."
- `cite` → "Cite the specific files and line numbers for what you just said."

Decision pending: voice-add (proper, slow) vs direct write (fast, no SD button offer).

### Reply-atom Stream Deck buttons
Single-press shortcuts for the most-typed numpad replies:
- `+1.+2` ("discuss 1 and 2")
- `-all` (reject everything, redo)
- `*` (switch to prose)
- `88` (tighten)
- `99` (restart approach)

These would be Deck B buttons that paste the literal string into the chat
input + Enter. Could be SendClaudePrompt entries with `auto_send: true`.

### Suffix vs prefix in numpad shorthand — RESOLVED (suffix)
Spec uses SUFFIX (`4-`, `7+`) — confirmed 2026-04-28. Updated in
`numbers.txt`, `feedback_numpad_reply_format.md`, and the legend footer
template.

### Audit decks for orphan voice-less buttons
Scan Deck A and Deck B manifests for buttons with no spoken-command
equivalent. Produce a list so Jamie can decide which deserve voice
coverage. Stream Deck prompt buttons are already symmetric; the gap is
elsewhere on the decks.

## Backlog — bigger redesigns

### "Spit Claude back" — reverse direction
Take text from Claude's output and re-prompt with it + a stock
instruction. Open design questions before building:
- **Source of text:** current VS Code selection / last assistant message
  programmatically (how would AHK identify it?) / `claude.txt` slot
  (paste-first manual flow).
- **Auto-submit vs paste-for-review.**
- **Stock instructions:** `summarize`, `actions` (extract action items),
  `tighten`, `cite`, `clarify`. Suggest a new directory parallel to
  `ClaudePrompts/` — `ClaudeBackPrompts/` — and a parallel
  `back_prompts.json` voice mapping.
- **Voice phrasing:** `spit back <choice>` vs folding into the existing
  `send` command.
- **Stream Deck:** symmetric Deck B page or live alongside the
  send-prompt buttons.

### Voice → numpad bridge
Caster rule that translates spoken `"yes one no two discuss three"` into
the numpad string. Fallback for when fingers are the bottleneck but
voice has range.

### Question-recap button
Voice/SD trigger for "List your open questions as a flat numbered list
with a count, no prose." Useful when Claude drifts back into nested
numbering or Jamie has lost the thread.

### QMK numpad layer for `?`
If Jamie wants `?` in the symbol set (currently absent because numpad
has no `?`), program a QMK layer on the Q0 Max so `+`-held sends `?`.
Low priority — `+` for "discuss" is intuitive enough.

## Pointers
- Memory rule: [[feedback_numpad_reply_format]]
- Primer file: `AutoHotkey/INIDATA/ClaudePrompts/numbers.txt`
- AHK helpers: `AutoHotkey/Helpers/ClaudeFunctions.ahk`
- AHK add flow: `AutoHotkey/Helpers/VoiceConfigManager.ahk` → `AddSendPrompt`
- Caster rule: `caster/rules/claude_commands.py`
- JSON config: `AutoHotkey/INIDATA/VoiceChoices/send_prompts.json`
