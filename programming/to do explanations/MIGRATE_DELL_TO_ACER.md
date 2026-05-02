# Migrate Dell ECT1250 → Acer Nitro 60

> **Active migration plan, 2026-05-01.** Supersedes `MIGRATE_TO_NEW_COMPUTER.md` (which still has Dragon 15→16, Nate→Jamie, and RTX 2070-install context that no longer applies) and `WIN11_SETUP_GUIDE.md` (deprecated entirely — that was the previous transfer).

## Why this exists

Returning the Dell ECT1250 (proprietary chassis) and replacing with an Acer Nitro 60 N60-640-UR26 (refurb, eBay). Goal: clone the Dell SSD onto the Acer with minimum disruption to the Dragon/Caster/AHK/Stream Deck stack.

Already done before this doc starts:
- Dragon 16 is installed and working (the v15→v16 upgrade described in `WIN11_SETUP_GUIDE.md` has been completed)
- Username is already `Jamie` (the Nate→Jamie flip described in old docs has been completed on the Dell)
- Caster `utilities.py` natlink patch is in place
- Most repos are tracked in git

## Target hardware — Acer Nitro 60 N60-640-UR26

| Component | Spec | Notes |
|---|---|---|
| Motherboard | Gigabyte B760M (mATX, Intel B760) | Standard, NOT proprietary Acer |
| CPU | Intel Core i7-14700F (20-core, 8P+12E) | Same Intel platform as Dell — eases driver swap |
| RAM | 32GB DDR5 | 4 DIMM slots, 2 free |
| Primary storage | 2TB PCIe 4.0 NVMe SSD | Stock — becomes the clone target |
| GPU | NVIDIA RTX 5060 Ti 16GB GDDR7 | Strictly better than the RTX 2070; **shelve/sell the 2070** |
| PSU | Thermaltake 850W modular | Standard ATX, plenty of headroom |
| M.2 slots | 2-3 (PCIe 4.0) | Confirmed ≥2 — enables Plan 2 |
| SATA bays | 2× 2.5" | For future expansion |
| Case | mATX mid-tower (15.9 × 14.9 × 8.5") | Smaller than ideal but acceptable |
| Warranty | 1-year (refurb) | Tighter than Costco's 90-day; verify eBay seller's return window on arrival |

This is meaningfully better than the Dell on every axis except the mATX form factor. The 5060 Ti 16GB unlocks 14B local LLMs without needing the 2070.

---

## Two viable paths — pick one before starting

Both are clone-based. Both end with the Dell SSD's contents booting on the Acer. The difference is *where* the cloning runs.

### Path 1 — USB enclosure, clone Dell-side (recommended default)

Acer's stock SSD goes into a USB enclosure as the clone target while the Acer is still in the box. Dell does the cloning. You only ever boot the Acer with a finished cloned drive in it.

**Pros:** Acer's stock SSD stays pristine until you're done — easy revert if anything goes sideways. Dell is the source of truth throughout. Minimal handling of the Acer (open once to install the cloned drive). Battle-tested workflow, matches the original `MIGRATE_TO_NEW_COMPUTER.md` Phase 2.

**Cons:** Requires a $25 USB 3.2 Gen 2 NVMe enclosure (1-2 day shipping). Clone runs at USB 3.2 speeds (~1 GB/s) instead of native PCIe 4.0 (~7 GB/s) — 20-40 min for a 1-2 TB drive instead of ~5 min.

### Path 2 — In-place clone using both M.2 slots on the Acer

Boot the Acer with its factory Windows install long enough to install Macrium. Power down. Drop the Dell SSD into slot 2. Clone slot 2 → slot 1 at native PCIe 4.0 speeds. Reboot, switch boot order, you're on the cloned drive. Wipe slot 2 for scratch.

**Pros:** No enclosure purchase. Clone runs at native PCIe 4.0 — significantly faster. Both drives live in the new box from the start; never ship the Dell SSD around in an enclosure. Slot 2 ends up as ready-to-use scratch space.

**Cons:** Requires booting the Acer's factory Windows once (consumes the OOBE you'd otherwise skip). The clone overwrites the pristine factory state — if anything goes wrong mid-clone, no easy revert (vs. Path 1 where the original SSD is still untouched in the enclosure). Slightly more steps and more things to get right in BIOS.

### Recommendation

**Path 1 unless you have the enclosure delay-aversion or want max clone speed.** Path 2 is technically nicer if everything goes right; Path 1 is more forgiving if anything goes wrong. Given this is a one-shot migration with a return-window deadline, Path 1's "Acer stock SSD is preserved as a fallback" property is worth more than Path 2's speed.

---

## Phase 1 — Prep on the Dell (do BEFORE the Acer arrives)

### 1. Push everything to git
Run `GitPushAll.ps1`. All tracked repos to origin. The clone might fail; git is the survivable fallback.

Specifically verify these are pushed:
- `C:\Users\jamie\Desktop\Important\AutoHotkey\`
- `C:\Users\jamie\AppData\Local\caster\rules\`
- `C:\Users\jamie\AppData\Roaming\Elgato\StreamDeck\ProfilesV3\`
- `C:\Users\jamie\.claude\`
- Project repos under `Desktop\Important\projects\`
- `C:\Users\jamie\Desktop\Important\TEXTDOCS\` (this doc itself)

### 2. Verify E: backups are current
- `Scripts/CopyPasteSlotCrypt/copy_paste_slot_crypt.key` — without this every `.enc` slot becomes unreadable
- `.atlantic_graph_certs/` — McLeod API key, Atlantic service principal `.env`, `.pfx` cert
- Dragon 16 user profile folder at `%LOCALAPPDATA%\Nuance\NS16\Users\` — copy to E: as belt-and-suspenders even though the clone will carry it

### 3. Link Windows activation to Microsoft account
Settings → Accounts → Your info → "Sign in with a Microsoft account instead." Then Settings → System → Activation should read **"Windows is activated with a digital license linked to your Microsoft account."** If it doesn't, fall back to using the Acer's bundled Win 11 Home license post-clone (run `slmgr /ipk <Acer key>` then `slmgr /ato`).

### 4. Install Macrium Reflect Free on the Dell
Download from `macrium.com/reflectfree`. (Or use Clonezilla if you prefer; Macrium is the easier GUI path for this scenario.)

### 5. Buy the USB-to-M.2 NVMe enclosure (Path 1 only)
USB 3.2 Gen 2 (10 Gbps) or better — Sabrent EC-SNVE, Orico M2PJM-C3, anything reputable. ~$20-30. Skip if doing Path 2.

### 6. Confirm eBay seller's return window
Whatever it is (probably 30 days), that's your hard deadline for getting the Acer verified working before the Dell return becomes irrevocable.

---

## Phase 2 — Clone

### Path 1 — USB enclosure clone

1. Acer arrives. **Do not boot it.** Power off, plug nothing in.
2. Open the Acer case, locate slot 1 (primary M.2 with the stock 2TB drive). Remove that drive.
3. Put the Acer's stock SSD into the USB enclosure. Plug enclosure into the Dell.
4. On the Dell: open Macrium Reflect → "Clone this disk" → source = Dell C:, target = the enclosure drive. Verify Macrium identifies it correctly (check size matches: 2TB).
5. Run the clone. ~20-40 min on USB 3.2 Gen 2.
6. Verify Macrium reports clean completion. Eject the enclosure.
7. Pull the cloned SSD from the enclosure. Install it in the Acer's slot 1 (where the stock drive came from).
8. Close the case. Proceed to Phase 3.

### Path 2 — In-place clone

1. Acer arrives. Power on, complete OOBE on the factory Windows install. Wi-Fi, MS account sign-in, the usual.
2. Install Macrium Reflect Free on the Acer.
3. Power off the Acer.
4. Open the case. Locate slot 2 (free M.2 slot). Install the Dell's SSD there.
5. Power on the Acer. Windows boots from slot 1 (the factory install), sees the Dell SSD as a secondary data drive in slot 2.
6. Open Macrium → clone → source = the slot-2 drive (the Dell content), target = slot 1 (the Acer factory install). **Double-check source/target before clicking go** — this overwrites the Acer's factory drive.
7. Run the clone. ~5-10 min at PCIe 4.0 speeds.
8. Reboot. Enter BIOS (mash F2 / Del during POST). Set boot order so slot 1 is first. Save & exit.
9. Boot — should land on the cloned (Dell-source) Windows.
10. Once verified, you can wipe slot 2 (Settings → Storage → format the secondary drive) for scratch space.

---

## Phase 3 — First boot on Acer hardware (both paths)

Expect 5-15 minutes of Windows discovering hardware and rebooting. This is normal.

### What to expect
1. **First boot** — Windows detects new motherboard/GPU/network/audio, installs generic drivers, reboots automatically.
2. **Second boot** — likely logs in. Resolution may be wrong. Device Manager will show ⚠️ on a few unknown devices. Network may default to ethernet (Wi-Fi card is on the new motherboard).
3. **Activation check** — Settings → Activation. Should reactivate via MS account link. If "Not activated," run the troubleshooter, or fall back to the Acer's bundled key (sticker on the case or in the Acer documentation).

### Driver cleanup — uninstall Dell-specific software
Control Panel → Programs and Features → uninstall:
- Dell SupportAssist
- Dell Update
- Dell Command | Update
- Dell Mobile Connect
- Dell Digital Delivery
- Dell Display Manager
- Dell Optimizer
- Anything else Dell-branded

These conflict with non-Dell hardware and can cause weird boot issues.

### Install Acer / Gigabyte / NVIDIA drivers
1. **Gigabyte B760 chipset driver** — download from Gigabyte's support page for the B760M variant in this box (or use Acer's Nitro 60 driver bundle if available). Replaces the generic Intel drivers Windows installed.
2. **NVIDIA Studio driver** for the RTX 5060 Ti — get the latest from `nvidia.com/drivers`. Custom install, uncheck GeForce Experience if you don't want the overlay. Reboot.
3. **Realtek audio driver** if Device Manager still shows ⚠️ on audio.
4. Verify: `nvidia-smi` should now list the 5060 Ti 16GB. Device Manager should be clean.

### Skip
- The RTX 2070 install — the 5060 Ti 16GB is strictly better. Sell the 2070 on r/hardwareswap or shelve.
- Anything from `INSTALL_RTX_2070.md` — that whole doc is moot.

---

## Phase 4 — Verify the stack

Top-to-bottom. If anything fails, note before proceeding — issues compound.

### Core OS / hardware
- [ ] `nvidia-smi` lists RTX 5060 Ti, driver version, 16GB VRAM
- [ ] Display resolution + refresh rate correct on all monitors
- [ ] Audio routing (speakers, mic, VoiceMeeter if used) works
- [ ] USB devices enumerate (keyboard, mouse, both Stream Decks, foot pedals, Arduino, Tobii, mic, Q0 Max numpad)
- [ ] Network (Wi-Fi or ethernet)
- [ ] Windows activation green

### Dev toolchain
- [ ] `py --version` returns Python 3.10.x; `py -3.10-32 --version` returns the 32-bit Python 3.10 (Natlink dependency)
- [ ] `git --version` works; `git config --global user.name` is correct
- [ ] VS Code opens, signed in, Claude Code extension loads
- [ ] `claude --version` works
- [ ] `qmd status` works

### AHK
- [ ] `C:\Users\jamie\AppData\Local\Programs\AutoHotkey\v2\AutoHotkey64.exe` exists
- [ ] `MAINFUN.bat` on PATH (`where MAINFUN.bat`)
- [ ] `MAINFUN.bat CheckProcessRunning chrome.exe` returns clean
- [ ] `MAINFUN.bat ReloadWithNotice` fires

### Dragon / Natlink / Caster
- [ ] Dragon 16 starts; voice profile recognized (carried over in clone)
- [ ] Natlink loads on Dragon start (tray icon, no error dialog)
- [ ] Caster loads (`caster_messages.log` shows startup banner)
- [ ] Say `"<domain> Commands"` for a few rules — debug probes fire
- [ ] Say `"reboot caster"` — restarts cleanly (confirms `utilities.py` patch survived)
- [ ] Say an AHK-dispatched voice command — `ahk_event.log` shows `DISPATCH/in` + `DISPATCH/ok`

### Stream Deck
- [ ] Stream Deck app opens, sees Deck A and Deck B
- [ ] Profiles load from `ProfilesV3/`
- [ ] Press a button — fires via `MAINFUN.bat`

### Project-specific
- [ ] DisplayFusion runs, license active, Win+Ctrl+Alt+1-8 snap hotkeys work
- [ ] Everything (voidtools) running; `verify.ps1 search "notepad.exe"` returns
- [ ] OneDrive signed in, `Important/` syncing
- [ ] Chrome extensions + bookmarks synced
- [ ] VS Code settings synced

### Voice + AHK end-to-end smoke test
- [ ] `"reboot caster"` → works
- [ ] `"show VS code commands"` → browser GUI on leftmost monitor
- [ ] `"swallow test"` → copies selection to slot
- [ ] `"spit test"` → pastes slot
- [ ] A voice command that dispatches to AHK to open an app

---

## Phase 5 — Post-migration cleanup

- Return the Dell within the eBay seller's window (only after Phase 4 is fully green)
- Wipe the Dell SSD before returning (Settings → Reset this PC → full clean)
- Update `reference_computer_layout.md` in memory with any paths that changed (PSU upgrade not needed — Acer's 850W is already plenty)
- List the RTX 2070 on r/hardwareswap ($150-200 expected)
- Optionally drop the Dell's now-wiped SSD into Acer slot 2 as scratch (Path 1 only — Path 2 already wiped slot 2)

---

## Plan B — Clone failed, fall back to fresh install

If the cloned drive won't boot, BSODs repeatedly, or behaves so badly that fixing it costs more than reinstalling:

1. Pull the cloned SSD. Reinstall Acer's stock SSD with its factory Windows (Path 1) or wipe + clean-install (Path 2).
2. Walk through `WIN11_SETUP_GUIDE.md` **selectively** — most of it is now historical (Dragon 15→16, Nate→Jamie), but the app inventory and backup-restore locations in Phase 5 of that doc are still useful as a checklist.
3. `git clone` everything back from `JamieHooper-code` org.
4. Restore keyfile from E: to `Scripts/CopyPasteSlotCrypt/`.
5. Restore Dragon profile from E: backup if needed.
6. Re-apply the Caster `utilities.py` patch (block in `WIN11_SETUP_GUIDE.md §2.1 Step 8` — that section is still correct).
7. Phase 4 checklist to verify.

Budget: one weekend. Better than redoing months of voice-profile tuning, but worth avoiding if the clone path works.

---

## Common gotchas

- **AHK64.exe SmartScreen popup** — right-click → Properties → Unblock, or add an exception. First-run-only.
- **OneDrive resync** — expected. Hours-to-days for full `Important/` re-download. Just wait.
- **Stream Deck shows wrong/blank profiles on first launch** — fully kill via taskkill (not the GUI Quit), wait 5 seconds, reopen. SD reads `ProfilesV3/` from disk on launch.
- **DisplayFusion Win+Ctrl+Alt+1-8 snap doesn't fire** — registry hotkeys at `HKCU\Software\Binary Fortress Software\DisplayFusion\HotKeys` may need re-registering. Format: `alt;ctrl;win;<VK>`. See ahk-functions skill § DisplayFusion hotkey editing.
- **`MAINFUN.bat` not on PATH after clone** — verify `C:\Users\jamie\Desktop\Important\AutoHotkey\Scripts\` (or wherever it lives) is in user PATH. Add via System Properties → Environment Variables → User PATH if missing.
- **Caster hot-reload doesn't fire** — the `_caster.py` stdout/stderr tee block can get clobbered. Re-apply the fenced comment block per `debugging-guide.md § caster_messages.log`.
