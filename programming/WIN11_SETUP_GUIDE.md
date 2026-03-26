# Windows 11 Fresh Install — Setup Guide
> Organized by priority: get accessible first, then everything else.
> Items marked [Claude can help] are ones I can assist with directly.

---

## PHASE 0 — DO BEFORE YOU WIPE (critical)

These have no backups. Export/document settings now or they're gone.

- [ ] **Display Fusion** — export settings via Settings > Backup > Export (no backup exists)
- [ ] **PowerToys / FancyZones** — export via PowerToys Settings > General > Backup & Restore (no backup exists)
- [ ] **VS Code** — Settings Sync is tied to your Microsoft/GitHub account; verify it's enabled (Settings > Turn on Settings Sync) so extensions and config restore automatically

Verify GitHub/OneDrive has latest versions of:
- [ ] Caster rules (rules/ folder) — push any uncommitted changes
- [ ] AutoHotkey files
- [ ] Startup program
- [ ] Stream Deck profiles (already in GitHub per notes)

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

### 1.3 Dragon NaturallySpeaking
- [ ] Install Dragon NaturallySpeaking 15
- [ ] Restore profile from OneDrive (should be at known path in OneDrive)
- [ ] Run through Dragon's microphone check
- [ ] Test basic dictation before moving on

### 1.4 Claude
- [ ] Open claude.ai in Chrome — usable immediately, no install needed
- [ ] Install Claude Code extension in VS Code once VS Code is up (Phase 4)

### 1.5 AutoHotkey
- [ ] Install AutoHotkey (needed for Caster to function fully)
- [ ] Clone or pull AHK repo from GitHub (or let OneDrive restore it)
- [ ] Do NOT start MAINFUNCTIONS.ahk yet — wait until Caster is set up

---

## PHASE 2 — Voice Commands: Caster + Stream Deck

Goal: Full voice command stack working so setup of everything else is faster.

### 2.1 Python + Caster Dependencies
- [ ] Install Python (check which version Caster requires — likely 3.x)
- [ ] Install Caster and dragonfly2 via pip
- [ ] Restore both Caster folders from OneDrive/GitHub:
  - AppData Caster folder (user data/settings)
  - Rules folder (the GitHub repo at `c:\Users\Nate\AppData\Local\caster\rules`)
- [ ] Verify `settings/rules.toml` has all rules listed in `_enabled_ordered`

### 2.2 Startup Program
- [ ] Pull startup program from OneDrive/GitHub
- [ ] Run it — this should start Dragon, Caster, AHK, and other components together
- [ ] Test a few voice commands to confirm Caster + AHK bridge is working
  - [Claude can help] Debug any rule or AHK wiring issues

### 2.3 Stream Deck
- [ ] Install Elgato Stream Deck software
- [ ] **Quit Stream Deck before restoring profiles**
- [ ] Pull Stream Deck profiles from GitHub
- [ ] Copy profiles to `%APPDATA%\Elgato\StreamDeck\ProfilesV3\`
- [ ] Open Stream Deck — it reads from disk on startup
- [ ] Verify buttons on both Deck A and Deck B
  - [Claude can help] Add or fix any buttons programmatically

### 2.4 SoundSwitch
- [ ] Install SoundSwitch
- [ ] Restore profile/settings from Phase 0 backup
- [ ] Verify tilde hotkey → AHK macro connection works

---

## PHASE 3 — Display & Window Management

Goal: Screen layout and virtual desktops working.

- [ ] **Display Fusion** — install, import settings from Phase 0 backup
  - Configure monitor layout, taskbars, hotkeys
- [ ] **Microsoft PowerToys / FancyZones** — install from Microsoft Store or GitHub
  - Set up zone layouts for each monitor
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
- [ ] **qBittorrent** — install, restore settings if backed up
- [ ] **Everything** (voidtools) — install, indexing runs automatically
- [ ] **Signal** — install, link as linked device or restore from backup
- [ ] **Kindle** — install, log in to Amazon account

---

## PHASE 6 — Optional: Eye/Head Tracking

Install if/when needed. None of these are required for the core accessibility stack.

- [ ] **Tobii** — install drivers + software (hardware dependency)
- [ ] **eViacam** — install and reconfigure (no backup; settings were not exported)
- [ ] **PrecisionGazeMouse** — install and reconfigure (no backup; settings were not exported)

---

## Reference: Where Everything Lives

| Tool | Backup Location |
|---|---|
| Dragon profile | OneDrive |
| Caster rules folder | OneDrive + GitHub |
| Caster AppData folder | OneDrive + GitHub |
| Startup program | OneDrive + GitHub |
| AutoHotkey | GitHub |
| Stream Deck profiles | GitHub |
| Notepad++ config | GitHub |
| Windows Virtual Desktop Enhancer | OneDrive + GitHub |
| eViacam | **Nowhere — back up in Phase 0** |
| PrecisionGazeMouse | **Nowhere — back up in Phase 0** |
| Display Fusion | **Nowhere — back up in Phase 0** |
| Tobii | Likely re-downloadable; check account |
| SoundSwitch | Check if it has export |
| Vimium | Google account sync |
| VS Code settings | Microsoft account sync |
| Anki decks | AnkiWeb account |

---

## Notes for Claude

- This system uses Dragon NaturallySpeaking 15 + Caster/dragonfly2
- AHK is called via `MAINFUN.bat` → `MAINFUNCTIONS.ahk` which `#Include`s helper files
- Caster rules are at `c:\Users\Nate\AppData\Local\caster\rules\`
- Stream Deck profiles are in GitHub; edits go to ProfilesV3 (not V2)
- User goes by Jamie; file paths use Nate (legacy)
- Default to Deck A for Stream Deck operations unless Deck B is specified
- Complex voice command logic belongs in AHK, not Caster Python files

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

### Uncommitted changes — push before wiping
These repos have local changes not yet on GitHub:

- **user-caster** — `rules/.claude/settings.local.json`, `rules/ahk_mainfun.py`, `rules/links_commands.py`, `settings/rules.toml`, `settings/settings.toml`
- **streamdeck-backup** — plugin logs + cache (probably safe to ignore, but check)
- **autohotkey** — `Helpers/LinkManager.ahk`, `INIDATA/MAINFUNCTIONS.ini`, `MAINFUNCTIONS.ahk`, and untracked `Helpers/StreamDeckManager.ahk`
- **textdocs** — `programming/TODOPROGRAMMING.txt` and untracked files including this guide
- **chrome-newtab** — `index.html`

### SSH setup on new machine
Before cloning the SSH repos (user-caster, sublime-backup, caster-main):
1. Generate a new SSH key: `ssh-keygen -t ed25519 -C "your_email"`
2. Add public key to GitHub → Settings → SSH Keys
3. Then clone normally

### Notes
- `Desktop/Important` has a `.git` folder but no remote — local-only repo, not backed up to GitHub
- [Claude can help] Run the pushes for any of the uncommitted repos above

---

## Loose Notes

**Eye/head tracking:** Tobii, eViacam, and PrecisionGazeMouse have no backups. Will need to be reconfigured from scratch after install. eViacam and PrecisionGazeMouse settings are not documented anywhere — expect to spend time re-tuning sensitivity and behavior.

**SoundSwitch:** No backup. Will need to be reconfigured after install — note hotkey settings and microphone profiles before wiping.
