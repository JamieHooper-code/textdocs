---
tags: [reference, qmd, search, gpu, setup, done]
created: 2026-05-21
updated: 2026-05-28
status: done
---

# QMD Warm GPU Search — Setup & Reference

QMD is the local markdown search engine (BM25 + vector + LLM rerank/expansion over Jamie's
docs, skills, and programming notes). This file records how the **GPU-accelerated, always-warm**
setup works, the real before/after numbers, and how to maintain it.

Supersedes the old `INSTALL_RTX_2070` checklist — the card actually installed was an **RTX 5060 Ti
(16 GB)**, not the 2070, and the whole thing is now done.

Related: [[pc-specs-and-toolchain]], [[reference_computer_layout]] (memory), [[claude-user-guide]].

---

## TL;DR — current state (2026-05-28)

- GPU: **RTX 5060 Ti, 16 GB**, driver CUDA 13.1. QMD offloads to it via **Vulkan**.
- A **warm MCP daemon** runs at logon (`http://localhost:8181/mcp`); Claude Code's qmd plugin
  connects to it. Models stay resident in VRAM, so queries are interactive (~3-4 s) instead of
  the ~50 s cold tax.
- CUDA was **not** pursued — see "Why Vulkan, not CUDA" below. Vulkan-warm is fast enough.

---

## Measured benchmarks (RTX 5060 Ti, Vulkan)

`qmd search` (BM25, no models) is unaffected by any of this — always ~0.5 s.

Full pipeline (`query` = expand + embed + rerank), measured in one process:

| Stage | Cold (fresh process) | Warm (models resident) |
|---|---|---|
| expand (1.7B) | 48,500 ms | ~3,000 ms |
| embed (0.3B) | 1,215 ms | 8 ms |
| rerank (0.6B) | 1,897 ms | ~100 ms |
| **total** | **~51 s** | **~3-4 s** |

The bottleneck was never the GPU — it was reloading three GGUF models (embed +
rerank + 1.7B query-expander) and recompiling Vulkan shaders on **every** CLI invocation.
Keep the models resident (the daemon) and it collapses to ~3-4 s.

Models in use: `embeddinggemma-300M` (embed), `Qwen3-Reranker-0.6B-Q8_0` (rerank),
`qmd-query-expansion-1.7B` (expansion).

---

## The four root-cause fixes that made this work

All four were real bugs/config gaps on native Windows, fixed cleanly (not worked around):

### 1. Broken npm launchers (`qmd` failed from PowerShell/cmd)
`@tobilu/qmd`'s `bin/qmd` starts with `#!/bin/sh`, so npm's generated `qmd.cmd` / `qmd.ps1`
tried to exec `/bin/sh` — which doesn't exist for native Windows shells (worked only from Git
Bash). **Fix:** rewrote both launchers to call node directly:
`node "...\node_modules\@tobilu\qmd\dist\cli\qmd.js" %*`
Files: `C:\Users\jamie\AppData\Roaming\npm\qmd.cmd` and `qmd.ps1`.

### 2. `HOME` unset → qmd used an empty `/tmp` index
`dist/store.js` ships its own `homedir()` = `process.env.HOME || "/tmp"`. Git Bash sets `HOME`;
native Windows (PowerShell, cmd, **and a logon-autostarted service**) does not, so qmd silently
resolved its index/config/model-cache under `/tmp` (empty). **Fix:** set a user env var
`HOME = C:\Users\jamie` (= USERPROFILE, the value every other tool already uses). The autostart
VBS also sets it explicitly as insurance.

### 3. Plugin MCP server was failing / cold
The qmd plugin shipped a **stdio** MCP server (`bash -c "qmd mcp"`) that showed
"✗ Failed to connect" and would have been cold-per-session anyway. **Fix:** repointed it at the
warm HTTP daemon. Edit in:
`C:\Users\jamie\.claude\plugins\cache\qmd\qmd\0.1.0\.claude-plugin\marketplace.json`
```json
"mcpServers": { "qmd": { "type": "http", "url": "http://localhost:8181/mcp" } }
```
⚠ **Maintenance:** if you ever `claude plugin update qmd`, this edit reverts to the cold stdio
server. Re-apply the HTTP block above. (`claude mcp` has no per-server disable, so reusing the
plugin's slot is the cleanest single-entry option.)

### 4. Always-warm daemon at logon
Startup-folder VBS launches the **foreground** HTTP server hidden (not `--daemon`: node's
detached spawn pops a console window on Windows). File:
`C:\Users\jamie\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\QMD-Daemon-AutoStart.vbs`
Models stay loaded because qmd's `disposeModelsOnInactivity` defaults to **false** — the 5-min
idle timer only frees cheap-to-rebuild contexts, not the models.

---

## Operating it

- **Is it running?** `netstat -ano | findstr :8181` (LISTENING = up). Note: `qmd status` will
  *not* show an "MCP: running" line, because the foreground launch writes no PID file — that's
  expected, not a fault.
- **Restart the daemon** (rare — only if it hangs; reboot re-runs it automatically):
  ```powershell
  Get-NetTCPConnection -LocalPort 8181 -State Listen | Select-Object -Expand OwningProcess | Stop-Process -Force
  wscript "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\QMD-Daemon-AutoStart.vbs"
  ```
- **First query after a reboot** pays the ~50 s cold model load once; everything after is warm
  for the rest of the session.
- After changing the plugin manifest or adding the daemon, **reload the VS Code window** so
  Claude Code picks up the qmd MCP tools for the current session.

---

## Why Vulkan, not CUDA

node-llama-cpp 3.18.1's prebuilt CUDA binary is *"not compatible with the current system"* — it
has no kernels for the 5060 Ti's **Blackwell** arch (sm_120). It then tries to build llama.cpp
from source and fails (no Visual Studio C++ toolchain installed). Vulkan loads fine and, once
warm, expand is already ~3 s; CUDA + tensor cores might shave that to ~1.5 s but costs a
multi-GB VS Build Tools + CUDA Toolkit install and a from-source compile. Not worth it for this
workload. If ever revisited: install VS Build Tools ("Desktop development with C++") + CUDA
Toolkit 12.8+, then `npm rebuild` node-llama-cpp.

---

## The `functions` collection — semantic search over AHK functions + voice commands

Added 2026-05-28. `ahk_search.py` ranks by literal substring overlap, which is noisy and
misses semantically-relevant matches (it ranked a URL-getter #1 for "pause youtube video").
The `functions` collection puts the same metadata into qmd so `query` ranks it by *meaning*
(e.g. `SnapMoveMonitor` at 93% for "switch to second monitor").

**How it works:**
- `gen_qmd_function_index.py` (in `AutoHotkey\Scripts\codebase_tools\`) turns the
  `.ahk.meta.json` sidecars + Caster rdescripts into **one markdown file per function/command**
  (H1 = name, body = summary/aliases/tags/args), written to
  `C:\Users\jamie\AppData\Local\qmd-function-index\` (non-synced; derived data). One file per
  function so qmd indexes each as its own chunk — file-grouped markdown gets merged into coarse
  multi-function chunks that blunt ranking. ~1900 docs.
- Registered as qmd collection `functions` (`qmd collection add ... --name functions`).
- The verify stop hook (`caster_ahk_verify.py` → `check_function_index`) regenerates the
  markdown + re-embeds when `.ahk` / sidecars / rule `.py` files change. The generator is
  idempotent (only rewrites changed files; prunes orphans) so re-embeds stay incremental.

**Using it:** query the warm MCP `query` tool with `collections: ["functions"]` for "is there a
function/command that does X?". CLI equivalent `qmd query -c functions "..."` works but is cold
(~50s). `ahk_search.py` stays for exact-name lookups + reading precise args/aliases (instant,
offline). Full tiering: the `feedback_search_tool_selection` memory in the caster/rules project.

**Manual rebuild** (if it ever drifts): `python "C:\Users\jamie\Desktop\Important\AutoHotkey\Scripts\codebase_tools\gen_qmd_function_index.py"` then `qmd update && qmd embed`.
