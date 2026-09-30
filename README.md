![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-Backend%20API-009688)
![React](https://img.shields.io/badge/React-Vite%20Frontend-61DAFB)
![Tailwind CSS](https://img.shields.io/badge/Tailwind%20CSS-Styling-06B6D4)
![XGBoost](https://img.shields.io/badge/XGBoost-V2%20Core-orange)
![Scikit-Learn](https://img.shields.io/badge/Scikit--Learn-Isolation%20Forest-F7931E)

# SkyGuard AI — Automated Weather Station Telemetry Diagnostics

**SkyGuard AI** is an operational, real-time telemetry fault diagnostic engine engineered for Automated Weather Stations (AWS). Developed by **Team NeuroCodex** for the Smart India Hackathon 2026, it eliminates the trade-off between false alarms during natural extreme weather and diagnostic blindness to persistent sensor failures.

Rather than treating sensor readings as isolated instantaneous values or relying on brittle sequential cascades, SkyGuard AI executes a **Dual-Channel Parallel Arbitration Architecture**. The system evaluates instantaneous point-level anomalies and 24-hour temporal shapes concurrently to detect and isolate sensor degradation across weather observation networks.

---

## Problem Statement

Automated Weather Stations deployed across diverse terrains operate under severe environmental conditions, leading to physical degradation, sensor fouling, and transmission dropouts. Traditional Quality Control (QC) pipelines suffer from critical structural flaws:

* **The Cascade Trap:** A stuck or biased sensor produces values that remain within physical limits (e.g., a humidity sensor locked at 65%). Sequential point-level detectors fail to flag them, causing downstream root-cause classifiers to be starved of inputs and collapsing baseline accuracy to **43.33%**.
* **False Alarms in Natural Extremes:** Extreme weather events (such as microbursts or cyclonic pressure dips) produce sudden gradients. Static threshold filters misclassify these genuine atmospheric anomalies as sensor spikes, deleting crucial severe-weather data.
* **Calendar & Temporal Leakage:** Machine learning models that train on calendar timestamps memorize seasonal warming and cooling cycles rather than learning the physical dynamics of sensor failure.

---

## Solution Overview

SkyGuard AI replaces the serial cascade with two concurrent pipelines and a terminal logic gate:

* **Channel A (Instantaneous Point Anomaly Detector):** Evaluates raw telemetry row by row using physical limit envelopes, rolling Median Absolute Deviation (MAD) bounds, and an Isolation Forest, emitting a binary flag ($S_{det}$).
* **Channel B (Temporal Scenario Classifier):** Operates on the same 24-hour window and computes 504 statistical aggregations (e.g., half-window variance, linear slope, step-changes) with all calendar timestamps stripped. A frozen XGBoost V2 model outputs a root-cause class and a softmax confidence ($C_{rc}$).
* **Decision Arbitrator:** Merges both outputs with deterministic rules to suppress false alarms or bypass point detectors for zero-variance flatlines.

---

# System Architecture

```text
=========================================================================
                 STAGE 1: DATA INGESTION & PREPROCESSING
=========================================================================
                           [ RAW AWS TELEMETRY ]
                       (Temp, RH, Pres — 24h Window)
                                     │
                                     ▼
                       [ LEAKAGE-FREE IMPUTATION ]
                        (Train-fitted median fill)
                                     │
=========================================================================
                 STAGE 2: DUAL-CHANNEL ML EXECUTION
=========================================================================
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
   [ CHANNEL A ]                                           [ CHANNEL B ]
Point Anomaly Detector                                  Scenario Classifier
         │                                                       │
• Rolling MAD Bounds                                    • 504 Aggregated Features
• Isolation Forest                                      • Zero Calendar Leakage
• Domain Rules Engine                                   • Frozen XGBoost V2
         │                                                       │
         ▼                                                       ▼
[ S_det (True/False) ]                                 [ y_rc & Confidence C_rc ]
         │                                                       │
=========================================================================
                 STAGE 3: DECISION ARBITRATION GATE
=========================================================================
         └───────────────────────────┬───────────────────────────┘
                                     ▼
             ┌───────────────────────┼───────────────────────┐
             ▼                       ▼                       ▼
       [ C_rc >= 0.50 ]      [ C_rc < 0.50 & S_det ]  [ C_rc < 0.50 & !S_det ]
     High-Confidence Bypass     Point Confirmation      False Alarm Suppression
             │                       │                       │
             ▼                       ▼                       ▼
          [ y_rc ]                [ y_rc ]               [ NORMAL ]
                                     │
=========================================================================
                 STAGE 4: ACTION, DISPATCH & UI
=========================================================================
                                     ▼
                       [ FINAL ARBITRATED DIAGNOSIS ]
                                     │
             ┌───────────────────────┼───────────────────────┐
             ▼                       ▼                       ▼
        [ NORMAL ]              [ STUCK / MISSING ]       [ BIAS / DRIFT ]
             │                       │                       │
             ▼                       ▼                       ▼
      [ LOG TO DB ]         [ DISPATCH TECHNICIAN ]    [ QUARANTINE DATA ]
