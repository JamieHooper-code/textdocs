---
tags: [hardware, bios, ram, performance, how-to]
related: ["[[WIN11_SETUP_GUIDE]]"]
status: TODO -- found 2026-10-18, not done yet. RAM running at 4800, rated 6000.
created: 2026-10-18
---

# RAM: turn on XMP so it runs at its rated speed

## The problem

The two RAM sticks are rated for **DDR5-6000**, but they're running at
**4800**. That's the plain default speed every DDR5 stick starts at. The 6000
speed is stored on the stick as an **XMP profile**, and the motherboard only
uses it if you switch it on in the BIOS. Nobody ever did, so you're getting
about 20% less memory speed than you paid for.

Nothing is broken or unstable; it's just slower. Everyday stuff (voice, AHK,
browsing) won't feel different. Memory-heavy work gets roughly 5–15% faster
after the fix: local AI models, big media/file jobs, compiling, some games.

## What's in the machine (read 2026-10-18)

| Thing | Value |
|---|---|
| RAM | 2 × 16 GB Team Group `UD5-6000` (32 GB) |
| Slots | DDR5-A2 + DDR5-B2 (correct for dual-channel, leave them) |
| Running at | 4800 MT/s, 1.10 V |
| Should be | 6000 MT/s (XMP profile 1) |
| Board | Gigabyte B760M C V3 |
| CPU | i7-14700F |

## How to fix it (~5 minutes)

1. Restart the PC. As soon as the screen comes on, keep tapping **Delete**
   until the BIOS opens.
2. It opens in **Easy Mode**. Look for **X.M.P.** (Extreme Memory Profile).
   Set it to **Profile 1**.
   - Can't see it? Press **F2** for Advanced Mode → **Tweaker** tab →
     **Extreme Memory Profile (X.M.P.)** → **Profile 1**.
3. Press **F10**, then pick **Yes** to save and exit.
4. The first boot after this can sit on a black screen for up to a minute while
   the board tests the memory. **That's normal — don't turn it off.**

## If it won't boot or acts weird

- Gigabyte boards usually give up after a few failed tries and boot back on the
  safe settings. Then just go back into the BIOS.
- Instead of XMP, set the memory speed by hand to **5600** (the highest speed
  Intel officially supports on this CPU). With two sticks, 6000 almost always
  works, so this is only a fallback.
- If it truly won't come back: power off, unplug, hold the **CLR_CMOS** pins/button
  on the board (or pull the coin battery for a minute). That resets the BIOS.

## Afterwards: check it took

Ask Claude to check, or run:

```powershell
Get-CimInstance Win32_PhysicalMemory | Select DeviceLocator, Speed, ConfiguredClockSpeed, ConfiguredVoltage
```

`ConfiguredClockSpeed` should say **6000**, and the voltage around **1350**.
Then have Claude run a memory stability test to make sure it's solid.
