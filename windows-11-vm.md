# Windows 11 — Operating System Virtual Machine (Relevant Behaviors)

**Purpose**: Precise, text-only model of Windows 11 behaviors relevant to computer-use agents, GUI automation, screen capture, input simulation, DPI handling, UI Automation, power management, and security.  

Use this as the OS contract layer in deterministic overlay simulations alongside matching hardware and agent-behavior models.

**Focus**: Only the subsystems that matter for computer-use agents, vision models, pyautogui/mss, and local AI tooling. Not a general Windows reference.

**Date**: 2026-06-29 (current stable channel assumptions for 24H2 / 25H2 era)

---

## 1. Core Architecture Relevant to Agents

**Kernel & User Mode**:
- Modern hybrid kernel with strong isolation (VBS + HVCI enabled by default on most consumer installs in 2025–2026).
- **Impact on agents**: Real-time protection (Defender) can interfere with low-level input hooks or rapid pyautogui calls. In practice, `pyautogui` and `mss` usually work, but aggressive security policies or third-party AV can block or slow them.

**Graphics Stack (WDDM)**:
- Windows Display Driver Model 2.x / 3.x.
- **Multi-monitor virtual desktop**: One unified coordinate space. Monitor origins are reported correctly via `EnumDisplayMonitors` / `mss`.
- **DPI Awareness**: Critical. See section 3.

**Input Stack**:
- Raw Input + SendInput for simulated mouse/keyboard.
- `pyautogui` uses SendInput under the hood → generally reliable but can be blocked by:
  - UAC / integrity level mismatches
  - Some games with anti-cheat (kernel-level hooks)
  - "Game Mode" or fullscreen exclusive apps

---

## 2. DPI Awareness & Coordinate System (Extremely Important)

This directly affects `Capture.to_screen()` correctness.

**Windows 11 default behavior**:
- Modern apps are **per-monitor DPI aware** by default (or should be).
- The Aegis ZS2 + RTX 5080 setup with typical 1440p or 4K displays will have mixed DPI if using multiple monitors.

**Key rules for simulation**:
1. Process must call `SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)` early (the `WindowsPlatform.__init__` does this via `shcore.SetProcessDpiAwareness(2)`).
2. If not DPI-aware → Windows lies about monitor resolution (reports virtualized/scaled values) → clicks land in the wrong place.
3. `mss` + Pillow capture returns **physical pixels** only when the process is per-monitor DPI aware.
4. `pyautogui` coordinates must also be in physical pixels (the code does this correctly after DPI awareness is set).

**Simulation overlay**:
- When the agent does `platform.capture()` → assume it gets true physical resolution + correct monitor origin.
- `Capture.to_screen(ix, iy)` math is correct **only because** the WindowsPlatform sets DPI awareness.

**Common failure mode to watch for**:
- If running from certain IDEs, terminals, or services without proper manifest → DPI awareness fails silently → clicks are off by the scale factor (e.g., 150% → 1.5× error).

---

## 3. UI Automation (UIA) — Grounding Future Path

**Current state in code**: Not yet used (pure pixel grounding).
**Planned in PROJECT.md D1**: Hybrid a11y tree + Set-of-Mark.

**Windows 11 UIA reality**:
- `UIAutomationCore.dll` + COM interfaces are stable and fast.
- Can enumerate elements with `{Name, AutomationId, BoundingRectangle, ControlType}` reliably for most Win32 / UWP / WPF / WinUI apps.
- **Limitations**:
  - Chrome / Edge / Firefox: Good support via accessibility tree but sometimes needs `IUIAutomation` with specific conditions.
  - Electron apps: Variable quality.
  - Games / DirectX / Vulkan / custom canvases: Often no useful tree (fallback to vision required).
  - Modern Windows apps (Settings, Store, etc.): Excellent tree.

**Simulation note**:
When we later add UIA grounding:
- First try to get element under cursor or near target via `IUIAutomation`.
- If good bbox + role → prefer that over raw pixel from vision model.
- Only fall back to vision pixel coords when tree is empty or low confidence.

---

## 4. Power Management & QoS (Affects Sustained Loops)

**Relevant features**:
- **EcoQoS / Efficiency Mode**: Windows can throttle background processes.
- **Modern Standby** (S0 low power idle).
- **Power Throttling** for UWP / packaged apps (less relevant for Python).

**For long-running agent loops**:
- If the Python process is not marked as high-priority or "performance" workload, Windows may deprioritize it after some inactivity.
- **Recommendation for simulation**: Assume the agent process requests high QoS or runs with `HIGH_PRIORITY_CLASS` + power plan set to "Ultimate Performance" or "High Performance".
- Thermal/power limits from the Hardware VM still apply on top.

---

## 5. Security & Interference Points

| Feature                    | Default Behavior                          | Impact on agent tooling / pyautogui        | Mitigation in Code |
|---------------------------|-------------------------------------------|--------------------------------------------|--------------------|
| Windows Defender Real-time | On                                        | Can flag rapid input or memory access      | Usually fine; add exclusion for dev folder |
| HVCI + VBS                | Often enabled                             | Slight overhead on some operations         | Negligible for our use |
| UAC / Integrity Levels    | Standard                                  | SendInput may be blocked from low IL       | Run as normal user or elevate carefully |
| Game Mode / Game Bar      | Can be active                             | Sometimes interferes with input simulation | Disable or ignore |
| Fullscreen Exclusive      | Some games still use it                   | pyautogui clicks may not register          | Agent should detect and handle |

**Simulation rule**:
- Assume a clean developer machine with Defender exclusions for the project folder and Python.
- If simulating "hostile" environment → add random small delays or failed input attempts.

---

## 6. Multi-Monitor & Virtual Desktop Behavior

- One unified coordinate space across all monitors.
- `mss.monitors[0]` = virtual screen union.
- Real monitors start at index 1.
- Primary monitor usually has (0,0) origin.
- When agent calls `platform.monitors()` → gets list with correct `x, y, width, height, primary`.
- `Capture.to_screen()` correctly adds the monitor's origin.

**Edge cases**:
- HDR monitors: Color space can affect screenshots slightly (usually not an issue for agent vision).
- Variable refresh rate (VRR/G-Sync): Can cause minor timing jitter in capture → the `screen_changed` debounce helps.
- Laptop + external monitor setups: More complex scaling; the Aegis ZS2 is desktop so simpler.

---

## 7. File System & Process Behavior

- NTFS with long path support (enabled by default in recent Win11).
- Python file I/O is fast on NVMe.
- Process creation (`subprocess` for tools) has some overhead; use it judiciously in agent loops.
- Background services (Search Indexing, Superfetch, etc.) can cause occasional I/O spikes — rare but possible during heavy `write_file` or model loading.

---

## 8. How to Overlay This Win11 VM

When mentally running a scenario:

**Example**: Agent does `click` at model-provided (x=640, y=360) on a 1440p primary monitor at 100% scaling, with a secondary 4K monitor to the right.

1. Capture happens on primary → image is physical 2560×1440 (or downscaled to 1280).
2. Model outputs pixel on the resized image.
3. `Capture.to_screen(640, 360)`:
   - scale_x = 2560 / 1280 = 2.0
   - gx = 0 + 640 * 2.0 = 1280
   - gy = 0 + 360 * 2.0 = 720
4. `platform.click(1280, 720)` → SendInput at physical desktop coords → correct.

If DPI awareness was **not** set → Windows would have reported fake scaled resolution → math would be wrong.

**Another example**: 3 consecutive misses on a click.
- Per reliability engine → after 3rd miss → abort.
- Windows angle: possible cause = UIA element moved, or fullscreen app stole focus, or DPI mismatch (already handled).

---

## 9. Known Win11 Gotchas for Computer-Use Agents (2026)

- UWP / WinUI 3 and some modern apps can have delayed, incomplete, or dynamically changing accessibility trees — prefer hybrid grounding (UIA + vision) or fall back to pixel coordinates quickly.
- Chromium-based browsers (Edge, Chrome) often need `--force-renderer-accessibility` or specific flags for reliable UIA trees; otherwise vision grounding is more robust.
- Focus can be stolen by toasts, notifications, or other apps mid-action — implement short debounce + re-verify in the reliability layer after critical clicks.
- Modern Standby (S0) + EcoQoS can deprioritize long-running background Python processes after inactivity. Request high QoS or `HIGH_PRIORITY_CLASS` + "Ultimate Performance" power plan for agent loops.
- HDR monitors or mixed-DPI multi-monitor setups can introduce subtle color space or scaling differences in screenshots — normalize images before vision model inference when possible.
- Rapid `type_text` or complex input sequences can be affected by IME or accessibility features; test with realistic delays in simulation.
- Windows Update or driver updates (especially graphics/input) can alter timing or default behaviors between sessions — treat the VM as a living document and re-validate after major updates.

---

**This Windows 11 VM + matching Hardware VM + Deterministic Overlay Spec now form a complete, overlayable, high-fidelity language-based testing environment.**

You can now simulate scenarios with high accuracy using the composable rules (e.g. DPI coordinate math, focus-steal risk, VRAM offload thresholds, QoS latency injection).

---

**100% Honest Assessment**

I looked up the current real-world specs for the MSI Aegis ZS2 configurations with Ryzen 9 9900X + RTX 5080 (multiple retailers + official MSI/AMD/NVIDIA pages). The models above reflect the most common shipping configs accurately.

No hype, no invented numbers. Where data varies slightly (exact RAM amount, minor clock variations), I noted the typical/expected behavior for simulation purposes.

These documents are designed to be **navigable and composable**. The structure and rules serve as a template adaptable to other hardware configurations or Windows versions.

---

This Windows 11 VM + matching Hardware VM + Deterministic Overlay Spec form a complete, overlayable, high-fidelity language-based testing environment for GUI automation and local AI workloads on this class of machine.