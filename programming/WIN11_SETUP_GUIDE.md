 Windows 11 Fresh Install — Setup Guide
> Organized by priority: get accessible first, then everything else.
> Items marked [Claude can help] are ones I can assist with directly.

## Note for Claude

This guide is a complete inventory, not a to-do list — not everything here will be installed. Work through it collaboratively: handle technical steps (file copies, config edits, path fixes) as they come up, but check in before installing apps since Jamie wants to keep the new machine uncluttered. When in doubt, ask.

---

## PHASE 0 — Before You Start

> Note: The old SSD is staying — nothing is being destroyed. This is a new SSD install. Everything on the old drive remains recoverable if needed, just inconvenient.

- [ ] **Push everything** — do a final commit and push on all repos before switching over
- [ ] **Anything else?** — take a moment to think if there's anything not in GitHub/OneDrive/E: that you'd miss

---

## PHASE 1 — First Hour: Get Input Working

Goal: Be able to control the computer without constant pain.

### 1.1 Windows + Microsoft Account
- [ ] Install Windows 11
- [ ] Sign in to Microsoft account → OneDrive starts syncing in background
- [ ] Let OneDrive sync before doing anything that depends on it

### 1.2 Chrome + Google Account
- [ ] Install Chrome
- [ ] Sign in to Google account → bookmarks, extensions restore automatically
- [ ] **Vimium** should restore from Chrome sync; verify it loaded
- [ ] Confirm Vimium keybinds are correct (they sync with Google account)

### 1.3 Claude

- [ ] Open claude.ai in Chrome — usable immediately, no install needed
- [ ] **Claude desktop app** — download and install from claude.ai/download
  - Lets you use Claude outside of the browser, with file access and better OS integration
- [ ] **Claude Code** (CLI) — install via terminal once Node.js is available:
  ```
  npm install -g @anthropic-ai/claude-code
  ```
  - Can also wait until Phase 4 when dev tools are set up
- [ ] **Claude Code VS Code extension** — install from the VS Code marketplace once VS Code is up (Phase 4)

---

### 1.4 Touchscreen

Two monitors — only one is a touchscreen. Windows 11 sometimes assigns touch input to the wrong display, and sometimes fails to recognize the touchscreen at all on fresh installs.

**Step 1: Check if Windows detects it**
- Open Device Manager (`Win + X` → Device Manager)
- Look under **Human Interface Devices** for entries like "HID-compliant touch screen"
- If it's there with no warning icon, touch is likely working — test it before going further

**Step 2: If touch isn't working**
- In Device Manager, right-click the touchscreen entry → **Update driver** → Search automatically
- If no entry exists at all: Action → **Scan for hardware changes**
- If there's a yellow warning icon: right-click → **Uninstall device** → restart — Windows will reinstall the driver on reboot

**Step 3: If driver install fails or touch is still broken**
- Go to the manufacturer's website for your touchscreen/monitor and download the driver manually
- Alternatively: Windows Update sometimes has the driver — Settings → Windows Update → Advanced options → Optional updates

**Step 4: Calibrate if needed**
- Search "Calibrate the screen for pen or touch input" in Start
- Run the touch calibration wizard

**Step 5: If touch works but registers on the wrong monitor**
- Search "Tablet PC Settings" → Display tab → **Setup** → follow the prompt to tap the correct screen
- Then Calibrate → Touch to fine-tune accuracy

> [Claude can help] If Device Manager shows the device but touch still isn't responding, share what the device entry says and I can help diagnose.

---

### 1.5 Dragon — UPGRADE TO DRAGON PROFESSIONAL v16

> **Dragon 15 will NOT work long-term on Windows 11.** Dragon 15 uses the Windows `JournalPlaybackHook` API to inject dictated text into apps. Microsoft deprecated this in Windows 11 and is removing it with each feature update. As of 24H2, dictation in browser address bars and many apps is already broken; it will get worse. Dragon 16 was rebuilt for Windows 11 and does not have this dependency.
>
> **You must buy Dragon Professional v16** (upgrade pricing is available from Dragon 15). The "Dragon Professional Anywhere" cloud product is NOT what you want — that's an enterprise server product. You want **Dragon Professional v16** (standalone desktop). Latest version as of early 2026 is approximately 16.10.x.

**Installer location:** `E:\Downloads\Important\Nuance Dragon Professional VLA 16.10.200.044` — already on E: drive, no purchase needed.

> This entire section is handled by Jamie, not Claude. Dragon installation and profile setup is a manual process.

**Install steps:**
- [ ] Run the Dragon installer from `E:\Downloads\Important\Nuance Dragon Professional VLA 16.10.200.044`
- [ ] Restore profile: copy `%APPDATA%\Nuance\NaturallySpeaking15\` backup from Phase 0 → place in `%APPDATA%\Nuance\NaturallySpeaking16\` (Dragon 16 can import Dragon 15 profiles)
  - Alternatively: if the profile import fails or Dragon behaves oddly, delete it and run the New User Wizard fresh — the acoustic model will adapt quickly
- [ ] Run Dragon's microphone check
- [ ] Test basic dictation before moving on
- [ ] Do NOT install Natlink yet — come back to that in Phase 2.1

**If you temporarily must use Dragon 15 on Windows 11** (e.g., while waiting for v16 to arrive):
- Avoid Windows 11 feature updates until Dragon 16 is installed
- Known broken in 23H2+: dictation in Chrome address bar, PDFs, file rename dialogs
- Workaround: use Dragon's "Dictation Box" and paste — unreliable but functional

### 1.6 AutoHotkey
- [ ] Install AutoHotkey (needed for Caster to function fully)
- [ ] Clone or pull AHK repo from GitHub (or let OneDrive restore it)
- [ ] Do NOT start MAINFUNCTIONS.ahk yet — wait until Caster is set up

### 1.7 Startup Program

`StartupOrder.bat` runs at login and launches Caster, AHK scripts, and other essentials in the correct order with delays between them.

**What it does:**
- Waits 10 seconds, then launches Caster
- Waits 95 more seconds, then launches `STARTUP.ahk`, `PARENTAUTOHOTKEY.ahk`, and `Footpedals.ahk`

**Restore steps:**
- [ ] Copy `E:\Old PC Backups\StartupOrder.bat` to the Startup folder on the new machine:
  `C:\Users\Jamie\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\`
- [ ] The bat uses `%USERPROFILE%` so paths will resolve correctly for the `Jamie` username automatically — no edits needed
- [ ] Recreate the `Caster.lnk` shortcut on the new machine — it must point to `C:\Users\Jamie\Documents\Caster\Run_Caster_DNS.bat` with working directory `C:\Users\Jamie\Documents\Caster`, and be saved to `C:\Users\Jamie\Desktop\Important\Caster.lnk`
  - The bat itself is fine — it uses `%~dp0` (self-relative path), no edits needed
- [Claude can help] Create the shortcut programmatically once on the new machine

### 1.8 Stream Deck
- [ ] Install Elgato Stream Deck software
- [ ] **Quit Stream Deck before restoring profiles**
- [ ] Pull Stream Deck profiles from GitHub
- [ ] Copy profiles to `%APPDATA%\Elgato\StreamDeck\ProfilesV3\`
- [ ] Open Stream Deck — it reads from disk on startup
- [ ] Verify buttons on both Deck A and Deck B
  - [Claude can help] Add or fix any buttons programmatically

---

## PHASE 2 — Voice Commands: Caster

Goal: Full voice command stack working so setup of everything else is faster.

### 2.1 Python + Natlink + Caster

> This section is the most technically involved part of the whole setup. Read it fully before starting. The order matters.

#### Why Python must be 32-bit
Natlink is the bridge between Dragon and Python. Dragon's plugin API is 32-bit, so Natlink must be 32-bit, which means Python must also be 32-bit. This is true even on a 64-bit machine. You can have a 64-bit Python for other things (e.g., Anaconda, VS Code) — they coexist fine — but Natlink and Caster must run on the 32-bit Python.

**Required Python version: 3.10.x 32-bit** (the Natlink installer may handle this automatically, but if not, get it from python.org — make sure to select the "Windows installer (32-bit)" option for Python 3.10)

---

#### Step-by-step install order

**Step 1: Install Python 3.10 32-bit**
- [ ] Go to python.org → Downloads → Python 3.10.x → scroll to "Files" → download **"Windows installer (32-bit)"**
- [ ] During install: check "Add Python to PATH" — but note this is the 32-bit Python, and you may not want it as the system default if you have other Python installs
- [ ] Verify after install: `py -3.10-32 --version` should return `Python 3.10.x`
- [Claude can help] Verify the Python architecture: `py -3.10-32 -c "import struct; print(struct.calcsize('P')*8)"` should print `32`

**Step 2: Install Natlink v5.5.7**
- [ ] Download from: github.com/dictation-toolbox/natlink/releases (get the latest `.exe` installer)
- [ ] Run the installer — it will detect your Dragon install and your Python 3.10 32-bit
- [ ] After install, open the Natlink configuration GUI (it creates a Start Menu entry: "Configure NatLink via GUI")
- [ ] In the GUI: click **(re)Register NatLink** — this registers Natlink as a Dragon plugin
- [ ] Restart Dragon — if Natlink is working, Dragon's splash screen will mention Natlink loading, and a "Messages from NatLink" window may appear

> **If Natlink doesn't register:** This usually means either (a) wrong Python architecture — Natlink found a 64-bit Python instead of 32-bit, or (b) Dragon wasn't fully closed when you ran the installer. Close Dragon completely, re-run the Natlink installer, re-register via GUI.

**Step 3: Clone the Caster repos from GitHub**
- [ ] Clone `user-caster` (your rules + settings) to `C:\Users\Jamie\AppData\Local\caster` via SSH
- [ ] Clone `caster-main` (the Caster source) to `C:\Users\Jamie\Documents\Caster` via SSH
  - SSH must be set up first — see the SSH section below

**Step 4: Install Caster Python dependencies**
- [ ] Using the **32-bit pip** (important!): `py -3.10-32 -m pip install -r C:\Users\Jamie\Documents\Caster\requirements.txt`
- [ ] If `wxpython` install fails, try: `py -3.10-32 -m pip install wxpython` (without a pinned version)
- [Claude can help] Debug any pip install failures

**Step 5: Update settings.toml**
- [ ] Open `C:\Users\Jamie\AppData\Local\caster\settings\settings.toml`
- [ ] Update all `C:\Users\Nate` references → `C:\Users\Jamie`
  - Specifically: `USER_DIR`, `BASE_PATH`, `ENGINE_PATH`, `REBOOT_PATH`, `AHK_PATH`, `DLL_PATH`, and all other path entries
  - `ENGINE_PATH` should now point to Dragon 16: `C:\Program Files (x86)\Nuance\NaturallySpeaking16\Program\natspeak.exe` (verify the actual install path)
- [Claude can help] Do the full search-and-replace on settings.toml once you're on the new machine

**Step 6: Verify rules are enabled**
- [ ] Open `C:\Users\Jamie\AppData\Local\caster\settings\rules.toml`
- [ ] Confirm all your rule names are listed in `_enabled_ordered`
- [Claude can help] Compare the rules.toml list against the actual .py files in the rules/ folder

**Step 7: Test Caster**
- [ ] Start Dragon first, then Caster (or use your startup program)
- [ ] Say the test phrase from the first rule in any file (each rule has a debug entry at the top — e.g., "Example Commands are working")
- [ ] If a rule fails to load, Dragon or the Natlink Messages window will show the Python error
- [Claude can help] Debug any rule load errors — most are import path issues or settings.toml path problems

### 2.2 Startup Program
- [ ] Pull startup program from OneDrive/GitHub
- [ ] Run it — this should start Dragon, Caster, AHK, and other components together
- [ ] Test a few voice commands to confirm Caster + AHK bridge is working
  - [Claude can help] Debug any rule or AHK wiring issues

---

## PHASE 3 — Display & Window Management

Goal: Screen layout and virtual desktops working.

- [ ] **Display Fusion** — install, import settings from Phase 0 backup
  - License: Jamie Hooper
    ```
    101-02-MOAPT66FCF-PQLQIBC3DF-zyd03mTZJQd/D17fzXmsrR_Vpzfqz1tICOSqLDrPz
    bzekhNBPDLUeUPRd1H0wDwZ18usnjKzp2HxaQ_W61kbgXcliO4f2PgkgZ3X35y4Mrc38E3
    6Y2Dl7Ndc7M1Djo8l7znOexDCgbyVR_FtwTmKFzc_DUNXb6SnaCFeRVjxcrA=
    ```
  - Settings backup: `Backups from old PC/DisplayFusion Backup (2026-03-26).reg` in `desktop-important`
  - Configure monitor layout, taskbars, hotkeys
- [ ] **Microsoft PowerToys / FancyZones** — install from Microsoft Store or GitHub
  - Restore settings: copy contents of `Backups from old PC/PowerToys/` from `desktop-important` to `%LOCALAPPDATA%\Microsoft\PowerToys\`
  - FancyZones layouts, hotkeys, and all module settings will be restored
- [ ] **Windows Virtual Desktop Enhancer** — pull from OneDrive/GitHub and install
  - Verify virtual desktop hotkeys work with AHK

---

## PHASE 4 — Development Tools

- [ ] **VS Code**
  - Install, sign in to sync account → extensions/settings restore automatically
  - Verify Claude Code extension is installed
  - [Claude can help] Any workspace/settings config
- [ ] **Notepad++** — pull config from GitHub, install plugins if needed
- [ ] **Claude** (claude.ai) — already accessible via Chrome; Claude Code via VS Code

---

## PHASE 5 — Remaining Apps (order doesn't matter much)

These don't affect accessibility, install as needed:

- [ ] **Anki** — log in to AnkiWeb account, decks sync from cloud
- [ ] **Calibre** — install, point library at wherever books are stored
- [ ] **Python** — install before qBittorrent search will work (qBittorrent may prompt for this)
- [ ] **Jackett** — install and run; it powers the jackett search engine in qBittorrent
  - After install, open Jackett (localhost:9117), copy the API key
  - Paste it into `%LOCALAPPDATA%\qBittorrent\nova3\engines\jackett.json` under `"api_key"` (the copy in `Backups from old PC` has a placeholder — update it after Jackett install)
- [ ] **qBittorrent** — install, then restore search plugins:
  - Copy `Backups from old PC/qBittorrent_nova3` folder from `desktop-important` to `%LOCALAPPDATA%\qBittorrent\nova3`
  - Rutracker will re-authenticate automatically on first search (credentials saved in `rutracker.json`)
  - All other engines (piratebay, solidtorrents, limetorrents, etc.) work immediately
- [ ] **VoiceMeeter** — install, then copy `Backups from old PC/VoiceMeeterDefault.xml` from `desktop-important` to `%APPDATA%\VoiceMeeterDefault.xml`
- [ ] **OpenRGB** — install, then copy `Backups from old PC/OpenRGB.json` from `desktop-important` to `%APPDATA%\OpenRGB\OpenRGB.json`
- [ ] **Audacity** — install, then restore settings from `Backups from old PC/Audacity/` in `desktop-important`:
  - Copy `audacity.cfg`, `pluginsettings.cfg`, `pluginregistry.cfg` to `%APPDATA%\audacity\`
- [ ] **f.lux** — install, sign in to account (settings sync via account)
- [ ] **ShareX** — installed on E: drive (keeping E: drive, so may already be present)
  - Restore hotkeys/settings: copy `Backups from old PC/ShareX/` from `desktop-important` to `Documents\ShareX\`
  - If reinstall needed, grab installer from E: drive or sharex.github.io
- [ ] **ClipboardFusion** — install, then copy `Backups from old PC/clipboardfusion.db` from `desktop-important` to `%LOCALAPPDATA%\ClipboardFusion\clipboardfusion.db`
- [ ] **KeyTik** — install, then copy `Backups from old PC/KeyTik/` from `desktop-important` to `%APPDATA%\KeyTik\` (restores pinned AHK profiles)
- [ ] **Everything** (voidtools) — install, indexing runs automatically
- [ ] **Signal** — install, link as linked device or restore from backup
- [ ] **Kindle** — install, log in to Amazon account

---

## PHASE 6 — Optional

Install if/when needed.

- [ ] **SoundSwitch** — install, copy `Backups from old PC/SoundSwitchConfiguration.json` from `desktop-important` to `%APPDATA%\SoundSwitch\SoundSwitchConfiguration.json`
- [ ] **Tobii** — install drivers + software (hardware dependency)
- [ ] **eViacam** — install and reconfigure (no backup; settings were not exported)
- [ ] **PrecisionGazeMouse** — install and reconfigure (no backup; settings were not exported)

---

## Reference: Where Everything Lives

| Tool | Backup Location |
|---|---|
| Dragon profile | Phase 0 backup to `desktop-important` + OneDrive. Lives at `%APPDATA%\Nuance\NaturallySpeaking15\` (Dragon 15) or `\NaturallySpeaking16\` (Dragon 16) |
| Caster rules folder | OneDrive + GitHub |
| Caster AppData folder | OneDrive + GitHub |
| Startup program | OneDrive + GitHub |
| AutoHotkey | GitHub |
| Stream Deck profiles | GitHub |
| Notepad++ config | GitHub |
| Windows Virtual Desktop Enhancer | OneDrive + GitHub |
| eViacam | **Nowhere — back up in Phase 0** |
| PrecisionGazeMouse | **Nowhere — back up in Phase 0** |
| Display Fusion | GitHub (`desktop-important`) — `Backups from old PC/DisplayFusion Backup (2026-03-26).reg` |
| SoundSwitch | GitHub (`desktop-important`) — `Backups from old PC/SoundSwitchConfiguration.json` |
| VoiceMeeter | GitHub (`desktop-important`) — `Backups from old PC/VoiceMeeterDefault.xml` |
| OpenRGB | GitHub (`desktop-important`) — `Backups from old PC/OpenRGB.json` |
| Tobii | Likely re-downloadable; check account |
| Vimium | Google account sync |
| VS Code settings | Microsoft account sync |
| Anki decks | AnkiWeb account |
| qBittorrent search plugins | GitHub (`desktop-important`) — `Backups from old PC/qBittorrent_nova3/` |
| Audacity settings | GitHub (`desktop-important`) — `Backups from old PC/Audacity/` |
| PowerToys / FancyZones | GitHub (`desktop-important`) — `Backups from old PC/PowerToys/` |
| ShareX | GitHub (`desktop-important`) — `Backups from old PC/ShareX/` (hotkeys, app config, uploaders) |
| ClipboardFusion | GitHub (`desktop-important`) — `Backups from old PC/clipboardfusion.db` |
| KeyTik | GitHub (`desktop-important`) — `Backups from old PC/KeyTik/` |
| f.lux | Account sync |

---

## Notes for Claude

- This system uses Dragon Professional v16 + Natlink v5.5.7 + Caster/dragonfly2 (on the new machine; currently Dragon 15 on Windows 10)
- Python for Caster/Natlink must be **32-bit, specifically Python 3.10.x 32-bit** — do not use 64-bit Python for Caster-related tasks
- AHK is called via `MAINFUN.bat` → `MAINFUNCTIONS.ahk` which `#Include`s helper files
- Caster rules are at `C:\Users\Jamie\AppData\Local\caster\rules\` (new machine) / `C:\Users\Nate\AppData\Local\caster\rules\` (current machine)
- All rule files use `castervoice.lib.*` imports — no migration needed on that front
- Stream Deck profiles are in GitHub; edits go to ProfilesV3 (not V2)
- User is Jamie (she/her); file paths use Nate on current machine (legacy), will use Jamie on new machine
- Default to Deck A for Stream Deck operations unless Deck B is specified
- Complex voice command logic belongs in AHK, not Caster Python files
- `settings.toml` has many hardcoded `C:\Users\Nate` paths that need updating to `C:\Users\Jamie` on the new machine

---

## Git Repositories — Backup Inventory

All repos are under `JamieHooper-code` on GitHub. Two repos use SSH (`git@github.com`) — SSH keys must be set up on the new machine before those can be cloned.

| Repo | GitHub URL | Clone path on new machine | Protocol |
|---|---|---|---|
| user-caster (rules + settings) | github.com/JamieHooper-code/user-caster | `C:\Users\Nate\AppData\Local\caster` | SSH |
| streamdeck-backup | github.com/JamieHooper-code/streamdeck-backup | `%APPDATA%\Elgato\StreamDeck` | HTTPS |
| sublime-backup | github.com/JamieHooper-code/sublime-backup | `%APPDATA%\Sublime Text\Packages\User` | SSH |
| autohotkey | github.com/JamieHooper-code/autohotkey | `C:\Users\Nate\Desktop\Important\AutoHotkey` | HTTPS |
| textdocs | github.com/JamieHooper-code/textdocs | `C:\Users\Nate\Desktop\Important\TEXTDOCS` | HTTPS |
| chrome-newtab | github.com/JamieHooper-code/new-tab | `C:\Users\Nate\Desktop\Important\chrome-newtab` | HTTPS |
| caster-main | github.com/JamieHooper-code/caster-main | `C:\Users\Nate\Documents\Caster` | SSH |
| claude-config-backup | github.com/JamieHooper-code/claude-config-backup | `C:\Users\Nate\.claude` | HTTPS |
| desktop-important | github.com/JamieHooper-code/desktop-important | `C:\Users\Nate\Desktop\Important` | HTTPS |

### Uncommitted changes — push before wiping
- **user-caster** — `rules/.claude/settings.local.json` (local settings file, low priority)
- **textdocs** — this guide and other untracked files should be committed before wiping

### SSH setup on new machine
Before cloning the SSH repos (user-caster, sublime-backup, caster-main):
1. Generate a new SSH key: `ssh-keygen -t ed25519 -C "your_email"`
2. Add public key to GitHub → Settings → SSH Keys
3. Then clone normally

### Notes
- `Desktop/Important` → `github.com/JamieHooper-code/desktop-important` (HTTPS) — excludes AutoHotkey/, TEXTDOCS/, chrome-newtab/, BOOKS/, Bitwig installers
- [Claude can help] Run the pushes for any of the uncommitted repos above

---

## Username Change: Nate → Jamie

The new Windows 11 install will use `Jamie` as the username instead of `Nate`. This affects any path starting with `C:\Users\Nate\`.

### Already fixed (dynamic — will just work)
All Python/Caster rule files now use `os.environ` at load time:
- `windows_commands.py` — all BringApp paths and Choice directory/edit_location dicts
- `chrome_commands.py` — SAVE_THE_LISTS path
- `command_line_commands.py` — OneDrive symlink command
- `reading_commands.py` — user path

### Needs manual attention on the new machine

**Claude memory folder path**
The memory folder is stored at a path derived from the project directory name:
`~\.claude\projects\c--Users-Nate-AppData-Local-caster\memory\`
On the new machine this will be `c--Users-Jamie-AppData-Local-caster`. After cloning `claude-config-backup`, manually move/rename the memory folder:
```
C:\Users\Jamie\.claude\projects\c--Users-Nate-AppData-Local-caster\memory\
→ rename parent folder to: c--Users-Jamie-AppData-Local-caster
```
[Claude can help] I can do this rename automatically once running on the new machine.

**Stream Deck button paths**
Stream Deck buttons that call `MAINFUN.bat` use relative-style calls (just `MAINFUN.bat FunctionName`) so those should be fine. If any buttons have full absolute paths, they'll need updating. Check after setup.

**App version paths in windows_commands.py**
`BringApp` calls still contain version numbers (e.g. `Discord\app-1.0.9016`). These will likely change on reinstall — update them once apps are installed.
[Claude can help] Run "go discord" and if it fails, update the version number in the path.

**Dragon profile**
Dragon stores the profile in OneDrive, but the profile may contain internal references to `C:\Users\Nate`. If Dragon has issues after restoring the profile, re-run acoustic training or create a new profile.

**Caster settings files**
`settings.toml` has many hardcoded `C:\Users\Nate` paths that all need updating to `C:\Users\Jamie`. Specifically: `USER_DIR`, `BASE_PATH`, `ENGINE_PATH`, `REBOOT_PATH`, `AHK_PATH`, `DLL_PATH`, and others. Also update `ENGINE_PATH` to point to Dragon 16 (`NaturallySpeaking16`) instead of Dragon 15 (`NaturallySpeaking15`).
[Claude can help] Do a full search-and-replace on settings.toml — just say "fix settings.toml paths" and I'll handle it.

**Custom PATH entries to re-add**
These were manually added and won't be set automatically. Add via System Properties > Environment Variables > User PATH:
- `C:\Users\Jamie\bin` — personal scripts folder (rename from Nate)
- `C:\Users\Jamie\Desktop\Important\PowershellScripts` — custom PowerShell scripts
- `C:\Users\Jamie\Desktop\Important\AutoHotkey\Scripts` — AHK scripts (already in GitHub)
- `C:\Program Files\AutoHotkey\v2` — set by AHK installer
- `C:\Program Files\GitHub CLI` — set by gh installer
- `C:\Program Files\ImageMagick-7.1.0-Q16-HDRI` — note version number may change on reinstall
- `C:\Program Files\gs\gs10.00.0\bin` — Ghostscript, note version number
- `C:\Program Files\Microsoft\jdk-21.0.5.11-hotspot\bin` — JDK 21, note version number
- `C:\ProgramData\chocolatey\bin` — set by Chocolatey installer
- `C:\Program Files\nodejs` — set by Node.js installer
- `C:\Users\Jamie\anaconda3` + sub-paths — set by Anaconda installer
- `E:\Program Files\Git\cmd` etc. — Git is on E: drive; if reinstalling Git, keep it on E: or update these paths
Note: PATH currently has many duplicates — good opportunity to clean it up on the fresh install.

---

## Completed Pre-Wipe Backups

These are done — no further action needed before wiping.

| Item | Backup Location | Restore Instructions |
|---|---|---|
| **Display Fusion** | `desktop-important/Backups from old PC/DisplayFusion Backup (2026-03-26).reg` + `E:\Old PC Backups\` | Double-click the `.reg` file after installing Display Fusion |
| **PowerToys / FancyZones** | `desktop-important/Backups from old PC/PowerToys/` + `E:\Old PC Backups\` | Copy contents to `%LOCALAPPDATA%\Microsoft\PowerToys\` after installing PowerToys |
| **Dragon profile** | `E:\Old PC Backups\Nate_DgnRenamed_DgnRenamed_DgnRenamed\` (1.1 GB, from `C:\ProgramData\Nuance\NaturallySpeaking15\Users\`) | Dragon menu → Manage User Profiles → Restore User Profile → point at `E:\Old PC Backups\Nate_DgnRenamed_DgnRenamed_DgnRenamed\` |

---

## Loose Notes

**Foot pedal (Savant Elite 2):** No driver needed — plug-and-play HID. All programming is stored on the pedal's onboard memory, so just plug it in and it works. SmartSet App (for reprogramming) is available at kinesis-ergo.com/support/savant-elite2/ if needed, no account required.

**Eye/head tracking:** Tobii, eViacam, and PrecisionGazeMouse have no backups. Will need to be reconfigured from scratch after install. eViacam and PrecisionGazeMouse settings are not documented anywhere — expect to spend time re-tuning sensitivity and behavior.

**SoundSwitch:** Config backed up to `Backups from old PC/SoundSwitchConfiguration.json` — see Phase 6 for restore instructions.

---

## Future Consideration: Talon Voice

> This is not an action item. Just something to be aware of for the long term.

The Dragon + Natlink + Caster stack that this setup uses is aging. As of early 2026:

- **Caster** is in maintenance mode — sporadic bug fixes, no new development, last formal release was 2019
- **Dragon Professional v16** will likely be the last standalone desktop Dragon ever released — Nuance (now owned by Microsoft) is investing in cloud/enterprise products, not the desktop version
- **The whole dependency chain** (Dragon → Natlink → Dragonfly2 → Caster) is low-activity; any link could break on a future Windows update with a slow or no fix

**Talon Voice** (talonvoice.com) is where the voice coding community has largely moved. Key points:
- Does not require Dragon — has its own built-in speech engine (Conformer D), competitive with Dragon for command recognition
- Actively developed, large community, free at the base tier
- Works on Windows 11, macOS, and Linux
- Your existing Caster rules can't be mechanically ported — the syntax is different — but the concepts map directly and the rewrite would take days, not weeks

**The realistic timeline:** Dragon 16 + Caster will likely continue working for another 2–4 years before something breaks irreparably. No need to act now. But if there's ever a moment where Dragon or Caster breaks badly and requires significant debugging, that's the natural time to evaluate switching rather than fighting to fix legacy infrastructure.
