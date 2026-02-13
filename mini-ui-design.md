# UI Design (Edge Node) — Mini UART Control Panel (Prototype)

This document defines a minimal, practical UI for controlling an RP2040 over UART
directly on a Raspberry Pi 5 edge node.  
Scope: **single node**, **local UART**, quick parameter changes, start/stop, and live status.

> Not in scope: CoreFusion UI, multi-node routing/tunnels, remote Ethernet control.

---

## Goals

- Fast local bring-up on Pi5 (prototype mode)
- Safe operation: explicit ARM/RUN/STOP
- Deterministic timing remains on RP2040 (UI only configures + reads status)
- Simple debugging: show TX/RX frames, CRC errors, warnings
- Works headless via SSH (terminal UI) or locally in a terminal window

---

## Recommended Implementation

### Option A (recommended): Terminal UI (TUI)
- Python 3 + `pyserial`
- UI framework: `textual` (or `rich` + custom loop)

Why:
- runs everywhere on Pi5
- works over SSH
- minimal dependencies
- easy to package as a single CLI app

Binary name suggestion:
- `edgetrack-uart-ui`

---

## UI Layout (Single Screen)

### Top Bar (Connection + Safety)
- **Serial Port**: dropdown (`/dev/ttyAMA0`, `/dev/serial0`, `/dev/ttyACM0`, etc.)
- **Baudrate**: dropdown (e.g. 115200, 230400, 460800, 921600)
- **Connect / Disconnect** button
- **CRC Status**: `OK` / `ERR` (live)
- **Link**: `RX rate`, `TX rate` (frames/s)
- **E-STOP** (big): immediately sends `STOP` + sets `ena=LOW` (if supported)

### Left Panel (Parameters)
Grouped inputs with “dirty state” (highlight when modified but not applied):

**Timing / Sync**
- `timing_source`: `internal | sensor`
- `fps_request`: numeric input
- `apply_mode`: `immediate | next_frame`

**Exposure / Strobe**
- `strobe_ref`: `XVS | FSTROBE`
- `strobe_offset_us`: numeric input (+/-)
- `strobe_width_us`: numeric input
- `exposure_us`: numeric input
- `dimmer_pct`: slider 0..100

**Settle / Feedback**
- `settle_count`: numeric input 0..10
- `settle_mode`: `frames | time`
- `settle_time_ms`: numeric input

Controls:
- **Send SET** (does not start running)
- **APPLY** (immediate / next_frame according to selection)
- **Save Preset** (writes local JSON/YAML)
- **Load Preset** (reads local JSON/YAML)

### Center Panel (Run Control)
Large, explicit state machine actions:

- **ARM** (sets `ena=HIGH`, sends `ARM` command)
- **RUN** (starts timing loop on RP2040)
- **STOP** (halts timing loop, keeps link alive)
- **DISARM** (sets `ena=LOW`, safe idle)

Optional:
- **Single Shot** (one frame trigger, if supported)
- **Query Status** (manual REPORT request)

### Right Panel (Live Status / Telemetry)
Updated continuously (e.g. 10–20 Hz, or on REPORT):

- `state`: BOOT / IDLE / ARMED / RUN / FAULT
- `fps_actual`
- `frame_period_us`
- `status_ok`
- `warn_flags` (decoded + raw hex)
- `error_code` (decoded + raw)
- Optional sensor lines:
  - `xvs_out_hz`
  - `fstrobe_in_hz`
  - `timing_source_active`
  - `strobe_ref_active`

### Bottom Panel (Log / Frames)
Tabbed log view:

- **Events**: connect/disconnect, apply, arm/run/stop, faults
- **UART Frames (decoded)**: cmd_id, seq_id, payload fields
- **Raw HEX**: raw bytes (optional toggle)
- **Errors**: CRC fails, seq drops, timeouts, malformed frames

---

## UX Rules (Important)

### Safety-first actions
- RUN must be a separate button from APPLY.
- Any FAULT state should:
  - stop RUN automatically (optional but recommended)
  - keep UART alive for diagnostics
  - require explicit user action to clear (e.g. `CLEAR_FAULT`)

### “Dirty parameters” concept
- Editing values should not immediately change hardware.
- Highlight fields changed but not sent.
- Only **SET + APPLY** changes the active config.

### Live feedback behavior
- If `settle_count > 0`, after APPLY the UI should:
  - show “SETTLING…” indicator
  - collect N samples (frames/time)
  - display min/avg/max of `frame_period_us` and `fps_actual`
  - mark “STABLE” when done

### Auto-reconnect
- If UART disconnects:
  - show “DISCONNECTED”
  - attempt reconnect every 2 seconds (optional)
  - do NOT auto-RUN after reconnect (safety)

---

## Presets (Local)

Store presets for reproducible testing:

- Format: JSON or YAML
- Suggested path:
  - `~/.config/edgetrack/presets/*.json`

Preset example fields:
- fps_request, exposure_us, strobe_width_us, strobe_offset_us, dimmer_pct
- timi
