---
tags: [reference, hardware, setup, pc, toolchain]
created: 2026-05-21
---

# PC Specs & Toolchain

Snapshot of the current machine (Acer Nitro 60) and the full software/hardware stack running on it. Replaces the migration-era documents now that the Dell → Acer transition is done.

Related: [[reference_computer_layout]] (memory), [[HOTKEYS]] (single source of truth for all hotkeys), [[migrate-dell-to-acer]] (historical — kept for reference, not active).

---

## 1. Machine — Acer Nitro 60 N60-640-UR26

| Component | Spec |
|---|---|
| Motherboard | Gigabyte B760M (mATX, Intel B760) — standard, not proprietary Acer |
| CPU | Intel Core i7-14700F (20-core, 8P + 12E) |
| RAM | 32 GB DDR5, 4 DIMM slots (2 free) |
| Primary storage | 2 TB PCIe 4.0 NVMe SSD (slot 1) |
| GPU | NVIDIA RTX 5060 Ti 16 GB GDDR7 (NVENC av1/hevc/h264) |
| PSU | Thermaltake 850W modular ATX |
| M.2 slots | 2–3 PCIe 4.0 (≥2 confirmed) |
| SATA bays | 2× 3.5", 2× 2.5" (brackets sold separately by Acer) |
| Case | mATX mid-tower, 15.9 × 14.9 × 8.5" |
| Warranty | 1-year refurb (eBay seller) |

### Storage expansion notes
- 3.5" HDD install requires an Acer bracket kit (or generic 3.5" cage) — chassis ships with the mounting holes but no caddy.
- Spare SATA power leads in the Thermaltake PSU box if not already plugged in.
- 2.5" SATA bays accept SSDs or 2.5" HDDs without a bracket.

---

## 2. Voice stack — Dragon → Natlink → Caster → AHK

The macro nerve center. Spoken phrase fires a keystroke or shells out to `MAINFUN.bat` which dispatches to an AutoHotkey function.

| Layer | What it does | Version | Path |
|---|---|---|---|
| Dragon NaturallySpeaking | Speech recognition | 16.10.200.044 | `C:\ProgramData\Nuance\NaturallySpeaking16\` |
| Natlink | Python bridge to Dragon | 5.5.7 | `C:\Users\jamie\.natlink\natlink.ini` |
| Python (Natlink) | Required by Natlink | 3.10.11 (32-bit) | `py -3.10-32` |
| Caster | Voice command framework on Dragonfly | 1.6.16 | Rules: `C:\Users\jamie\AppData\Local\caster\rules\` |
| Caster settings | Enabled rules + config | — | `C:\Users\jamie\AppData\Local\caster\settings\` |
| MAINFUN bridge | Batch dispatcher Caster → AHK | — | `MAINFUN.bat` (on PATH) → calls `MAINFUNCTIONS.ahk` |

**Skill:** `caster-voice` — load it when working on rule files.
**Docs:** `C:\Users\jamie\Desktop\Important\AutoHotkey\docs\caster-ahk-bridge.md` — every `mainfun_action` variant.
**Logs:** `C:\Users\jamie\AppData\Local\caster\log.txt` (caster), `C:\ProgramData\Nuance\NaturallySpeaking16\logs\jamie\Dragon.log` (Dragon).

---

## 3. AutoHotkey

PascalCase function names. Helpers split by domain. `MAINFUNCTIONS.ahk` is the include hub `MAINFUN.bat` calls.

| What | Path |
|---|---|
| AHK v2 binary | `C:\Users\jamie\AppData\Local\Programs\AutoHotkey\v2\AutoHotkey64.exe` |
| Root | `C:\Users\jamie\Desktop\Important\AutoHotkey\` |
| Entry point | `MAINFUNCTIONS.ahk` (includes all helpers) |
| Helpers | `Helpers\` (one file per domain — VSCodeFunctions, AppHandling, etc.) |
| Helper map | `Helpers\_DIRECTORY_DOCUMENTATION.ahk` |
| Always-on macros | `GENERALMACROSALWAYSON.ahk` |
| INI config | `INIDATA\` |
| Docs | `docs\` (README, bridge, debugging-guide, design-decisions, …) |

**Skill:** `ahk-functions`.
**Search:** `ahk_search.py "<term>"` for "does this function/command exist" (0.1s, structured).

---

## 4. Stream Deck

Two physical Stream Deck XL units (8×4 = 32 buttons each, model 20GAT9902). Both AHK-dispatched via `MAINFUN.bat` button settings.

| Deck | Serial |
|---|---|
| Deck A (~27 profiles) | `A00NA52731YDM1` |
| Deck B (~23 profiles) | `A00NA53332QT2R` |

| What | Path |
|---|---|
| Active profiles dir | `C:\Users\jamie\AppData\Roaming\Elgato\StreamDeck\ProfilesV3\` |
| Old (ignore) | `…\ProfilesV2\` |
| Helper script | `streamdeck.py` (in skill `references/`) |

**Skill:** `streamdeck` — handles add/edit/move/copy across both decks.

---

## 5. Keychron Q0 Max numpad

Standalone 5-page macropad. Runs as its own persistent AHK process (NOT included from MAINFUNCTIONS — would cause hotkey conflicts).

| What | Path |
|---|---|
| Project README + protocol | `C:\Users\jamie\Desktop\Important\AutoHotkey\Scripts\KeychronQ0Max\README.md` |
| Python CLI (VIA-over-HID, cable only) | `Scripts\KeychronQ0Max\q0max.py` |
| Behavior file (hand-edited) | `Scripts\KeychronQ0Max\q0max_pages.ini` |
| Page state | `Scripts\KeychronQ0Max\q0max_page.txt` |
| AHK entry | `Q0MaxNumpad.ahk` |
| AHK dispatcher | `Helpers\Q0MaxDispatcher.ahk` |
| Hotkey ranges | Alt+F13–F24, Ctrl+Alt+F13–F17, Shift+Alt+F13–F16, Ctrl+Alt+F20–F24 (page switchers) |

**Wireless caveat:** LED control / keymap edits work on **cable only** — the 2.4GHz dongle exposes a stub interface that times out. Regular hotkey events still pass through wireless. See [[reference_q0max_numpad]].

---

## 6. Arduino Leonardo (real-HID fallback)

ATmega32u4 board for cases where Windows-injected input gets filtered (`LLKHF_INJECTED` flag). Currently wired up for `WarpToGaze` (Tobii integration).

| What | Path / detail |
|---|---|
| Sketch | `C:\Users\jamie\Desktop\Important\AutoHotkey\Arduino\WarpToGaze\WarpToGaze.ino` |
| COM port (current PC) | COM5 (COM3 = bootloader) |
| AHK bridge | `Helpers\ArduinoFunctions.ahk` — `SendArduinoCommand(cmd)` |
| Arduino IDE | 2.3.8 (winget: `ArduinoSA.IDE.stable`) |
| Arduino CLI | 1.4.1 at `C:\Program Files\Arduino CLI\arduino-cli.exe` |

**Rule:** before reaching for Arduino, try an elevated AHK / Scheduled Task first.

---

## 7. Window/monitor management

| What | Path |
|---|---|
| DisplayFusion | `C:\Program Files\DisplayFusion\DisplayFusion.exe` |
| DF CLI | `C:\Program Files\DisplayFusion\DisplayFusionCommand.exe` |
| DF config (hotkeys, all) | Registry: `HKCU\Software\Binary Fortress Software\DisplayFusion\` |
| Snap hotkeys | `Win+Ctrl+Alt+1`–`8` |

---

## 8. AI / Claude tooling

| What | Path |
|---|---|
| Claude Code global settings | `C:\Users\jamie\.claude\` |
| Skills | `C:\Users\jamie\.claude\skills\` |
| Memory (auto-loaded) | `C:\Users\jamie\.claude\projects\c--Users-jamie-AppData-Local-caster\memory\` |
| Helper scripts (on PATH) | `C:\Users\jamie\.claude\scripts\` |
| Subprocess wrapper | `llm_gateway.py` (chokepoint for local scripts calling Claude) |

**QMD** (markdown search, installed user-scope): index at `C:\Users\jamie\.cache\qmd\index.sqlite`. Commands: `qmd search` (BM25, 0.5s), `qmd vsearch` (semantic, 11s).

---

## 9. Obsidian vault

Resource capture + notes + this doc.

| What | Path |
|---|---|
| Vault root | `C:\Users\jamie\Desktop\Important\ObsidianVault\` |
| Obsidian.exe | `C:\Users\jamie\AppData\Local\Programs\Obsidian\Obsidian.exe` |
| Vault registry | `C:\Users\jamie\AppData\Roaming\obsidian\obsidian.json` |
| TEXTDOCS (this doc lives here) | `…\ObsidianVault\TEXTDOCS\` |
| Programming notes | `…\TEXTDOCS\programming\` |

---

## 10. Email accounts

Decision rule: Atlantic recipient → Atlantic Graph. Other client/work → JamieHooperCode. Personal → jamieeehooper.

| Account | Slot | Tooling |
|---|---|---|
| jamieeehooper@gmail.com (personal) | u/1 | Gmail MCP |
| JamieHooperCode@gmail.com (client work) | u/3 | Gmail MCP |
| jamie.hooper@shipatlantic.com (Atlantic) | — | Microsoft Graph via `atlantic_graph.py` |
| natehoop@gmail.com | u/0 | Gmail MCP (read-only, rarely drafted from) |
| nathooper96@gmail.com | u/2 | Gmail MCP (read-only, rarely drafted from) |

Atlantic Graph location: `…\ObsidianVault\TEXTDOCS\programming\Atlantic Logistics\atlantic_graph\atlantic_graph.py`.

---

## 11. Other peripherals

- **Tobii Eye Tracker 5** — used by `WarpToGaze` and gaze-driven cursor work.
- **Foot pedals** — wired into AHK (`Helpers\Footpedals.ahk`).
- **Microphone** — Dragon-tuned profile (carried over from Dell clone).
- **Monitors** — multi-monitor with DisplayFusion managing snap zones.

---

## 12. External storage — E: drive

Large media + binaries live on E: per [[feedback_external_drive_storage]].

| What | Path |
|---|---|
| Jellyfin install root | `E:\Jellyfin\` |
| Jellyfin binary | `E:\Jellyfin\app\jellyfin\jellyfin.exe` |
| Jellyfin data (DB, metadata) | `E:\Jellyfin\data\` |
| Jellyfin admin creds + API token | `E:\Jellyfin\admin_creds.json` |
| Jellyfin web UI | `http://127.0.0.1:8096` |
| Media library — Movies | `E:\Media\Movies\` |
| Media library — TV | `E:\Media\TV\` |
| Downloads (qBittorrent + browser) | `E:\Downloads\` |
| Backups | Scripts/keys, Atlantic certs, Dragon profile (see migration doc Phase 1 inventory) |

**Jellyfin auto-starts** via VBS in `Startup\Jellyfin-AutoStart.vbs` at user logon.

---

## 13. Dev toolchain (versions worth pinning)

| Tool | Version |
|---|---|
| Python (default) | 3.10.x (`py --version`) |
| Python (Natlink, 32-bit) | 3.10.11 (`py -3.10-32`) |
| AutoHotkey | v2 |
| VS Code | latest (synced settings) |
| Claude Code CLI | `claude --version` |
| Git | system git |
| Arduino IDE | 2.3.8 |
| Arduino CLI | 1.4.1 |
