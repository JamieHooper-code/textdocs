# SUPERSEDED — Migrating to a New Computer

> **Superseded 2026-05-01 by `MIGRATE_DELL_TO_ACER.md`.** This doc was written before the target hardware was known, and includes Dragon 15→16 upgrade context (no longer relevant — already on Dragon 16), Nate→Jamie username flip context (no longer relevant — already Jamie), and an RTX 2070 install path (no longer relevant — Acer Nitro 60 ships with an RTX 5060 Ti 16GB, which strictly outperforms the 2070). Use `MIGRATE_DELL_TO_ACER.md` for the active plan; this file is kept for the workflow shape and gotcha list, but do not follow it as a literal checklist.

---

## Why this exists

Returning the Dell ECT1250 (proprietary chassis, can't take the RTX 2070) and replacing with a standard-ATX prebuilt. The voice/AHK/Caster/Stream Deck stack is too involved to redo from scratch if avoidable — cloning the SSD preserves every installed app, driver, activation, config, and voice profile. This is the plan that does that cleanly, with a documented fallback to fresh install if cloning fails.

Primary dependencies that make clone preferable over fresh:
- Dragon NaturallySpeaking 16 + activation count
- Natlink 5.5.7 + Python 3.10 32-bit registration + local `castervoice\lib\utilities.py` patch
- Caster voice profile tuning (months of accumulated tweaks)
- DisplayFusion license + hotkey registry entries
- Stream Deck app's internal state (settings, SF Symbols pack path)
- Various minor app licenses (VS Code sync, Chrome profile, etc.)
- Windows-level config (power profiles, audio routing, pinned apps, shortcuts)

Git-tracked surfaces (AHK repo, Caster rules, SD profiles, `.claude/`, project repos, OneDrive-synced Desktop) survive either way — they're recovered by cloning the repos. The clone's value is everything NOT in git.

---

## Timeline — order matters

**Do NOT return the Dell before the new machine works.** You need a working machine between now and the replacement. The correct order:

1. **Now** (while Dell still in return window) → Pre-return prep below.
2. **New machine arrives** → Clone SSD on external enclosure → install in new machine → first boot → verify.
3. **After new machine is confirmed working** → Return the Dell.
4. **If clone fails** → Fresh install on new machine using `WIN11_SETUP_GUIDE.md`.

---

## Phase 1 — Pre-return prep (do these now)

### 1. Link Windows activation to your Microsoft account

- Settings → Accounts → Your info → "Sign in with a Microsoft account instead" (if not already).
- Settings → System → Activation — confirm message reads **"Windows is activated with a digital license linked to your Microsoft account"**.
- If it reads "activated" without mentioning the Microsoft account, the key is likely OEM-baked into the Dell motherboard. That key will NOT transfer. Options:
  - Link it now (may or may not convert, try once)
  - Budget ~$140 for a new Windows 11 Pro key
  - Use the new prebuilt's included Windows license

### 2. Check Dragon NaturallySpeaking activation status

- Open Dragon → Help → About.
- Note the license info and remaining activation count (usually starts at 3 seats).
- If you're down to 1 seat, contact Nuance support **before** the migration to request a seat reset. Cloning may or may not trigger re-activation; don't gamble your last seat on it.

### 3. Save BitLocker recovery key (if BitLocker is on)

- Settings → Privacy & security → Device encryption.
- If enabled: either disable before cloning, OR save the recovery key offline (print + USB, not just "sync to Microsoft account").
- BitLocker-enabled drives will refuse to boot on different hardware without the key.

### 4. Verify keyfile + secret backups are current on E:

Confirm these exist on your E: external drive before you unplug anything:
- `Scripts/CopyPasteSlotCrypt/copy_paste_slot_crypt.key` — slot encryption key (without it, every `.enc` slot is permanently unreadable)
- `.atlantic_graph_certs/` if Atlantic work — McLeod API key, service principal `.env`, `.pfx` cert for subjectorders@shipatlantic.com
- Dragon user profile folder — typically `%LOCALAPPDATA%\Nuance\NS16\Users\` — copy to E: as belt-and-suspenders even if the clone carries it

### 5. Push everything to git

Every tracked repo should be pushed to origin so the clone failure path is survivable:
- `C:\Users\jamie\Desktop\Important\AutoHotkey\`
- `C:\Users\jamie\AppData\Local\caster\rules\`
- `C:\Users\jamie\AppData\Local\caster\settings\` (if separately tracked)
- `C:\Users\jamie\AppData\Roaming\Elgato\StreamDeck\ProfilesV3\`
- `C:\Users\jamie\.claude\`
- Every project under `Desktop\Important\projects\`
- `C:\Users\jamie\Desktop\Important\TEXTDOCS\` if tracked

`GitPushAll.ps1` handles this — run it once before the migration day.

### 6. Buy the USB-to-M.2 NVMe enclosure

You need one to clone. ~$15-30. Get a 10 Gbps USB 3.2 Gen 2 or better — saves ~40 minutes on the clone of a 1 TB drive.

---

## Phase 2 — New machine arrival

### Install cloning software on the Dell (before shutting down)

- **Macrium Reflect Free** (recommended) — `macrium.com/reflectfree`. Supports imaging AND direct clone-to-target.
- OR **Clonezilla** — bootable USB, more technical but free and robust.
- OR **Samsung Magician** — only works if your target SSD is Samsung.

### Take the clone

1. Shut down the new machine. Don't boot it yet with its stock Windows install.
2. Open the new machine's case, pull the stock SSD (save it — you may want to swap back).
3. Put the stock SSD into the USB-to-M.2 enclosure on the Dell side (you'll use it as your clone target).
4. On the Dell: run Macrium Reflect → clone `C:` (Dell internal SSD) → target = the enclosure drive.
5. Wait ~20-40 min depending on drive size + USB speed.
6. Shut down the Dell. Pull the cloned SSD from the enclosure. Install it in the new machine's M.2 slot.

**Alternative if you don't trust clone-before-verify:** image the Dell SSD to a file on E:, then after the new machine boots up successfully, restore the image onto its stock SSD. Same outcome, slightly safer because you have an image file as backup.

---

## Phase 3 — First boot on new hardware

Expect 5-15 minutes of Windows auto-discovering drivers and reboots. This is normal.

### What will happen

1. **First boot** — Windows detects new hardware, installs generic drivers, reboots.
2. **Second boot** — probably logs you in, screen may be at wrong resolution. Device Manager will show some yellow ⚠️ on unknown devices.
3. **Windows activation** — may show "Not activated." Run Settings → Activation → Troubleshoot. If linked to Microsoft account, it should reactivate.

### Driver cleanup (uninstall Dell-specific software)

Control Panel → Programs and Features → uninstall anything Dell-branded:
- Dell SupportAssist
- Dell Update
- Dell Command | Update
- Dell Mobile Connect
- Dell Digital Delivery
- Dell Display Manager
- Dell Optimizer

These conflict with non-Dell hardware and occasionally cause boot issues.

### Install new motherboard chipset drivers

The prebuilt should have come with a driver USB or download link. Install the chipset driver for the new motherboard's brand (MSI, ASUS, Gigabyte, ASRock). This replaces the Intel generic drivers Windows put in.

### Install GPU driver

- Download latest **NVIDIA Studio driver** from nvidia.com (or Game Ready if you prefer).
- Uninstall any lingering Intel Graphics driver through "NVIDIA-only" custom install if desired.
- Reboot. `nvidia-smi` should now list the RTX 2070.

---

## Phase 4 — Verify the stack works

Systematically, top to bottom. If any step fails, note it before proceeding — issues tend to compound.

### Core OS / hardware
- [ ] `nvidia-smi` shows RTX 2070, driver version, 8 GB VRAM
- [ ] Display resolution + refresh rate correct on all monitors
- [ ] Audio routing (speakers, mic) works
- [ ] USB devices enumerate (keyboard, mouse, Stream Deck, foot pedals, Arduino, Tobii, mic)
- [ ] Network works (wifi/ethernet)
- [ ] Windows activation is green

### Dev toolchain
- [ ] `py --version` returns Python 3.10.x (32-bit — important for Natlink)
- [ ] `py -3 --version` / `python --version` match expectations
- [ ] `git --version` works; `git config --global user.name` shows your name
- [ ] VS Code opens, signed in, Claude extension works
- [ ] `claude` CLI works (`claude --version`)
- [ ] `qmd status` works (QMD plugin survives the clone)

### AHK
- [ ] `C:\Users\jamie\AppData\Local\Programs\AutoHotkey\v2\AutoHotkey64.exe` exists
- [ ] `MAINFUN.bat` on PATH (`where MAINFUN.bat`)
- [ ] `MAINFUN.bat CheckProcessRunning chrome.exe` returns without error
- [ ] `MAINFUN.bat ReloadWithNotice` fires

### Dragon / Natlink / Caster
- [ ] Dragon NaturallySpeaking starts — may prompt for re-activation (burns a seat if so)
- [ ] Dragon recognizes your voice profile (should have carried over in the clone; if not, re-import from E: backup)
- [ ] Natlink loads on Dragon start (tray icon shows Natlink status; no error dialog)
- [ ] Caster loads (check `caster_messages.log` for startup banner)
- [ ] Say `"<domain> Commands"` for a few rules — debug probes fire
- [ ] Say `"reboot caster"` — Caster restarts cleanly (confirms the `utilities.py` patch survived)
- [ ] Say an AHK-dispatched voice command — check `ahk_event.log` for `DISPATCH/in` + `DISPATCH/ok`

### Stream Deck
- [ ] Stream Deck app opens, sees both Deck A and Deck B
- [ ] Profiles load from `ProfilesV3/` (should be the case — git-tracked folder carried over)
- [ ] Press a button — fires via `MAINFUN.bat`

### Project-specific
- [ ] DisplayFusion starts, license is active, hotkeys work (Win+Ctrl+Alt+1-8 snap)
- [ ] Everything (voidtools) service is running; `verify.ps1 search "notepad.exe"` returns a result
- [ ] OneDrive signed in, Desktop/Documents/Important synced
- [ ] Chrome syncs extensions + bookmarks
- [ ] VS Code settings sync landed

### Voice + AHK end-to-end smoke test
Say each of:
- [ ] `"reboot caster"` → works
- [ ] `"show VS code commands"` → browser Gui appears on leftmost monitor
- [ ] `"swallow test"` → copies selection to slot
- [ ] `"spit test"` → pastes slot
- [ ] A voice command that dispatches to an AHK function that opens an app

---

## Phase 5 — Install the RTX 2070

Once the above checklist is clean, the 2070 install from `INSTALL_RTX_2070.md` is straightforward. Physical install → NVIDIA Studio driver (already done above) → CUDA Toolkit → node-llama-cpp rebuild → QMD daemon benchmark.

---

## Plan B — Clone failed, fresh install

If the cloned SSD won't boot, BSODs repeatedly, or Dragon refuses to re-activate:

1. Pull cloned SSD, reinstall new machine's stock SSD with its included Windows.
2. Follow `WIN11_SETUP_GUIDE.md` top to bottom (Natlink, Python 3.10 32-bit, Dragon, Caster, etc.).
3. `git clone` everything back:
   - `AutoHotkey/` into `Desktop/Important/`
   - `caster/rules/` into `AppData/Local/caster/`
   - `StreamDeck/ProfilesV3/` into `AppData/Roaming/Elgato/StreamDeck/`
   - `.claude/` into the user home
   - Projects into `Desktop/Important/projects/`
4. Restore keyfile from E: to `Scripts/CopyPasteSlotCrypt/`.
5. Restore Dragon profile from E: if not preserved.
6. Re-apply the `castervoice\lib\utilities.py` patch (diff in `WIN11_SETUP_GUIDE.md §2.1 Step 8`).
7. Run the Phase 4 checklist to verify.

Budget: one full weekend for fresh install + voice-profile re-training. Dragon will need 1-2 enrollment sessions to tune to the new mic acoustics.

---

## Post-migration cleanup

- Return the Dell within the return window (documentation + receipt to the original seller).
- Wipe the Dell SSD before returning (Settings → Reset this PC → full clean).
- Update `reference_computer_layout.md` in memory with any paths that changed.
- Update `reference_hotkeys.md` if any DF hotkeys had to be re-registered (VK codes + modifier strings — see `reference_displayfusion_hotkeys` content in the ahk-functions skill).
- Benchmark `qmd query` on the new hardware, paste numbers into `INSTALL_RTX_2070.md`.

---

## Common gotchas

- **Dragon keeps asking to re-activate every boot.** Likely the clone broke Nuance's hardware fingerprint; contact Nuance support for a one-time seat reset.
- **Natlink fails to load.** Re-register via `natlinkconfig_cli.py` (see WIN11 setup guide). `sys.modules` cache doesn't matter on first boot — fresh Dragon process, fresh Python state.
- **"AHK64.exe not trusted" SmartScreen popup.** Right-click → Properties → Unblock, or add an exception.
- **Caster hot-reload doesn't fire.** The `_caster.py` stdout/stderr tee block may have been clobbered on reinstall. Re-apply the fenced comment block documented in `debugging-guide.md § caster_messages.log`.
- **Win+Ctrl+Alt+1-8 snap doesn't work.** DisplayFusion hotkey registry entries at `HKCU\Software\Binary Fortress Software\DisplayFusion\HotKeys` may need re-assigning. Values are `alt;ctrl;win;<VK>` format — see ahk-functions skill § DisplayFusion hotkey editing.
- **OneDrive re-syncing everything.** Expected — may take hours/days to fully re-download `Important/`. Don't panic; just wait. Check OneDrive status icon in tray.
- **Stream Deck shows wrong profiles or blank buttons.** Close SD fully (taskkill, not --quit — see `design-decisions.md § Stream Deck graceful-quit flush`), then reopen. Reads disk on launch.
