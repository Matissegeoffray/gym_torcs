# TORCS Autonomous Driving Client

This project implements a rule-based autonomous driving agent for [TORCS](http://torcs.sourceforge.net/) (The Open Racing Car Simulator) using the SCR (Simulated Car Racing) UDP protocol.

The work progresses from a minimal base client (`torcs_jm_par_v1.py`) to a fully-tuned driving agent (`torcs_jm_par.py`) with adaptive speed control, ABS, traction control, and per-section track parameters.

---

## Files at a Glance

| File | Role |
|------|------|
| `torcs_jm_par_v1.py` | **Starting point** — base UDP client, simple fixed driving logic |
| `torcs_jm_par.py` | **Final version** — full modular agent with all features |

---

## Version 1 — `torcs_jm_par_v1.py`

This is the base version, built on top of the [snakeoil3](http://xed.ch/project/snakeoil/index.html) Python client for TORCS.

It handles:
- UDP socket connection to the TORCS server (SCR protocol)
- Parsing sensor data from TORCS (`speedX`, `angle`, `trackPos`, `track`, `rpm`, etc.)
- Sending control commands back (`steer`, `accel`, `brake`, `gear`)
- Basic fixed-parameter steering and throttle logic

**This is the starting point.** The driving logic is simple and not tuned for any specific track section.

---

## Final Version — `torcs_jm_par.py`

The final version adds a full set of improvements on top of V1:

### Section-based parameters
The track is divided into 3 zones, each with its own set of tuned parameters:

| Zone | Distance | Character |
|------|----------|-----------|
| `FAST` | 0 – 1900 m | Long straights, high speed |
| `LIGHT` | 1900 – 2800 m | Transition zone |
| `TECH` | 2800 m+ | Technical section, tight corners |

Each zone has independent values for steering gain, target speed, gear shift RPMs, sigmoid scales, ABS thresholds, etc.

### Key features

- **Dynamic target speed** — forward distance sensors compute how much speed is safe based on the road ahead
- **Sigmoid throttle & brake** — smooth acceleration and braking instead of binary on/off
- **ABS** — prevents wheel lock-up under heavy braking
- **Traction control** — reduces throttle when rear wheels spin faster than front wheels
- **Gear logic** — RPM-based upshift/downshift with a configurable cooldown between shifts
- **Stuck recovery** — detects when the car is stuck and applies reverse with counter-steering
- **Telemetry** — writes `telemetry.csv` every run for debugging and tuning

---

## How to Run

TORCS must already be running and waiting for a client on the chosen port.

```bash
# Run the final version
python torcs_jm_par.py --port 3001

# Run the base version
python torcs_jm_par_v1.py --port 3001
```

**Options:**

| Flag | Default | Description |
|------|---------|-------------|
| `--host` / `-H` | `localhost` | TORCS server host |
| `--port` / `-p` | `3001` | TORCS server port |
| `--steps` / `-m` | `100000` | Maximum simulation steps |
| `--debug` / `-d` | off | Print full telemetry to stdout |

---

## Telemetry

After each run, `telemetry.csv` is written (overwritten) in the same directory.

| Column | Description |
|--------|-------------|
| `step` | Timestep index |
| `speedX` | Longitudinal speed (km/h) |
| `rpm` | Engine RPM |
| `gear` | Current gear |
| `accel` | Throttle output (0–1) |
| `brake` | Brake output (0–1) |
| `steer` | Steering output (−1 to +1) |
| `trackPos` | Lateral position (−1 = left edge, +1 = right edge) |
| `track9` | Forward distance sensor (metres) |
| `target_speed` | Computed target speed (km/h) |

---

## Requirements

- Python 3
- TORCS with SCR patch (or vtorcs-RL-color included in this repo)
- No external Python dependencies beyond the standard library

---

## Base Code

This project builds on:
- [vtorcs-RL-color](https://github.com/giuse/vtorcs) — patched TORCS for RL use
- [snakeoil3](http://xed.ch/project/snakeoil/index.html) — original Python SCR client
- [gym_torcs](https://github.com/ugo-nama-kun/gym_torcs) — OpenAI-gym-like wrapper (also included)
