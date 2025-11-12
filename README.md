# TDMStrobe

**Time‑Division Multiplexed IR strobe and trigger hub** for multi‑camera hand/gesture capture. Designed for **2–4 stereo rigs** (4–8 mono cams) placed with small baselines/stagger to improve precision and occlusion robustness. Supports **single‑strobe** and **double‑strobe** per frame (e.g., **Throw 60°** + **Fill 90°**). The current prototype uses **120° emitters** for quick bring‑up; production aims for 60°/90° optics.

> **Status:** early prototype (electronics/firmware WIP). API and connectors may change.

---

## Why TDMStrobe?

* **Deterministic lighting** for global‑shutter stereo/triangulation
* **TDM (A/B/C/D)** phases prevent cross‑illumination between stereo pairs
* **Double‑strobe** within one exposure (Throw+Fill) for more uniform histograms without longer exposure
* **Low‑latency trigger fan‑out** with simple 2‑wire sync
* **Integrated dimming** (global & per-channel) for fine-grained brightness control

---

## System Overview

```
PC (Host; LAN) ──► Pi 5 (motion host) ──► UART ──► TDMStrobe (RP2040 / Pico)
                                         │
                                         ├─► TRIG A/B (2‑wire) ─► Stereo #1 (L+R)
                                         ├─► TRIG A/B          ─► Stereo #2 (L+R)
                                         ├─► TRIG A/B          ─► Stereo #3 (L+R)
                                         └─► TRIG A/B          ─► Stereo #4 (L+R)

TDMStrobe ─► LED Drivers (per string) ─► IR LED strings (Throw / Fill per camera)
                                 └─► **Dimmer** (global + per‑channel)
```

* **Up to 4 stereo rigs** (expandable hub style). With 3 or more rigs, up to **8 trigger ports** are available from a “mother” stereo pair fan‑out.
* **UART control** from **Pi 5** (which later exposes a LAN control page/API). Centralized settings for the **whole multi‑stereo setup**; TDMStrobe focuses on lighting/trigger only (including **continuous dimming**).

---

## Features (prototype)

* **Phased strobe:** A/B (optionally C/D) per frame @ 60–240 Hz
* **Double‑strobe per exposure:** e.g. 60° Throw, 100–200 µs guard, 90° Fill
* **Per‑channel pulse width:** 100–1500 µs (typical 200–800 µs)
* **Global‑shutter friendly;** rolling‑shutter can use frame‑TDM
* **Continuous dimming (PWM/analog):** global and per‑channel brightness control independent of strobe length
* **Fail‑safe:** outputs LOW on watchdog/reset; optional thermal derating via NTC

---

## Bill of Materials (BOM)

**Compute & Control**

* 1× **Raspberry Pi Pico / RP2040** (controller for TDMStrobe)
* 1× **Raspberry Pi 5** (host; runs capture/triangulation; talks UART→Pico; LAN to PC)

**LED Power Stage (choose one family per string)**

* **AL8861** (1.5 A buck LED driver, PWM/**analog dim**; 4.5–40 V in) — recommended
* or **PT4115** (~1.2 A buck; **fast PWM dim** >10 kHz; very low cost)
* or **Mean Well LDD‑1000H/NLDD‑1400H** (1.0–1.4 A modules; **100–1000 Hz PWM dim**)

**IR Emitters (prototype)**

* 850 nm **10 W stars** (~5 V @ 1 A) on aluminum heatsinks (120°)

  * Production target: 3535 OSLON Black (e.g., **SFH4715B 80°**, **SFH4716B 150°**) with **TIR 60°/90°**

**Power & Protection**

* **24 V PSU** (size for number of strings: *strings* × 1 A × 1.4 headroom)
* **Buck 24→5.1 V** for Pi 5 (≥5 A, low ripple)
* **Fuses**: 1.5–2 A (slow blow) **per LED string/driver**
* **Electrolytics** per driver input: 470–1000 µF low‑ESR + 100 nF
* **TVS** on 24 V rail (e.g., SMBJ33A)

**Thermal & Mechanics**

* Aluminum mounting plate/enclosure; **thermal paste/pads** under LED stars
* Optional **NTC/DS18B20** on heatsink for derating
* Black matte baffles to avoid lens flare

**Cabling & Connectors**

* TRIG A/B 2‑wire sync harnesses (locking)
* Power wiring **AWG18 (0.75 mm²)** to LED strings; short, twisted where possible

---

## Electrical Topologies

**Recommended (per camera side): 3–4 LEDs in series @ 1 A**

* 24 V input → **3 in series** (Vf ≈ 13.5–16.5 V)
* 26–27 V input → **4 in series** (Vf ≈ 18–22 V)
* One **driver per series string**; no parallel strings on a single driver

**Double‑strobe**

* Within one exposure: **Throw pulse** (e.g., 250 µs) → **100–200 µs guard** → **Fill pulse** (e.g., 250 µs)
* Ensure `t_throw + t_guard + t_fill ≤ t_exposure`

---

## Sync & Timing

* **Click‑Stereo (2‑wire)**: distributed **EXPOSE/FLASH** and **phase** markers A/B (optionally C/D)
* **Global‑shutter** preferred. For rolling‑shutter, use frame‑interleaved TDM or long enough pulses to cover row exposure
* Typical: **120 Hz** frames; **0.8–1.2 ms** exposure; **200–800 µs** strobe pulses

---

## Firmware (Pico/RP2040)

* UART CLI (baud configurable): set **rate, exposure window, pulse widths, phase map, channel enables**
* **Dimming modes** (per channel & global):

  * **PWM dim**: 100–1000 Hz (LDD/NLDD) or >10 kHz (AL8861/PT4115) as available
  * **Analog dim** (VSET) for AL8861 if preferred
  * **Priority**: When both strobe and dim are active, **strobe pulses are AND‑gated** by the dim level
* Watchdog; **LED‑OFF default** on reset/no‑sync
* Optional temperature input (NTC) → **auto derate** pulses above threshold

---

## Host Integration

* **Pi 5** collects settings via UART → exposes a **LAN REST/GUI** for centralized control
* GUI includes **dimmer controls** (global + per channel) with presets; can store per‑rig exposure/pulse/dim profiles
* Host also coordinates camera triggers and can store per‑rig presets (geometry, gain, exposure)

---

## Quick Start

1. Wire **24 V PSU** → drivers → LED series strings; add **fuse per string**
2. Add **buck 24→5.1 V** for Pi 5; power Pico from Pi or from 5 V rail
3. Connect TRIG A/B to each stereo rig (lockable connectors)
4. Flash Pico with firmware; open UART console; set **rate/exposure/pulses**
5. **Set dimmer**: start with **Global 80%** (continuous dim), then tune **per‑channel** to equalize histograms
6. Verify with histogram (target **70–80% FS**, no clipping); adjust **pulse widths** and/or **dimmer**

---

## Safety

* 850 nm IR is **invisible and hazardous** at close range. Use shields and comply with **IEC 62471**. Never look into emitters. Provide interlocks where possible.
* When using **continuous dimming**, verify that **maximum average power** stays within LED thermal limits (derate with temperature).

---

## Roadmap

coming soon. 

---

## License

**Apache‑2.0** (code, firmware, docs). Hardware files: to be added; may use CERN‑OHL‑S.

---

## Disclaimer

Prototype hardware. Use at your own risk. Ensure eye‑safety and proper thermal design in all setups.
