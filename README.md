# 🏎️ TORCS Autonomous Driving Agent

> Rule-based autonomous racing bot for TORCS — from a bare UDP client to a fully-tuned agent with ABS, traction control, and section-adaptive parameters.

![Python](https://img.shields.io/badge/Python-3.x-blue)
![Protocol](https://img.shields.io/badge/Protocol-SCR%20UDP-orange)
![TORCS](https://img.shields.io/badge/Simulator-TORCS-green)
![Deps](https://img.shields.io/badge/Dependencies-stdlib%20only-lightgrey)

---

## 🎯 Objective

Build a deterministic, rule-based driving agent that controls a car in TORCS through the SCR UDP protocol — starting from a minimal base client and iterating up to a fully-tuned bot with adaptive speed control, ABS, traction control, and per-section track parameters.

Project completed as part of the **ESILV A3 Algorithms & Autonomous Systems course** — rule-based control on a real racing simulator.

---

## 📊 Results

| Version | Behaviour | Stability |
|---|---|---|
| `torcs_jm_par_v1.py` (baseline) | Fixed steering + throttle | Leaves track on fast corners |
| `torcs_jm_par.py` (final) | Section-adaptive, sigmoid control | Completes full laps reliably |

**Final agent** completes the track across all 4 section types (FAST → LIGHT → TECH → NORMAL) with no manual intervention — using only sensor data and hand-coded control laws.

---

## 🛠️ Stack

- **Language**: Python 3 — `socket`, `math`, `csv`, `os` (no external dependencies)
- **Simulator**: TORCS with SCR patch (`vtorcs-RL-color` included)
- **Protocol**: SCR UDP — sensor parsing + action serialisation
- **Telemetry**: CSV logging (`telemetry.csv`) for offline analysis

---

## 🔬 Approach

### 1. UDP client & sensor parsing
- UDP socket connects to the TORCS server on port 3001
- `ServerState` parses sensor strings: `speedX`, `angle`, `trackPos`, `track[19]`, `rpm`, `wheelSpinVel`, `gear`, `stucktimer`, …
- `DriverAction` serialises control outputs back to the server every step

### 2. Section-based parameter system
The track is divided into 6 zones, each with its own independently tuned config dict:

| Zone | Distance | Character |
|------|----------|-----------|
| `FAST` | 0 – 700 m | Long straights, high speed (≤ 300 km/h) |
| `LIGHT` | 700 – 2400 m | Fast transition zone (≤ 330 km/h) |
| `TECH` | 2400 – 2500 m | Tight corners (≤ 195 km/h) |
| `NORMAL` | 2500 m+ | 90° corners (≤ 305 km/h) |
| `LAST_CORNER_APPROACH` | 2600 m | Wide entry — push to right side |
| `LAST_CORNER` | 2700 m | Turn-in to left apex |

Each zone defines its own `STEER_GAIN`, `MAX_SPEED`, `RPM_UPSHIFT`, sigmoid scales, ABS thresholds, traction-control params, and racing-line target.

### 3. Control modules

- **Steering** — angle correction + lateral centering, attenuated by speed above a configurable threshold
- **Sigmoid throttle & brake** — smooth, continuous control output replacing binary on/off logic
- **Dynamic target speed** — forward distance sensor (`track[9]`) sets safe speed based on road ahead
- **ABS** — compares longitudinal speed to mean wheel-ground speed; modulates brake when slip is detected
- **Traction control** — cuts throttle when rear wheels spin faster than front wheels
- **Gear logic** — RPM-based upshift/downshift with a configurable step cooldown
- **Stuck recovery** — detects prolonged zero-speed and reverses with counter-steer

### 4. Telemetry
Every run writes `telemetry.csv` with step-by-step: `speedX`, `rpm`, `gear`, `accel`, `brake`, `steer`, `trackPos`, `track9`, `target_speed`, `distFromStart` — used for manual tuning.

---

## 📂 Project structure

```
gym_torcs/
├── torcs_jm_par.py          # Final agent — full modular implementation
├── torcs_jm_par_v1.py       # Baseline — bare UDP client, fixed logic
├── telemetry.csv            # Last-run telemetry (overwritten each run)
├── practice.xml             # TORCS race config
├── vtorcs-RL-color/         # Patched TORCS build (SCR + vision support)
├── Etat de l'art/           # Background research documents
└── README.md
```

---

## ⚙️ How to run

TORCS must already be running and waiting for a client on the target port.

```bash
# Final agent
python torcs_jm_par.py --port 3001

# Baseline
python torcs_jm_par_v1.py --port 3001
```

| Flag | Default | Description |
|------|---------|-------------|
| `--host` / `-H` | `localhost` | TORCS server host |
| `--port` / `-p` | `3001` | TORCS server port |
| `--steps` / `-m` | `100000` | Max simulation steps (~50 steps/s) |
| `--debug` / `-d` | off | Print full sensor telemetry to stdout |

---

## 💡 Key takeaways

- **Section-based configs beat global tuning**: a single global parameter set works on straights or corners, never both — splitting the track into zones was the decisive step for stable lap completion.
- **Sigmoid control is worth the extra lines**: replacing `if speed > target: brake` with a sigmoid gave smooth, continuous outputs that eliminated the brake-chatter visible in the V1 telemetry.
- **ABS needs real wheel physics**: the first ABS version compared speeds naively and triggered randomly at low speed — computing actual wheel-ground speed from `wheelSpinVel × radius` fixed it immediately.

---

## 👤 Author

**Matisse Geoffray** — 3rd-year Engineering Student at ESILV

🔍 Looking for a **2-month ML / Data Science internship** (June–July 2026) — Paris or Singapore

📧 matisse.geoffray@gmail.com · [GitHub](https://github.com/Matissegeoffray)
