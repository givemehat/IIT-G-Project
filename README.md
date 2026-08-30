# NASA Li-ion Battery Prognostics & RUL Aging Dataset

A curated, clean, and benchmark-ready dataset for **Battery Health Monitoring**, **State of Health (SOH) Estimation**, and **Remaining Useful Life (RUL) Prediction** for Autonomous Underwater Vehicles (AUVs), Electric Vehicles (EVs), and Energy Storage Systems (ESS).

Derived from the official **NASA Prognostics Center of Excellence (PCoE) Battery Aging Data Set** (ARC-FY08Q4 and ARC 25–56) with standardized cycle telemetry, electrochemical health indicators, and capacity regeneration tags.

---

## 📁 Repository Structure

```tree
battery-health-rul-dataset/
├── data/
│   ├── nasa_battery_cycles.csv        # Master dataset (2,744 cycles across all 32 cells)
│   ├── B0005_cycles.csv               # Benchmark Cell 5 (168 discharge cycles)
│   ├── B0006_cycles.csv               # Benchmark Cell 6 (168 discharge cycles)
│   ├── B0007_cycles.csv               # Benchmark Cell 7 (168 discharge cycles)
│   ├── B0018_cycles.csv               # Benchmark Cell 18 (132 discharge cycles)
│   ├── B0025_cycles.csv ... B0056_cycles.csv  # Additional 28 NASA aging cells
│   └── auv_mission_profile_cycles.csv # Simulated AUV mission profile with thermal variations
├── LICENSE                            # MIT License
└── README.md                          # Comprehensive dataset documentation
```

---

## 📊 Benchmark Cells Summary

| Cell ID | Chemistry | Nominal Capacity | Ambient Temp | EOL Threshold | EOL Cycle | Total Discharge Cycles | Primary Usage |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **B0005** | LiCoO2 (18650) | 2.0 Ah | 24°C | 1.40 Ah (70% SOH) | Cycle 124 | 168 | Model Training |
| **B0006** | LiCoO2 (18650) | 2.0 Ah | 24°C | 1.40 Ah (70% SOH) | Cycle 109 | 168 | Model Training |
| **B0007** | LiCoO2 (18650) | 2.0 Ah | 24°C | 1.50 Ah (75% SOH)* | Cycle 148 | 168 | Model Validation |
| **B0018** | LiCoO2 (18650) | 2.0 Ah | 24°C | 1.40 Ah (70% SOH) | Cycle 96 | 132 | Zero-Shot Out-of-Sample Test |
| **B0025–B0056** | LiCoO2 (18650) | 2.0 Ah | Various (4°C–44°C) | 1.40 Ah | Variable | 2,108 | Large-Scale Pretraining & Robustness |
| **AUV_PACK** | Li-ion Marine | 2.0 Ah | 4°C–20°C (Ocean Depth) | 1.40 Ah | Cycle 142 | 200 | Underwater Vehicle Mission Simulation |

*\*Note: For B0007, the capacity did not drop below 1.40 Ah during the experiment, so 1.50 Ah is the standard research failure threshold per Qiu et al. (2024).*

---

## 📋 Data Schema & Column Definitions

Each CSV file is structured cycle-by-cycle with the following columns:

| Column Name | Data Type | Units | Description |
| :--- | :---: | :---: | :--- |
| `cell_id` | `string` | — | Unique identifier of the battery cell (e.g. `B0005`, `B0018`) |
| `cycle_index` | `integer` | — | Sequential index of the discharge cycle ($1, 2, 3, \dots$) |
| `raw_cycle_idx` | `integer` | — | Original raw telemetry index (includes charge & impedance cycles) |
| `ambient_temperature`| `float` | °C | Ambient room / environmental chamber temperature |
| `capacity` | `float` | Ah | Discharge capacity extracted from constant current / voltage discharge |
| `soh` | `float` | % | State of Health ($\frac{C_k}{C_{\text{nominal}}} \times 100\%$, $C_{\text{nominal}} = 2.0\text{ Ah}$) |
| `v_start` | `float` | V | Terminal voltage at the beginning of discharge |
| `v_end` | `float` | V | Cut-off terminal voltage at the conclusion of discharge |
| `v_min` | `float` | V | Minimum measured voltage during the discharge profile |
| `v_mean` | `float` | V | Average voltage throughout the discharge cycle |
| `v_std` | `float` | V | Standard deviation of voltage trajectory |
| `i_mean` | `float` | A | Mean discharge current drawn |
| `t_start` | `float` | °C | Cell surface temperature at cycle start |
| `t_max` | `float` | °C | Peak maximum temperature reached during discharge |
| `t_mean` | `float` | °C | Mean cell surface temperature over the cycle |
| `t_rise` | `float` | °C | Thermal elevation ($\Delta T = T_{\text{max}} - T_{\text{start}}$) |
| `discharge_duration` | `float` | seconds | Total duration of the discharge cycle |
| `energy_discharged` | `float` | Wh | Total electrical energy delivered ($\int V \cdot I \, dt$) |
| `eol_threshold` | `float` | Ah | End-of-Life capacity failure threshold |
| `eol_cycle` | `integer` | — | Cycle number where capacity permanently falls below EOL threshold |
| `rul_true` | `integer` | cycles | Ground truth Remaining Useful Life ($\max(0, \text{EOL Cycle} - \text{Cycle Index})$) |
| `capacity_diff` | `float` | Ah | Cycle-to-cycle capacity change ($\Delta C = C_k - C_{k-1}$) |
| `is_regeneration` | `integer` | binary | $1$ if $\Delta C > 0.005\text{ Ah}$ (electrochemical rest rebound event), $0$ otherwise |
| `degradation_rate` | `float` | Ah/cycle | 5-cycle rolling slope of capacity loss |

---

## ⚡ Quickstart: How to Load & Use in Python

### 1. Load with Pandas
```python
import pandas as pd

# Load master combined dataset
df_all = pd.read_csv("data/nasa_battery_cycles.csv")
print(f"Total cycle records: {len(df_all)}")
print(f"Available cells: {df_all['cell_id'].unique()}")

# Load specific benchmark cell
df_b0005 = pd.read_csv("data/B0005_cycles.csv")
print(df_b0005[['cycle_index', 'capacity', 'soh', 'rul_true', 'is_regeneration']].head())
```

### 2. Prepare Zero-Leakage Cross-Cell Train/Val/Test Splits
```python
import pandas as pd

df = pd.read_csv("data/nasa_battery_cycles.csv")

# Strict zero-leakage cross-cell split
train_df = df[df['cell_id'].isin(['B0005', 'B0006'])].reset_index(drop=True)
val_df   = df[df['cell_id'] == 'B0007'].reset_index(drop=True)
test_df  = df[df['cell_id'] == 'B0018'].reset_index(drop=True)

print(f"Train samples: {len(train_df)} | Val samples: {len(val_df)} | Test samples: {len(test_df)}")
```

---

## 🔬 Electrochemical Phenomenon Notes

1. **Capacity Regeneration**: After resting periods between discharge runs, internal concentration gradients equilibrate, leading to temporary capacity spikes ($\Delta C > 0$). This non-monotonic behavior is tagged in `is_regeneration`.
2. **Thermal Growth with Aging**: As internal resistance increases due to SEI (Solid Electrolyte Interphase) layer growth, `t_rise` ($\Delta T$) increases significantly across late cycles.
3. **Knee-Point Acceleration**: Degradation transitions from steady linear fade to accelerated exponential decay near the EOL threshold (70% SOH).

---

