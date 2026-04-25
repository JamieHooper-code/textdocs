# Buying a New PC — Research + Decision

## Context

The Dell ECT1250 has to go — proprietary PSU (no PCIe power connectors), 460W ceiling, motherboard won't accept standard ATX PSU. Can't install the RTX 2070 sitting on the desk. Return before window closes.

Once the new PC is chosen, migration plan is in `MIGRATE_TO_NEW_COMPUTER.md` (clone SSD or fresh install). GPU install plan is in `INSTALL_RTX_2070.md`.

---

## Purchasing philosophy

Built from multiple conversations with Claude + other AI assistants:

1. **Standard ATX mid-tower** — non-proprietary, upgradable for 6-8 years
2. **32GB DDR5** minimum (RAM crisis means this is expensive but non-negotiable)
3. **1-2TB NVMe Gen4** (prefer 2TB)
4. **Strong CPU for dev + AI work** — Ryzen 9800X3D, Core Ultra 7 265K, or equivalent
5. **No discrete GPU preferred** since RTX 2070 is already owned — but GPU-less prebuilts are rare; realistic options include a GPU bundle
6. **Prefer Costco** for the 90-day return + 2-year warranty safety net; Newegg/Best Buy acceptable; direct-from-builder (CPU Solutions, iBUYPOWER) for GPU-less specialty
7. **RAM-crisis context (April 2026):** DDR5 prices 3-4x pre-crisis. Prebuilts locked in at pre-crisis contract rates are better value than DIY right now.
8. **Avoid Alienware/Dell/HP Envy/Lenovo IdeaCentre** — proprietary parts are the exact trap being escaped.

---

## Candidates directly considered

### Top contenders

#### iBUYPOWER Element — Ryzen 9800X3D + RTX 5070 12GB
- **Link:** https://www.costco.com/p/-/ibuypower-element-gaming-pc-desktop-amd-ryzen-7-9800x3d-nvidia-geforce-rtx-5070-12gb-windows-11-home-32gb-ram-2tb-ssd/4000384603
- 32GB DDR5, 2TB NVMe, standard ATX mid-tower, 2-year Costco warranty, 90-day return
- Regular price $1,499.99; Costco coupon cycle has had $400 off (~$1,099-1,299)
- **Status (2026-04):** sold out / no active coupon. May restock in monthly Costco circular cycle.
- **Verdict:** If it restocks at sale price, best-value pick. Shelve the 2070 — the 5070 is strictly better (2x CUDA cores, 12GB VRAM, 5th-gen tensor cores, FP8, newer architecture).

#### CyberPowerPC Gamer Supreme Liquid Cooled — Ryzen 9700X + RX 9060 XT 16GB
- **Link:** https://www.costco.com/p/-/cyberpowerpc-gamer-supreme-liquid-cooled-amd-ryzen-7-9700x-38ghz-amd-radeon-rx-9060-xt-16gb-windows-11/4000395323
- Ryzen 7 9700X (8 cores Zen 5), AMD RX 9060 XT **16GB VRAM**, 32GB DDR5-6000, 2TB NVMe, **850W 80+ Gold PSU**, liquid AIO cooling, Windows 11 Home, keyboard/mouse included, 2-year Costco warranty
- **Price (2026-04):** ~$1,900 online. Costco warehouse price may be lower — verify in-store/app.
- **Expandability:** 1 PCIe x16 (used), 2 PCIe x1 (free), 2 M.2 (1 free), 4 DIMM (2 free). ATX mid-tower (ASUS case dimensions 18.9" × 9.4" × 18.9", ~40 lbs).
- **Strengths:** 16GB GPU unlocks 14B local LLMs, 850W Gold PSU needs no upgrade, liquid cooling = quiet, Costco return/warranty.
- **Tradeoffs:** AMD GPU — CUDA ecosystem gap for some AI tooling (but fine for QMD + Vulkan-based tools). 8-core CPU is solid but fewer cores than 265K or 14700F.
- **Verdict:** At $1,500-1,700 this is the clear winner. At $1,900 it's a judgment call vs CPU Solutions.

#### CPU Solutions Intel Core Ultra 7 265K Business Pro Workstation — GPU-less
- **Link:** https://www.cpusolutions.com/store/pc/Intel-Core-Ultra-7-265K-Business-Pro-Workstation-32GB-DDR5-2TB-SSD-Win-11-Pro-p7717.htm
- Intel Core Ultra 7 265K (20 cores, 8P+12E), 32GB DDR5, 2TB NVMe, **no GPU** (use the 2070), MSI Pro B860 WIFI ATX motherboard, Thermalright Phantom Spirit 120SE air cooler, ASUS Prime AP303 tempered glass ATX mid-tower, 650W Bronze (upgradeable at order to 750W/850W Gold for $50-80)
- **Windows 11 Pro pre-installed** (vs Home on Costco options — ~$60 delta value)
- **1-year warranty + 24-hour burn-in testing**, 30-day return (buyer pays return shipping)
- **Price:** $1,747.99
- **Strengths:** matches philosophy exactly (GPU-less, full ATX, named components, no proprietary anything). 20-core CPU is best-in-class for parallel dev work. Windows Pro. Burn-in testing is a real quality signal.
- **Tradeoffs:** 30-day return is tight vs Costco's 90. Smaller builder = no physical return to a store. 2070's 8GB VRAM caps local AI at 7B models.
- **Verdict:** At $1,748, this is the right pick if local AI ambitions are nice-to-have rather than a real commitment. Budget the PSU upgrade to 750W Gold.

### Runners-up / fallbacks

#### iBUYPOWER Slate (Best Buy) — 9800X3D + RX 9070 XT 16GB
- **Link:** https://www.bestbuy.com/product/ibuypower-slate-gaming-desktop-pc-amd-ryzen-7-9800x3d-amd-radeon-rx-9070xt-16gb-32gb-ddr5-rgb2tb-nvme-ssd-black/J3R75JYGZ5
- Ryzen 9800X3D, RX 9070 XT 16GB (better than 9060 XT), 32GB DDR5, 2TB NVMe
- Best Buy 15-45 day return (weaker than Costco's 90 days)
- **Verdict:** Strong alternative if Costco Gamer Supreme isn't available. 9070 XT is a meaningful GPU step up from 9060 XT.

#### iBUYPOWER Element Pro (Best Buy) — 9800X3D + RTX 5070 Ti 16GB
- **Link:** https://www.bestbuy.com/product/ibuypower-element-pro-gaming-desktop-pc-amd-ryzen-7-9800x3d-nvidia-geforce-rtx-5070ti-16gb-32gb-ddr5-rgb2tb-ssd-white/J3R75JY8G3
- Ryzen 9800X3D, RTX 5070 Ti **16GB NVIDIA**, 32GB DDR5, 2TB NVMe
- **Verdict:** NVIDIA 16GB is the gold standard for local AI — best CUDA ecosystem support. ~$1,900-2,200. If budget allows and local AI is a real priority, this is the capability ceiling among considered options.

### Rejected

#### STORMCRAFT SIRIUS (Newegg) — i7-14700F + RTX 5060 Ti 16GB — $1,579
- 16GB NVIDIA GPU is attractive, but:
- **Micro-ATX motherboard + case** — fails the "full ATX upgradability" philosophy
- **"Components brands may vary"** disclaimer — can't verify actual PSU/case/cooler brands before buying
- Unnamed "tower cooler fan" on a hot 14700F is a thermal risk
- 14th-gen Intel degradation history (patched but reputational drag)
- **Verdict:** GPU is tempting, rest of build undermines priorities. Skip.

#### Alienware Aurora (Newegg/Dell) — Core Ultra 7 265F + RTX 5060 Ti
- Alienware = Dell. Aurora line has proprietary components historically. Newer R16/R17 moved toward more standard parts but not fully standard.
- **Do NOT buy.** Exact trap being escaped.

#### iBUYPOWER Y40 Liquid Cooled (Costco) — Ryzen 9700X + RTX 5070
- **Link:** https://www.costco.com/p/-/ibuypower-y40-liquid-cooled-gaming-pc-amd-ryzen-7-9700x-geforce-rtx-5070-windows-11-32-gb-ram-2tb-ssd-black/4000366690
- Tom's Hardware review of Y40 PRO (same chassis): "limited expandability... better suited for style-focused users than serious upgraders."
- Y40 chassis works against the "upgrade for years" intent.
- **Verdict:** Chassis is wrong for this use case. Skip.

#### iBUYPOWER Element Intel 265F (Costco) — Core Ultra 7 265F + RTX 5060 8GB
- **Link:** https://www.costco.com/p/-/ibuypower-element-gaming-pc-intel-core-ultra-7-265f-20-core-processor-nvidia-geforce-rtx-5060-8gb-32-gb-ram-2tb-ssd-white-windows-11-home/4000409185
- RTX 5060 8GB is a lateral move from the RTX 2070 (same VRAM, ~30% more compute). Not worth the premium.
- 265F has no integrated graphics (F suffix). If dGPU fails = no display.
- **Verdict:** Weak value. Skip.

#### CyberPowerPC Gamer Xtreme (Costco) — Ultra 5 225F + RTX 5060 — $1,099
- **Link:** https://www.costco.com/p/-/cyberpowerpc-gamer-xtreme-gaming-desktop-intel-core-ultra-5-225f-geforce-rtx-5060-32gb-ram-1tb-ssd-windows-11/4000445406
- Budget option. 225F is entry-level 10-core (6P+4E) Arrow Lake. 5060 8GB same as above.
- Tom's Hardware flagged cooling as loud.
- **Verdict:** Cheap but dead-end for stated priorities.

#### CPU Solutions Barebones AMD 9800X3D — $1,408 (+$140 Windows = $1,548)
- **Link:** https://www.cpusolutions.com/store/pc/Custom-AMD-Ryzen-7-9800X3D-Barebones-PC-8-Core-16-Threads-5-2-GHz-Max-Boost-1000GB-NVMe-SSD-32GB-DDR5-RAM-p7554.htm
- **Micro-ATX motherboard** (fails ATX upgradability philosophy)
- Only 1TB SSD, no OS, 30-day warranty, 650W Bronze PSU
- **Verdict:** Too many compromises vs the Intel workstation at $200 more. Skip.

#### Skytech Nebula (Newegg) — Ryzen 7700 + RTX 5060 Ti 8GB — $1,499
- Old-gen Zen 4 CPU, 8GB GPU, only 1TB SSD, 650W PSU. No dimension wins. Skip.

#### MSI Codex R2 (Newegg) — i5-14400F + RTX 5060 — $1,339
- Entry-level 14th-gen Intel, 8GB GPU. Weak value. Skip.

#### MSI Aegis Z2 (Newegg) — Ryzen 8700F + RTX 5070 — $1,699
- Older Zen 4 CPU (8700F), 5070 12GB, only 1TB SSD. Runner-up but weaker than top picks. Skip unless others unavailable.

#### Skytech higher-tier (Costco) — Legacy 4 / Chronos 3 with RTX 5080/5090
- Overkill for stated workload. 5080/5090 is paying 2x for GPU capability not needed. Skip.

---

## Decision framework

### If the iBUYPOWER Element 9800X3D + RTX 5070 restocks at $1,099-1,299 Costco coupon
**Buy it.** Clear winner. Shelve/sell the 2070. Best value hands-down.

### If Costco Gamer Supreme Liquid is ≤ $1,700
**Buy the Supreme.** 16GB GPU unlocks serious local AI, 850W Gold PSU is the best in class, liquid cooling is quieter, Costco's return/warranty is the best safety net. Sell/shelve the 2070.

### If Costco Gamer Supreme Liquid is $1,900 (confirmed online price)
**Judgment call between Supreme and CPU Solutions Intel Workstation.** The $152 premium for Supreme buys a 16GB GPU but costs 12 CPU cores and Windows Pro. For daily dev work, CPU cores matter more than GPU capability. For speculative local AI, 16GB unlocks real capability. Depends which use case is real for you.

### If no good Costco option is available AND Dell return window is closing
**Buy CPU Solutions Intel Workstation at $1,748 with 750W Gold PSU upgrade.** Full ATX, named components, 20-core CPU, Windows 11 Pro, 24hr burn-in, 1-year warranty. Genuinely matches the philosophy. Use the 2070 that's been sitting on the desk.

### If local AI is the biggest priority
**iBUYPOWER Element Pro (Best Buy) — 9800X3D + RTX 5070 Ti 16GB** at $1,900-2,200. NVIDIA 16GB = best CUDA ecosystem. Sell the 2070.

---

## Pre-purchase verification (any option)

Before clicking buy, confirm on product page or by asking:

- [ ] Motherboard form factor: **ATX or Micro-ATX** — target ATX, avoid mATX unless deliberate
- [ ] PSU: **standard ATX form factor**, 80+ Gold preferred, ≥750W for GPU headroom
- [ ] PSU has: standard 24-pin ATX + 8-pin CPU + PCIe 8-pin/12VHPWR for GPU
- [ ] Case: **mid-tower or full tower** (not "slim," "SFF," or "compact")
- [ ] Free PCIe slots for future add-in cards
- [ ] Multiple M.2 slots for future SSD upgrades
- [ ] YouTube-search "[exact model] teardown" — verify standard parts inside

For CPU Solutions specifically, email sales before ordering:
- Exact motherboard SKU (MSI Pro B860 **WIFI** vs B860M-A WIFI vs B860 MAX WIFI — want full ATX)
- Confirm 750W or 850W Gold PSU upgrade is available at order time
- Confirm return window starts on delivery, not ship date
- Get 1-year warranty terms in writing

---

## Sell or shelve the RTX 2070

If bought prebuilt includes a GPU equal-or-better than 2070 (any RTX 5060/5070/5070 Ti, RX 9060 XT/9070 XT): **sell the 2070 on r/hardwareswap** for $150-200 or shelve for 6 months then sell. The 2070 is a 2018 card; value depreciates.

Only install the 2070 if bought prebuilt is GPU-less (CPU Solutions) or if keeping as secondary compute card for dual-GPU local LLM work (niche; only if deeply committed to local AI).

---

## After purchase

1. Run `MIGRATE_TO_NEW_COMPUTER.md` Phase 1 prep (Windows MS-account link, Dragon activation check, BitLocker key, E: backups, git push)
2. When machine arrives: clone SSD from Dell via USB-to-M.2 enclosure before booting new machine
3. Run Phase 4 verification checklist against new machine
4. If clone succeeds: return Dell within window
5. If GPU-less (CPU Solutions) or planning to use 2070 specifically: run `INSTALL_RTX_2070.md` after migration verifies
6. Update `reference_computer_layout.md` in memory with any changed paths

---

## Timeline considerations

- **Dell return window is the hard deadline.** Don't return Dell until new machine is verified working.
- **Costco coupon cycles monthly** — if the 9800X3D + 5070 Element is sold out, check back in 2-4 weeks.
- **DDR5 prices expected to stay high until late 2026 / early 2027** per industry analysts. No price relief coming soon; don't wait hoping for cheaper RAM.
- **RTX 5070 / 5070 Ti are current-gen (2025 release)** — won't be superseded until 2027-2028. Buying now is not buying the tail end of a generation.
