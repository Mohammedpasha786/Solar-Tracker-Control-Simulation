# Solar Tracker Control Simulation — Design Report

**Project:** Multi-Axis Solar Tracker Controller  
**Institution:** SR University, Warangal — Department of ECE  
**Author:** Afreed  
**Date:** June 2024  

---

## 1. Introduction

This report documents the design, implementation, and validation of a simulation-based control system for a multi-axis solar tracker. The system controls three mechanical axes — Azimuth, Elevation, and Polar/Roll — to maximise solar energy capture throughout the day and across seasons.

---

## 2. System Architecture

### 2.1 Layer Overview

| Layer | Components | Tools |
|---|---|---|
| Sun Position | NREL SPA, Optimal Angle Calc, LSTM | MATLAB |
| Control | 3× PID controllers | Simulink, Control System Toolbox |
| Motor | BLDC motor model | Simscape Electrical |
| Mechanical | Worm gear, rigid body, joints | Simscape Multibody |
| Analysis | PV Array, MPPT, efficiency comparison | Simscape Electrical, MATLAB |

### 2.2 Signal Flow

```
NREL SPA → setpoints [az, el, pol]
         → PID Azimuth  → V_az  → BLDC Az  → Worm Gear → Panel Az
         → PID Elevation → V_el  → BLDC El  → Worm Gear → Panel El
         → PID Polar    → V_pol → BLDC Pol → Worm Gear → Panel Roll
                                              ↑
                         Encoder feedback ────┘
```

---

## 3. Solar Position Algorithm

The NREL SPA (Reda & Andreas, 2004) is implemented in `algorithms/solar_position_spa.m`. It computes solar azimuth and elevation from:

- Geographic coordinates (lat, lon, altitude)
- Date and UTC time
- Atmospheric refraction correction (Bennet's formula)

**Accuracy:** Sub-0.01° for elevations > 5°.  
**IST handling:** UTC offset = +5.5 hours. Tested against NOAA Solar Calculator.

---

## 4. Motor Model

A Brushless DC (BLDC) motor was selected for:
- High efficiency at partial loads
- Clean torque control via PWM
- No brush wear in outdoor environments

**Key parameters:**

| Parameter | Value |
|---|---|
| Supply voltage | 24 V |
| Kv | 300 RPM/V |
| Phase resistance | 0.5 Ω |
| Pole pairs | 4 |
| Gear ratio | 40:1 (worm) |

---

## 5. PID Controller Design

Each axis uses a discrete-time PID controller with:
- Anti-windup integrator clamp (±50 units)
- Derivative filter (N = 10)
- Sample time: 0.1 s

**Elevation axis addition:** gravity feedforward term `Vff = Tg * cos(el) / Kmotor` to cancel gravity load torque.

**Tuning method:** `pidtune()` with Phase Margin = 50°, then manual refinement.

**Performance targets (all axes):**

| Metric | Target | Achieved |
|---|---|---|
| Overshoot | < 15% | < 8% |
| Settling time | < 10 s | < 6 s |
| Phase margin | > 30° | > 45° |
| Steady-state error | < 0.5° | < 0.2° |

---

## 6. Efficiency Results

Annual simulation (Warangal, India, 18°N):

| Configuration | kWh/year | Gain |
|---|---|---|
| Fixed tilt (18°, south) | ~850 | Baseline |
| Single-axis azimuth | ~1100 | +29% |
| Dual-axis (az + el) | ~1200 | +41% |
| Triple-axis | ~1250 | +47% |

*Panel: 400 Wp, 1.96 m², 20% efficiency. Clear-sky model.*

---

## 7. Advanced Features

### 7.1 LSTM Trajectory Predictor
- Input: 30-minute history of (az, el) pairs
- Output: predicted position 5 minutes ahead
- Architecture: LSTM(64) → Dropout(0.2) → FC(32) → FC(2)
- Training: 300 days of SPA data; validation RMSE < 0.5°

### 7.2 Fault Detection
- Stall detection: velocity = 0 AND error > 2° for > 3 s
- Response: flag fault, move to safe park position (az=180°, el=30°)

---

## 8. References

1. Reda, I. & Andreas, A. (2004). *Solar Position Algorithm for Solar Radiation Applications*. NREL/TP-560-34302.
2. MathWorks. *Using the Worm and Gear Constraint Block – Solar Tracker*.
3. Jha, S.K. et al. (2020). *Sun's Position Tracking by Solar Angles Using MATLAB*. ICRESG 2020.
4. NOAA Solar Position Calculator. https://gml.noaa.gov/grad/solcalc/
