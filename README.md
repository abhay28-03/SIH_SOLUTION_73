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
```

### System Architecture

**Stage 1 — Data Ingestion & Preprocessing**

- Raw AWS Telemetry (Temp, RH, Pres — 24h Window)
- Leakage-Free Imputation (Train-fitted median fill)

**Stage 2 — Dual-Channel ML Execution**

- **Channel A (Point Anomaly Detector):** Rolling MAD Bounds, Isolation Forest, Domain Rules Engine  Outputs 
- **Channel B (Scenario Classifier):** 504 Aggregated Features, Zero Calendar Leakage, Frozen XGBoost V2  Outputs  & 

**Stage 3 — Decision Arbitration Gate**

- **High-Confidence Bypass:** If  Accept 
- **Point Confirmation:** If  &  is True  Accept 
- **False Alarm Suppression:** If  &  is False  Force Output to NORMAL

**Stage 4 — Action, Dispatch & UI**

- NORMAL  Log to Database
- STUCK / MISSING  Dispatch Technician
- BIAS / DRIFT  Quarantine Data

### Operational Use Cases

- **Preserving Genuine Extreme Weather Data:** During rapid temperature drops or pressure instability, Channel A flags a point anomaly, but Channel B recognizes the correlated shifts across sensors and outputs normal with high confidence. The Arbitrator suppresses Channel A's warning, preserving vital storm data.
- **Detecting the Invisible Flatline:** If a damaged sensor transmits a constant physically valid value (e.g., 3.2 m/s) for a day, Channel A sees no outliers. Channel B finds a 24-hour variance of 0.0 and outputs `stuck_sensor` with 98% confidence, utilizing the High-Confidence Bypass to quarantine the fault.
- **Automated Role-Based Dispatch:** Confirmed faults interface with network operations via Role-Based Access Control (RBAC). The system generates a geospatial maintenance ticket for Field Technicians and a digital quarantine flag for Data Analysts.

### Benchmark Performance & Evaluation

Tested out-of-sample on held-out 24-hour test scenarios:

| Metric             | Serial Cascade Baseline | SkyGuard AI Arbitrated |
| ------------------ | ----------------------- | ---------------------- |
| **Test Accuracy**  | 43.33%                  | **76.67%**             |
| **Macro F1-Score** | 48.12%                  | **77.62%**             |
| **Normal Recall**  | 30.00%                  | **80.00%**             |

### Technology Stack

- **Machine Learning Core:** Python 3.10+, XGBoost V2 (Frozen Model Core), Scikit-Learn (Isolation Forest Ensemble), Pandas & NumPy (504 Statistical Window Aggregations)
- **Backend Microservice:** FastAPI (Asynchronous REST API service), Uvicorn (High-performance ASGI web server), Pydantic (Strictly-typed telemetry ingestion models)
- **Frontend Dashboard:** React 18 & Vite, Tailwind CSS, Plotly.js & Lucide React

### Quickstart Guide

**1. Setup Virtual Environment**

Bash

```bash
git clone https://github.com/NeuroCodex/SIH_SOLUTION_73.git
cd SIH_SOLUTION_73
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

```
**2. Start Backend Service**

Bash

```

uvicorn Backend.main:app --host 0.0.0.0 --port 8000 --reload

```

**3. Start Frontend Application**

Bash

```

cd frontend
npm install
npm run dev

```

### Prototype Demonstration

**1. Dashboard Overview** Telemetry operational health, total processed records, anomaly rate, and fault classification distribution. *<img width="1535" height="862" alt="Screenshot 2026-09-30 140159" src="https://github.com/user-attachments/assets/0b153357-68a6-4839-b65e-81aae8012966" />
*

**2. Model Architecture & Pipeline Flow** The four-stage processing pipeline showing parallel ML execution and the Decision Arbitration Gate. *<img width="1535" height="848" alt="Screenshot 2026-09-30 140238" src="https://github.com/user-attachments/assets/a49594bb-bed4-467e-b9b6-b7b547801670" />
*

**3. Diagnostic Pipeline & Arbitration Analysis** Held-out scenario evaluation logs demonstrating High-Confidence Bypass and False Alarm Suppression. *<img width="1535" height="857" alt="Screenshot 2026-09-30 140259" src="https://github.com/user-attachments/assets/5b95394c-707c-4d9c-81f9-4e3891aedaf4" />
*
```
