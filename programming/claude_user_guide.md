# Claude Code — Concepts Guide

A reference for understanding Claude's extension system: what each piece is,
when to use it, and practical use cases relevant to this setup.

---

## Overview — The Four Layers

| Layer | What it is | Who controls it |
|-------|-----------|-----------------|
| **Skills** | Knowledge/instructions that change how Claude works | You install, Claude applies |
| **Hooks** | Shell commands that fire automatically on events | System fires them, always |
| **MCPs** | Connections to external tools and data sources | Claude calls them on demand |
| **Memory** | Persistent facts Claude recalls across conversations | Claude reads/writes automatically |

---

## Skills

### What they are
Skills are instruction files that get loaded into Claude's context and change how
it approaches specific tasks. When a skill is installed and the task matches, Claude
follows the skill's guidance instead of defaulting to generic behavior.

### When to use
- You want Claude to follow specific patterns or best practices in a domain
- You want higher quality output for a recurring type of task
- You're working in a specialized framework or tool with non-obvious conventions

### How they activate
Most skills activate automatically when the task matches their description. Some
are triggered by slash commands (e.g. `/commit`).

### Installed skills (this setup)
| Skill | When it fires | What it does |
|-------|--------------|--------------|
| `frontend-design` | Building web UIs, components, pages | Pushes for polished, non-generic design |
| `microsoft-docs` | Azure, M365, Windows, .NET questions | Queries official Microsoft docs |
| `streamdeck` | Any Stream Deck button task | Manages SD button editing workflow |
| `update-config` | Configuring Claude Code settings/hooks | Edits settings.json correctly |
| `keybindings-help` | Customizing Claude Code keybindings | Edits keybindings.json |
| `simplify` | After making code changes | Reviews for quality and efficiency |
| `skills-discovery` | Finding/installing new skills | Searches claude-plugins.dev registry |
| `claude-api` | Building apps with Claude/Anthropic SDK | API usage patterns and best practices |
| `loop` | Running something on a recurring interval | Sets up polling or repeat tasks |
| `schedule` | Setting up cron-style scheduled agents | Creates remote scheduled triggers |

### Use cases
- Atlantic Logistics web work → `frontend-design` raises UI quality automatically
- M365/McLeod questions → `microsoft-docs` pulls current official docs
- Adding Stream Deck buttons → `streamdeck` handles the whole workflow

---

## Hooks

### What they are
Hooks are shell commands wired to events in Claude's lifecycle. They fire
automatically — Claude doesn't decide when to run them, the harness does.
They run outside Claude's context.

### When to use
- You want something to happen on *every* occurrence of an event, no exceptions
- You want a safety net that doesn't depend on Claude remembering to do something
- You want to block or validate something before Claude proceeds
- You need to log, notify, or trigger external systems automatically

### Hook events
| Event | When it fires |
|-------|--------------|
| `PreToolUse` | Before Claude uses any tool — can block it |
| `PostToolUse` | After a tool succeeds |
| `SessionStart` | When a new Claude session begins |
| `Stop` | When Claude finishes responding |
| `PreCompact` | Before conversation compaction |
| `UserPromptSubmit` | When you submit a message |
| `Notification` | On Claude notifications |

### Hook types
| Type | What it does |
|------|-------------|
| `command` | Runs a shell command |
| `prompt` | Asks a small LLM to evaluate something |
| `agent` | Spins up a mini agent with tools to verify something |

### Use cases for this setup
- **Auto-format Python** after every Caster rule edit (black/autopep8)
- **Log all Bash commands** Claude runs to a file for review
- **Run AHK syntax check** after editing any .ahk file
- **Block edits** to protected files (e.g. MAINFUNCTIONS.ahk) without confirmation
- **Play a sound / send notification** when Claude finishes a long task (Stop hook)
- **Auto-enable new Caster rules** after Claude creates a rule file

### Currently configured
None. Blank slate — good opportunity to add the Python formatter.

---

## MCPs (Model Context Protocol)

### What they are
MCPs are connections to external systems that give Claude the ability to read
data and take actions in real tools — directly in the conversation, without
copy-pasting. Claude calls MCP tools on demand, like any other tool.

### When to use
- You want Claude to interact with an external system without manual data transfer
- You have a recurring need to query or act on data from a specific source
- You want to build a custom integration between Claude and your own tools

### Installed MCPs (this setup)
| MCP | What it connects to | What Claude can do |
|-----|--------------------|--------------------|
| Gmail | Your Gmail accounts | Read, search, draft emails |
| Google Calendar | Your calendars | List, create, update events |
| Playwright | A real browser | Navigate, click, fill forms, screenshot |

### Building a custom MCP
A custom MCP is a Python (or Node.js) program that exposes tools Claude can call.
It connects to Claude Code via settings.json. Weekend-scale project for something useful.

**Requires:** API access on the external system's side first.

### Use cases — existing
- Search Gmail for Atlantic Logistics emails from Rob
- Check calendar before scheduling work sessions
- Test a web UI by having Claude actually click through it in a browser

### Use cases — custom MCPs worth building
- **McLeod TMS MCP** — query load data, customer profiles, order status directly
  in conversation. Would replace manual copy-paste of TMS data.
  *Requires: McLeod API access (the critical dependency for the load email project)*
- **Caster rule manager MCP** — enable/disable rules, check what's loaded,
  list all voice commands without digging through files
- **AHK function index MCP** — query available AHK functions by name or domain,
  so Claude can check what exists before writing new ones

---

## Memory

### What it is
A file-based persistent memory system at `~/.claude/projects/.../memory/`.
Claude reads these files at the start of conversations and writes to them when
it learns something worth keeping. Survives across sessions.

### When Claude saves memory
- Learning something about you or your preferences
- Feedback you give (corrections, confirmations of approach)
- Project context that isn't obvious from the code
- Pointers to where information lives in external systems

### Types of memory
| Type | What goes in it |
|------|----------------|
| `user` | Who you are, your role, preferences, knowledge level |
| `feedback` | How you want Claude to work — corrections and confirmations |
| `project` | Ongoing work, goals, decisions, deadlines |
| `reference` | Where to find things: files, tools, external systems |

### What does NOT go in memory
- Code patterns (read the code instead)
- Git history (use git log)
- Anything already in CLAUDE.md files
- Ephemeral task details from the current session

### Use cases
- Claude remembers you prefer AHK-first design without being told every time
- Claude knows Deck A is the default for Stream Deck tasks
- Claude knows where your Atlantic Logistics files live
- Claude knows your email tone for professional writing

---

## Getting New Skills

### Finding skills
Skills come from the community registry at claude-plugins.dev. You can browse it
in a browser, or ask Claude to search it:

> "Search for a skill that does X"
> "Find skills for [technology/topic]"
> "Are there any good skills for [task]?"

Claude will show you results with star counts and install counts. Prefer skills
with **high installs** over high stars — installs reflect real usage.

### Installing
Just tell Claude which one you want:
> "Install that one" / "Install @namespace/skill-name"

Claude will run the installer automatically. Skills install globally by default
(available in all projects).

### Uninstalling
> "Uninstall the [skill name] skill"

Claude runs: `npx skills-installer uninstall @namespace/skill-name`

### What to look for
- **Official Anthropic skills** (`@anthropics/...`) — highest trust, well-maintained
- **High install count** — means people are actually using it and it works
- **Clear, specific description** — vague descriptions usually mean vague behavior
- **Avoid 0-install skills** — untested, may give bad guidance

### When NOT to install a skill
- Your setup already has the relevant context in CLAUDE.md or memory
- The skill covers a technology you don't use (e.g. AHK v2 skill when you're on v1)
- The skill would conflict with established patterns in your project
- Stars are high but installs are very low (inflated/fake stars)

### Keeping skills updated
Skills don't auto-update. If a skill seems stale or broken, uninstall and
reinstall to get the latest version.

---

## Slash Commands (Built-in)

These are built into Claude Code — no skill needed.

| Command | What it does |
|---------|-------------|
| `/clear` | Clear conversation context and start fresh |
| `/compact` | Summarize and compress the conversation to save context space |
| `/memory` | View and manage Claude's memory files |
| `/hooks` | Open the hooks configuration UI |
| `/cost` | Show token usage and cost for the current session |
| `/rename` | Rename the current conversation |
| `/help` | List all available commands |
| `/loop 5m` | Run a prompt repeatedly on an interval |
| `/schedule` | Create a scheduled recurring agent |

### When to use /compact vs /clear
- `/compact` — you want to keep working on the same task but Claude is running
  low on context. Summarizes what happened so far and continues.
- `/clear` — you're done with the current task and starting something new.
  Full reset, no memory of the previous conversation.

---

## CLAUDE.md Files

### What they are
Markdown files that Claude reads automatically at the start of every session.
They provide persistent instructions that don't need to go in memory.

### Locations and scope
| File | Scope |
|------|-------|
| `~/.claude/CLAUDE.md` | Global — applies to all projects |
| `<project>/.claude/CLAUDE.md` or `<project>/CLAUDE.md` | Project-specific |

### What goes in CLAUDE.md vs memory
| CLAUDE.md | Memory |
|-----------|--------|
| Stable rules and architecture | Things Claude learned mid-conversation |
| Project structure and conventions | Your preferences and feedback |
| How to call AHK functions | Ongoing project context |
| Team/project-wide instructions | Pointers to external resources |

Your global CLAUDE.md currently has your name/pronouns. Your project CLAUDE.md
(in the Caster rules repo) has the full AHK-first architecture, rule file structure,
mainfun bridge patterns, Stream Deck editing workflow, etc.

---

## Permission Modes

Claude Code has several modes controlling what it can do without asking:

| Mode | Behavior |
|------|---------|
| `default` | Asks for confirmation on most tool use |
| `acceptEdits` | Auto-approves file edits, asks for Bash |
| `bypassPermissions` | Auto-approves everything — your current setting |
| `plan` | Claude proposes a plan first, then executes on approval |

Your setup uses `bypassPermissions` — Claude runs everything without prompting.
This is efficient but means hooks are your main safety net if you want guardrails.

**Plan mode** is useful for large or risky tasks — tell Claude to work in plan
mode and it will lay out exactly what it's going to do before touching anything.

---

## Decision Guide — Which to Use?

| Situation | Use |
|-----------|-----|
| You want Claude to be smarter about a domain | Skill |
| You want something to happen automatically every time | Hook |
| You want Claude to read/write to an external system | MCP |
| You want Claude to remember something between sessions | Memory |
| You want to block Claude from doing something dangerous | Hook (PreToolUse) |
| You want Claude to notify you when done | Hook (Stop) |
| You want to query live data without copy-pasting | MCP |
| You want better code/UI quality automatically | Skill |
| You want a recurring automated task | Schedule skill / Cron |

---

## Quick Reference — This Setup's Entry Points

| Task | How to invoke |
|------|--------------|
| Add a Stream Deck button | Just describe it — `streamdeck` skill activates |
| Find/install a new skill | "Search for a skill that does X" |
| Add a hook | "Add a hook that does X" — `update-config` skill handles it |
| Search Gmail | Claude uses Gmail MCP directly |
| Remember something | "Remember that..." — Claude writes to memory |
| Schedule a recurring task | `/schedule` |
| Run something repeatedly | `/loop 5m` |
