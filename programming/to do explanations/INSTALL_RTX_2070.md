# Install the RTX 2070 — Checklist

## Why this exists

The RTX 2070 has been sitting on the desk uninstalled. Current machine runs Intel integrated graphics (Vulkan, shared system memory, ~50 GB/s bandwidth, no tensor cores). This is fine for general use but throttles any local-LLM workload — currently QMD (local markdown search with GGUF models) pays ~60s on a cold `qmd query` because the 0.6B reranker stage is bandwidth- and compute-starved.

The 2070 has 8 GB GDDR6, 448 GB/s bandwidth, 288 Turing tensor cores. For sub-2B-parameter GGUF quants (which is everything QMD runs, and most local-LLM tooling worth using), this is night and day. Projected speedups:

| Query type | Current iGPU | RTX 2070 + daemon warm |
|---|---|---|
| `qmd search` (BM25) | 0.5s | 0.5s (CPU path, unchanged) |
| `qmd vsearch` | 11s | <1s |
| `qmd query` cold | 63s | ~8-12s |
| `qmd query` warm daemon | ~15-20s (projected) | ~2-4s |

More importantly: this unlocks other local-LLM tooling (Ollama, LM Studio, finetuning small models, etc.) that's currently impractical. It's a general-purpose upgrade, not just a QMD speedup.

---

## Pre-install check

Do these BEFORE opening the case. If any fail, don't start the install until resolved.

1. **PSU wattage** — RTX 2070 TDP is 175W. Whole-system target ≥ 550W PSU. If PSU < 550W, budget a PSU upgrade first.
2. **PSU PCIe connectors** — 2070 needs one 8-pin OR one 6+2-pin PCIe power connector. Older/cheaper PSUs sometimes don't have it; confirm by eye before install. Molex-to-PCIe adapters exist as a fallback but are a compromise.
3. **PCIe slot** — any modern motherboard has a PCIe x16 slot for the GPU. Confirm nothing's blocking the slot (M.2 heatsinks, tall RAM, case brackets).
4. **Physical clearance** — the 2070 is a 2-slot card, ~27cm long depending on model. Measure from the rear of the case to the nearest HDD cage / front fan.
5. **Drivers installed for the iGPU currently** — that's fine. Windows handles the transition when a dGPU is added.

---

## Physical install

Standard GPU install. Brief version:

1. Shut down, unplug power cable, press power button to bleed residual current, wait 30s.
2. Open case side panel.
3. Remove the two PCIe slot covers on the rear of the case aligned with the top PCIe x16 slot.
4. Release the motherboard's PCIe retention clip (small lever at the end of the slot — varies by board).
5. Seat the 2070 firmly in the x16 slot until the retention clip snaps.
6. Screw the GPU bracket into the case rear.
7. Plug in the 8-pin (or 6+2) PCIe power from PSU.
8. Close case, plug in power, boot.

On first boot Windows will spin up a generic display driver. Don't panic if the resolution looks wrong — drivers come next.

---

## Software setup (where Claude can help)

Once the card is physically in and Windows boots:

### 1. NVIDIA driver

- Download latest **Studio** driver from nvidia.com (Game Ready is fine too; Studio is recommended for mixed productivity/dev workloads).
- Run installer → custom install → uncheck "GeForce Experience" if you don't want the account-required overlay.
- Reboot.
- Verify: `nvidia-smi` on PowerShell/bash should show the 2070 and driver version.

### 2. CUDA Toolkit

- Download CUDA Toolkit 12.x from developer.nvidia.com/cuda-downloads.
- Default install path: `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v12.X\`.
- Reboot if prompted.
- Verify: `nvcc --version` should print the CUDA version.

**Why CUDA and not just Vulkan:** QMD uses node-llama-cpp which supports both. Vulkan works on NVIDIA hardware but CUDA is 20-40% faster — tensor cores are only used through CUDA.

### 3. Rebuild node-llama-cpp with CUDA

- `npm rebuild @tobilu/qmd` after CUDA toolkit is present. Usually auto-detects.
- If `qmd status` still shows `GPU: vulkan` after the rebuild, force CUDA via env var (check node-llama-cpp docs for current flag — typically `NODE_LLAMA_CPP_BUILD_CUDA=1` during install or a `.npmrc` entry).

### 4. Display routing

- Windows routes apps to dGPU automatically when needed, but for consistency plug your primary monitor into the 2070 directly.
- Settings → System → Display → Graphics → set "High performance" default for Chrome, VS Code, and anything else you care about running on the dGPU.

### 5. QMD daemon (separate task)

Once CUDA is live, set up the QMD HTTP daemon to run at startup:

```
qmd mcp --http --daemon
```

Add to Windows startup via Task Scheduler or a startup-folder shortcut. With warm models and CUDA inference, `qmd query` should drop to ~2-4s.

### 6. Benchmark before/after

Intel iGPU baseline (recorded pre-install):
- `qmd search` — 0.5s
- `qmd vsearch` — 11s
- `qmd query` cold — 63s
- `qmd query` cached (same phrase) — 2.5s
- `qmd query` with warm HTTP daemon — **~68s, no meaningful improvement over cold**. Confirmed with 3 back-to-back novel queries. The Intel iGPU is compute-bound, not load-bound; daemon pays off only on real GPUs.

After CUDA is live, re-run the same queries for a clean before/after. The expected post-GPU numbers:
- `qmd search` — 0.5s (CPU path, unchanged)
- `qmd vsearch` — <1s
- `qmd query` cold — ~8-12s
- `qmd query` with warm daemon — **~2-4s (where the daemon finally matters)**

Paste actual numbers into this file for posterity. This is where the "daemon auto-start via Task Scheduler" step earns its keep — on iGPU it's wasted setup, on GPU it's the difference between "interactive search tool" and "one-minute lookup tax."

---

## Ping Claude when ready

The physical install + driver + CUDA toolkit is human work. Once `nvidia-smi` and `nvcc --version` both return clean output, Claude can handle:
- node-llama-cpp rebuild + verification
- QMD daemon setup + Task Scheduler entry
- Benchmarks + updating this file with the post-install numbers
- Any follow-on local-LLM tooling (Ollama install, model pulls, etc.) if you decide to spread the use case.

Do it in one session — CUDA rebuild + daemon + benchmark are all chained.
