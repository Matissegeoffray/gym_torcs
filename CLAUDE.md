# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Focus

The active working file is **`torcs_jm_par.py`**. All work should focus on this file.

**IMPORTANT: After every change made to `torcs_jm_par.py`, update this CLAUDE.md to reflect the new state of the file (parameters, functions, logic, etc.).**

## Overview

`torcs_jm_par.py` is a TORCS UDP client implementing the SCR (Simulated Car Racing) protocol. It contains:

- **`Client`** — manages the UDP socket connection to TORCS, sends/receives data each step
- **`ServerState`** — parses the sensor string `(key value)(key value)...` from TORCS into a dict `S.d`
- **`DriverAction`** — holds the control outputs (steer, accel, brake, gear, clutch) serialized back to TORCS as `R.d`
- **`drive_modular(c)`** — modular driving logic with user-configurable parameters at the top of the file

## Running

```bash
python torcs_jm_par.py --port 3001
```

TORCS server must already be running and configured.

## Telemetry

Every run automatically writes `telemetry.csv` next to the script (overwritten each run, so read it immediately after a run). Columns:

| Column | Description |
|--------|-------------|
| `step` | Timestep index |
| `speedX` | Longitudinal speed (km/h) |
| `rpm` | Engine RPM |
| `gear` | Current gear |
| `accel` | Throttle output (0–1) |
| `brake` | Brake output (0–1) |
| `steer` | Steering output (-1–1) |
| `trackPos` | Lateral position (-1 left, +1 right) |
| `track9` | Forward distance sensor `track[9]` (metres) |
| `target_speed` | Computed target from `dynamic_target_speed()` (km/h) |

**How to diagnose with telemetry:**
- `brake=0` when `speedX >> target_speed` → ABS or sigmoid bug zeroing brake
- `accel` oscillating 0.9/1.0 → traction control threshold too low
- `gear` hunting (2→3→2→3) → `GEAR_SHIFT_DELAY` too small
- `target_speed` drops suddenly mid-straight → sensor discontinuity in `dynamic_target_speed`
- `trackPos` drifting past ±1 → car off track, steering attenuation too aggressive

## Key Sensors (`S.d`)

| Key | Description |
|-----|-------------|
| `speedX` | Longitudinal speed (km/h) |
| `angle` | Car angle relative to track axis (radians) |
| `trackPos` | Lateral position (-1 = left edge, +1 = right edge) |
| `track` | 19 range sensors; `track[9]` is straight ahead |
| `wheelSpinVel` | 4 wheel spin velocities |
| `rpm` | Engine RPM (used for gear shifting) |
| `stucktimer` | Steps the car has been stuck |
| `gear` | Current gear |

## Configurable Parameters (top of file)

Parameters are now split into two dicts — `FAST` (fast section) and `TECH` (technical section). `get_params(S)` selects the right dict based on `distFromStart` vs `SECTION_SPLIT_DIST`.

**Shared globals:**
```python
SECTION_SPLIT_DIST = 1800   # metres: below = FAST, above = TECH
STUCK_THRESHOLD = 100
STUCK_REVERSE_STEER = 0.5
ENABLE_TRACTION_CONTROL = True
```

**Per-section keys (same keys in both `FAST` and `TECH` dicts):**

| Key | FAST | TECH | Description |
|-----|------|------|-------------|
| `STEER_GAIN` | 50 | 50 | Steering sensitivity |
| `CENTERING_GAIN` | 0.20 | 0.20 | Track-centre correction |
| `STEER_LOCK` | 0.366 | 0.366 | Max steering lock (radians) |
| `STEER_ATTN_SPEED` | 80 | 60 | km/h above which attenuation kicks in |
| `STEER_ATTN_COEFF` | 0.05 | 0.05 | Speed-based attenuation multiplier |
| `RPM_UPSHIFT` | 18500 | 18500 | Upshift RPM |
| `RPM_DOWNSHIFT` | [0,3300,6200,7000,7300,7700] | same | Downshift thresholds per gear |
| `GEAR_SHIFT_DELAY` | 10 | 10 | Steps cooldown between shifts |
| `MAX_SPEED` | 300 | 200 | km/h — section top speed |
| `MIN_SPEED` | 80 | 56 | km/h — section minimum speed |
| `LOOK_AHEAD_FAR` | 110 | 110 | metres — full speed above this |
| `ACCEL_SIGMOID_SCALE` | 3.29 | 3.29 | Throttle sigmoid steepness |
| `BRAKE_SIGMOID_SCALE` | 3.29 | 3.29 | Brake sigmoid steepness |
| `ABS_SLIP_THRESHOLD` | 2.0 | 2.0 | m/s — ABS engagement threshold |
| `ABS_MIN_SPEED` | 3.0 | 3.0 | m/s — minimum speed for ABS |
| `ABS_RANGE` | 3.0 | 3.0 | m/s — brake scaling range |
| `TC_SLIP_THRESHOLD` | 5.0 | 5.0 | rear−front wheel spin diff triggering TC |
| `TC_REDUCTION` | 0.30 | 0.30 | throttle cut per TC trigger |
