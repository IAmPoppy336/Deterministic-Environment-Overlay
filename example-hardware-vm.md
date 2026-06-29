# MSI Aegis ZS2 + Ryzen 9 9900X + RTX 5080 — Hardware Virtual Machine

**Purpose**: A precise, text-only, overlayable model of a target machine (concrete example: MSI Aegis ZS2 with Ryzen 9 9900X + RTX 5080 16 GB).  

Use this together with a matching OS VM and deterministic overlay spec to simulate real-world behavior of local AI inference, image/video generation workloads, agent action loops, file I/O, thermals, VRAM pressure, and power limits **without ever running code on real hardware**.

**Date**: 2026-06-29  
**Source**: Aggregated from official AMD, NVIDIA, MSI specifications + consistent retailer configs (Amazon, Micro Center, MSI site) for the common Aegis ZS2 configuration with Ryzen 9 9900X + RTX 5080.

---

## 1. Physical System Snapshot (Target Machine Example)

| Component              | Specification                                      | Notes for Simulation |
|------------------------|----------------------------------------------------|----------------------|
| **Model**             | MSI Aegis ZS2 (AEZS2C9NVV-1277US or similar)      | Prebuilt, upgrade-friendly B650 platform |
| **CPU**               | AMD Ryzen 9 9900X (Zen 5, Granite Ridge)          | 12 cores / 24 threads, unlocked |
| **GPU**               | NVIDIA GeForce RTX 5080 16 GB GDDR7               | Blackwell (GB203), ~360 W TGP, up to ~2617 MHz boost, ~960 GB/s GDDR7 bandwidth |
| **RAM**               | 32 GB or 64 GB DDR5-6000 (2×32 GB)                | Dual-channel, typical config |
| **Storage**           | 2 TB NVMe PCIe Gen4 SSD                           | Fast sequential + random I/O |
| **Cooling**           | 360 mm AIO Liquid Cooler                          | Good sustained loads; can handle 170W+ PPT |
| **PSU**               | 850W 80+ Gold                                     | Headroom for 120W CPU + 360W GPU + spikes |
| **Motherboard**       | AMD B650 chipset                                  | Good VRM for 9900X |
| **OS (stock)**        | Windows 11 Home / Pro                             | See Win11 VM |
| **Networking**        | Wi-Fi 7 + 2.5 GbE LAN                             | Low latency for downloads |
| **Form Factor**       | Mid-tower, good airflow                           | Positive pressure typical in Aegis series |

**Typical sustained power draw under heavy AI load** (realistic ranges on this chassis with good airflow):
- Idle / light desktop: 80–120 W system
- llama.cpp-style inference (70B Q4/Q5, mostly GPU offloaded): 320–480 W total
- Heavy local video/image generation (high VRAM models, sustained): 420–620 W (GPU dominant)
- Mixed agent loop (vision inference + capture + actions): 280–480 W average

Note: 360 mm AIO + case airflow keeps the 9900X well under thermal limits; the RTX 5080 typically hits its ~360 W power limit before thermal throttling in this prebuilt.

---

## 2. CPU Model — Ryzen 9 9900X (Zen 5)

**Core specs**:
- Cores: 12 (2 CCDs × 6 cores)
- Threads: 24 (SMT enabled)
- Base Clock: 4.4 GHz
- Max Boost: 5.6 GHz (single-core)
- L1 Cache: 960 KB total
- L2 Cache: 12 MB total (1 MB per core)
- L3 Cache: 64 MB (shared across CCDs, very important for large context)
- TDP: 120 W
- PPT (Package Power Tracking): ~170 W (sustained limit on most motherboards)
- Process: TSMC 4nm (cores) + 6nm I/O die
- Memory Controller: Dual-channel DDR5-5600+ (official), excellent real-world scaling to 6000-6400 MT/s
- PCIe: 28 lanes Gen5 from CPU (typically 16 for GPU + 4+ for NVMe)

**AI / inference relevant behaviors**:
- Excellent IPC uplift vs Zen 4 (~16% average) + improved AVX-512 datapath for AI workloads.
- Strong multi-threaded performance for prompt evaluation, batching, and embedding models.
- 64 MB shared L3 cache provides meaningful benefit for large-context KV cache when some layers remain on CPU.
- PPT limit is typically ~162–170 W on B650 boards. The 360 mm AIO handles sustained loads well.
- Under sustained multi-threaded AI workloads with good cooling: realistic all-core boost ~4.8–5.2 GHz before PPT or thermal limits. Package temps usually stay <85 °C.

**Simulation overlay rule**:
When simulating inference:
- If >80% of layers offloaded to GPU → CPU mostly handles prompt eval + sampling (light load).
- If many layers on CPU (low VRAM situation) → expect high utilization across 20+ threads and noticeable power/heat.

---

## 3. GPU Model — RTX 5080 16 GB GDDR7 (Blackwell)

**Key specs**:
- Architecture: Blackwell (GB203)
- CUDA Cores: ~10,752
- Memory: 16 GB GDDR7 on 256-bit bus
- Memory Bandwidth: ~896–960 GB/s (very high)
- Boost Clock: ~2.617 GHz
- TGP / TDP: 360 W
- Power Connectors: 3× 8-pin (or 12V-2x6 adapter)
- Outputs: 1× HDMI 2.1b + 3× DisplayPort 2.1b
- Features: DLSS 4, advanced tensor cores, better FP8/INT8 performance than Ada

**VRAM reality for local AI (critical)**:
- **16 GB GDDR7** is the hard limit.
- llama.cpp with Q4_K_M or Q5_K_M:
  - 70B class model: comfortably fits with room for context (128k+ tokens possible).
  - 120B+ models: tight or requires heavy offloading / lower quant.
  - ComfyUI video (Wan 2.1 14B, SD3, etc.): often needs 12–15+ GB depending on resolution, steps, and motion modules.
- High bandwidth helps a lot with KV cache movement and tensor operations.

**Thermal / Power behavior in Aegis ZS2**:
- Reference ~360 W TGP. The card sustains high boost clocks well thanks to the chassis airflow and 850 W Gold PSU headroom.
- Under sustained heavy inference/video generation: expect GPU temperatures in the 70–85 °C range (junction/hotspot higher but within spec). The card typically hits the power limit before thermal throttling.
- High GDDR7 bandwidth (~960 GB/s) helps maintain good tokens/sec even with partial layer offload to system RAM.

**Simulation overlay rules**:
- When modeling `llama-server` or ComfyUI:
  - Track current VRAM usage. If approaching 14–15 GB → expect swapping / OOM risk or forced layer offload to system RAM (much slower).
  - High VRAM bandwidth means good tokens/sec even with partial offload.
- For agent loop (screenshot + inference): GPU is mostly idle during capture/execute phases; spikes during `decide()` call.

---

## 4. Memory & Storage Model

**RAM (DDR5-6000, dual channel)**:
- Typical: 32 GB or 64 GB (2×32 GB kits common).
- Bandwidth: Very high (~90+ GB/s real-world).
- For llama.cpp: Excellent for offloading layers or running embedding models alongside.
- Simulation: When VRAM is exhausted, system RAM becomes the next tier (still usable but 5–10× slower than GDDR7 for inference).

**Storage (2 TB NVMe Gen4)**:
- Fast enough that model loading from disk is rarely the bottleneck once models are cached in RAM/VRAM.
- Good for project workspaces, history.jsonl, memory.md, and large ComfyUI workflows.

---

## 5. Thermal & Power Envelope (Sustained AI Workloads)

| Workload                          | Approx. System Power | CPU Temp | GPU Temp | Notes |
|-----------------------------------|----------------------|----------|----------|-------|
| Idle / light desktop              | 80–120 W            | <50°C   | <45°C   | — |
| llama.cpp server (moderate load)  | 250–400 W           | 65–80°C | 60–75°C | Good headroom |
| Heavy ComfyUI video gen           | 450–650 W+          | 75–88°C | 75–85°C | 360mm AIO + case fans handle it |
| Full agent loop (vision + tools)  | 300–500 W           | 70–82°C | spikes  | Mostly GPU during inference |

**Key behaviors**:
- 360mm AIO + Aegis airflow keeps the 9900X from thermal throttling under normal AI loads.
- RTX 5080 will hit its 360W power limit before thermal limit in this chassis.
- Sustained high load → expect fans to ramp up (audible but not extreme).

---

## 6. How to Overlay This VM

When mentally executing a scenario with this hardware model:

**Example**: "Run a 12-move agent task with Qwen2.5-VL-7B on this hardware"

1. **Inference step** (`client.decide`):
   - Model: Qwen2.5-VL-7B + mmproj → ~6–8 GB VRAM typical.
   - On this machine: Fits easily. Expect fast response (low seconds per move).
   - If using 32B+ vision model: Check against 16 GB limit.

2. **Capture step** (`platform.capture`):
   - mss + Pillow resize to 1280 px → negligible load.
   - Multi-monitor: Correctly handled by the Win11 + platform code.

3. **Execute + verify**:
   - pyautogui actions: very low CPU/GPU.
   - Screen diff in `screen_changed`: runs on CPU, fast.

4. **Thermal/power note**:
   - 12 moves × vision inference = short burst. No sustained thermal concern.
   - If running long autonomous loops: watch for cumulative heat in closed case.

**Rule of thumb for simulation**:
- VRAM usage < 12 GB → comfortable headroom; expect good performance.
- VRAM 13–15 GB → tight; anticipate possible forced layer offload to DDR5 (significant slowdown due to ~90 GB/s vs ~960 GB/s GDDR7 bandwidth) or context truncation.
- Sustained CPU load > 70% across many threads → CPU will boost aggressively until ~162–170 W PPT or ~85 °C; 360 mm AIO provides good margin.
- Mixed workload (vision inference + light CPU work): GPU dominates power/thermals; monitor for cumulative heat in longer autonomous loops.

---

## 7. Known Quirks & Limitations for Accurate Simulation

- **VRAM is the primary constraint** for large models (not raw compute).
- The 360mm AIO is strong but the case has finite airflow — sustained 600W+ loads will eventually ramp fans and raise ambient inside chassis.
- PCIe Gen5 GPU slot: full x16 available; no bandwidth starvation for RTX 5080.
- DDR5-6000 is sweet spot; higher speeds give diminishing returns for most local AI inference workloads.
- Windows 11 power plan (Balanced or High Performance) affects boost behavior — assume "High Performance" or Ultimate Performance for AI work.

---

**This hardware VM is now ready to be overlaid on any scenario.**

It pairs with the matching OS VM and Deterministic Overlay Spec to form a complete, high-fidelity language-based simulation environment for this class of hardware.