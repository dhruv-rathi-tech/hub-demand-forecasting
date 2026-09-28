# Company_X Micro-Fulfillment Hub Demand Forecasting

> **Dual LightGBM + Bias Calibration Solution** for predicting daily micro-fulfillment order volumes across distributed hubs.

---

## 📌 Overview

This repository contains the end-to-end forecasting pipeline for predicting hub-level daily `OrderVolume`. The solution utilizes a **Dual LightGBM architecture** combining direct demand forecasting and an intermediate session-residual model, followed by **log-space ensemble blending** and **bias calibration**.

---

## 🚀 Solution Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │       Feature Engineering Pipeline           │
                 │  - 364-day seasonal lag & YoY hub growth     │
                 │  - Recency stats (30d/90d momentum)          │
                 │  - Hub & Weekday Bayesian statistics         │
                 │  - Promotional streak & cyclical calendars   │
                 └──────────────────────┬───────────────────────┘
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
       ┌────────────────────────┐              ┌────────────────────────┐
       │   Model 1 (Direct)     │              │  Model 2 (Two-Stage)   │
       │  LightGBM on           │              │  Stage 1: AppSessions  │
       │  log1p(OrderVolume)    │              │  Stage 2: Residual     │
       └───────────┬────────────┘              └───────────┬────────────┘
                   │                                       │
                   └───────────────────┬───────────────────┘
                                       ▼
                      ┌─────────────────────────────────┐
                      │    50/50 Log-Space Ensemble     │
                      │               +                 │
                      │  Bias Calibration (+0.029 shift)│
                      └────────────────┬────────────────┘
                                       ▼
                      ┌─────────────────────────────────┐
                      │  Post-Processing & Validation   │
                      │  (Zero-fill on IsOpen == 0)     │
                      └─────────────────────────────────┘
```

### Key Engineering Highlights:
- **Seasonality & Lag Features**: 364-day exact same-day-of-week historical order volume, app sessions, and spend per session with hub-level year-over-year growth multipliers.
- **Promotional Dynamics**: Continuous promotional run length (`PromoRunDay`), boundary flags (`IsFirstPromoDay`, `IsLastPromoDay`), and forward/backward shift indicators.
- **Hierarchical Hub Aggregations**: Bayesian mean and median statistics at both `HubID` and `(HubID, Weekday)` granularity.
- **Recency & Momentum**: Rolling 30-day and 90-day volume trends capturing demand velocity (`h_growth_vol`, `hub_momentum_30_90`).
- **Sample Time Weighting**: Exponential decay weighting (`0.5 ** (days / 365.0)`) prioritizing more recent operational dynamics.
- **Domain Constraints**: Strict enforcement of closed status (`IsOpen == 0 => OrderVolume = 0`).

---

## 📁 Repository Structure

```
├── solution.ipynb        # Complete reproducible pipeline (EDA, Feature Engineering, Training, Inference)
├── requirements.txt      # Python dependencies
├── .gitignore            # Excludes datasets, submissions, and caches
└── README.md             # Project documentation
```

---

## 🛠️ Getting Started

### 1. Prerequisites & Environment Setup
Clone this repository and install the dependencies:
```bash
git clone <YOUR_REPO_URL>
cd <REPO_DIRECTORY>
pip install -r requirements.txt
```

### 2. Dataset Setup
Create a `datasets/` folder in the project root and place the competition files inside:
```
datasets/
├── orders_train.csv
├── orders_test.csv
└── hub_metadata.csv
```
*(Note: The notebook also natively detects Kaggle environment paths at `/kaggle/input`)*

### 3. Execution
Launch Jupyter Notebook or JupyterLab:
```bash
jupyter notebook solution.ipynb
```
Run all cells sequentially to train the models and generate `submission.csv`.

---

## 📊 Evaluation & Results
- **Validation Metric**: RMSLE (Root Mean Squared Logarithmic Error)
- **Bias Shift Validation**: Validated +0.029 log-shift yields ~3.2% RMSLE improvement (0.114 → 0.110).
