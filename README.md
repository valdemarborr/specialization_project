# Residual-based hydraulic warning indicators from drilling data

This repository contains the code and report materials for a thesis project on residual-based monitoring of drilling hydraulics. Multivariate regression models predict expected standpipe pressure (SPP) and annular pressure (AP) from surface drilling parameters. Standardized residuals are then converted into persistence-filtered warning indicators, including cross-pressure signals that capture shared and decoupled behavior between SPP and AP.

## Motivation

Pressure channels often reflect changes in the drilling system before they become obvious in single-parameter plots. This project explores whether data-driven "expected pressure" models can support decision-making by flagging sustained deviations from expected behavior, while avoiding frequent alarms during normal operational transients.

## Research scope

* Targets: **SPP** and **AP**

* Inputs (per sample): measured depth, TVD, Flow In, ROP, Torque, WOB, RPM, mud density (and optional derived features)

* Data: **four horizontal wells (A–D)**, depth-indexed, approximately 1 m sampling

* Output: residual-based indicators and warning flags designed for monitoring, not event classification

> Important: The project does **not** claim event detection performance for specific downhole events without event-level ground truth. Warnings are interpreted as deviations from the learned expectation.

## Repository structure

The project is organized as follows:

```
.
├── data/
│   ├── 4_well_dataset.xlsx          # Raw input data (4 wells in separate sheets)
│   ├── all_wells_processed.parquet  # Processed dataset with mode labels
│   ├── drill_intra_well_split.parquet    # Intra-well train/test splits
│   ├── drill_leave_one_well_out.parquet  # Leave-one-well-out splits
│   ├── processed/                   # Additional processed datasets
│   └── tables/                      # Generated statistics and metrics tables
├── noteboooks/                      # Main analysis notebooks (sequential execution)
│   ├── 01_explore_wells.ipynb      # Data loading and exploration
│   ├── 02_preprocessing.ipynb      # Cleaning, mode classification, splitting
│   ├── 03_feature_analysis.ipynb   # Feature selection and correlation analysis
│   ├── 04_models.ipynb              # Model training and evaluation
│   └── 05_warning_signs.ipynb      # Residual computation and warning indicators
├── figures/                         # Generated figures organized by workflow stage
│   ├── 01_explore_wells/           # Exploration visualizations
│   ├── 02_classify_operation/      # Mode classification plots
│   ├── preprocessing/               # Preprocessing diagnostics
│   ├── FE/                          # Feature engineering analysis
│   ├── models_n_preds/              # Model predictions and diagnostics
│   └── residuals_n_warnings/       # Residual analysis and warning indicators
└── README.md
```

## Data

Expected raw data schema (one row per ~1 m measured depth):

| Column                  | Description                    |
| ----------------------- | ------------------------------ |
| Depth(m)                | Measured depth (MD)            |
| TVD(m)                  | True vertical depth            |
| SPP Avg(bar)            | Standpipe pressure             |
| Annular Press(barg)     | Annular pressure               |
| Flow In Pum Avg(lpm)    | Pump flow rate                 |
| ROP Avg(m/hr)           | Rate of penetration            |
| Torque Abs Avg(kN-m)    | Surface torque                 |
| WOB Avg(mton)           | Weight on bit                 |
| RPM Surface Avg(rpm)    | Surface RPM                    |
| Dens Mud(sg)            | Mud density                    |

The raw data is stored in `data/4_well_dataset.xlsx` with each well (A, B, C, D) in a separate sheet.

If the dataset is proprietary, keep it out of git and add to `.gitignore`:
* `data/4_well_dataset.xlsx`
* `data/processed/` (optional, depending on size)

## Pipeline overview

The workflow follows these stages, executed sequentially through Jupyter notebooks:

1. **Exploration** (`01_explore_wells.ipynb`)
   * Load raw data from Excel file (4 wells in separate sheets)
   * Basic statistics and data quality checks
   * Initial visualizations of pressure vs depth, flow relationships

2. **Preprocessing** (`02_preprocessing.ipynb`)
   * Clean invalid placeholder values (e.g., -999)
   * Remove unrealistic ROP values (>300 m/hr)
   * Operational mode classification (drill vs low-pressure events)
     * Uses rolling-median plateau reference
     * Classifies "valleys" as low-pressure events when pressure drops below threshold (default: 90% of local plateau)
   * Core drilling mask identification
   * Create train/test splits:
     * Intra-well splits (80/20 per well, depth-ordered)
     * Leave-one-well-out (LOO) splits

3. **Feature Analysis** (`03_feature_analysis.ipynb`)
   * Correlation analysis between features and targets
   * Feature stability across wells
   * Selection of base feature set for modeling
   * Optional engineered features (mass flow, inclination proxy, rolling means)

4. **Model Training** (`04_models.ipynb`)
   * Multivariate regression models for SPP and AP
   * Models: Linear Regression, Random Forest
   * Evaluation using:
     * Within-well splits (80/20 per well)
     * Leave-one-well-out (LOO) splits
   * Metrics: MAE, RMSE, R²
   * AP models can use measured SPP or predicted SPP as input

5. **Warning Indicator Construction** (`05_warning_signs.ipynb`)
   * Compute residuals `r = y - ŷ` for SPP and AP
   * Standardize residuals into z-scores using robust local statistics
   * Rolling magnitude score from z-series
   * Thresholding with persistence requirement
   * Cross-pressure indicators (common-mode and difference-mode) derived from joint SPP/AP residual behavior

6. **Outputs**
   * Metrics tables (MAE/RMSE/R² and warning activity summaries) in `data/tables/`
   * Figures organized by workflow stage in `figures/`

## Running the code

The project uses Jupyter notebooks for all analysis. Execute notebooks in sequential order:

### Recommended execution order

1. **`01_explore_wells.ipynb`** - Load and explore raw data
2. **`02_preprocessing.ipynb`** - Clean data, classify modes, create splits
3. **`03_feature_analysis.ipynb`** - Analyze features and correlations
4. **`04_models.ipynb`** - Train and evaluate models
5. **`05_warning_signs.ipynb`** - Compute residuals and warning indicators

### Starting Jupyter

```bash
jupyter notebook
```

Or use JupyterLab:

```bash
jupyter lab
```

Navigate to the `noteboooks/` directory and execute cells sequentially. Each notebook depends on outputs from previous notebooks:

* `02_preprocessing.ipynb` reads from `data/4_well_dataset.xlsx` and writes to `data/all_wells_processed.parquet`
* `03_feature_analysis.ipynb` reads from `data/all_wells_processed.parquet`
* `04_models.ipynb` reads processed data and model configurations
* `05_warning_signs.ipynb` reads model predictions and computes indicators

### Notebook configuration

Key configuration parameters are set in notebook cells:

**Preprocessing** (`02_preprocessing.ipynb`):
* `spp_valley_frac`: 0.9 (SPP valley threshold as fraction of local plateau)
* `ap_valley_frac`: 0.9 (AP valley threshold)
* `smooth_window`: 21 (rolling window for plateau reference)
* `valley_expand_window`: 4 (expansion window for valley masks)

**Model Training** (`04_models.ipynb`):
* `HOLDOUT_WELL`: "D" (well to hold out for LOO evaluation)
* `TRAIN_MODE`: "drill" (filter to drill mode data)
* `FEATURE_SMOOTHING_MODE`: "actual" (use raw features vs rolling means)
* `RF_PARAMS`: Random Forest hyperparameters (n_estimators=100, max_depth=10, etc.)

**Warning Indicators** (`05_warning_signs.ipynb`):
* `WINDOW_SIZE`: 15 (rolling window for z-score magnitude)
* `THRESHOLD`: 1.5 (z-score threshold for warnings)
* `MIN_RUN`: 5 (minimum consecutive points above threshold)
* `LOWESS_FRAC`: 0.025 (fraction for robust residual smoothing)

Modify these parameters in the respective notebook cells to change behavior.

## Reproducing the thesis results

To reproduce the results:

1. Ensure `data/4_well_dataset.xlsx` is present
2. Execute notebooks in order: `01_...`, `02_...`, `03_...`, `04_...`, `05_...`
3. Run all cells in each notebook sequentially
4. Generated outputs:
   * Processed datasets in `data/`
   * Figures in `figures/`
   * Statistics tables in `data/tables/`


## Key configuration knobs

Important choices documented in the notebooks:

* **Training data filter**: All data vs drill-only (`TRAIN_MODE` in `04_models.ipynb`)
* **Valley masking**: Thresholds for low-pressure event detection (`spp_valley_frac`, `ap_valley_frac` in `02_preprocessing.ipynb`)
* **Valley expansion**: Window for expanding valley masks (`valley_expand_window`)
* **Standardization baseline**: Local statistics vs global (configured in `05_warning_signs.ipynb`)
* **Warning thresholds**: Z-score threshold and persistence requirements (`THRESHOLD`, `MIN_RUN` in `05_warning_signs.ipynb`)
* **Persistence settings**: Rolling window size (`WINDOW_SIZE`)
* **AP conditioning**: Measured SPP vs predicted SPP (configured in `04_models.ipynb`)

## Interpreting warnings

Warnings indicate sustained deviation from expected pressure behavior. They do not identify a specific physical cause on their own. Use them as a triage signal and review drilling context (Flow In, WOB, RPM, ROP, torque) before forming operational interpretations.

Warning indicators are designed to:
* Flag sustained anomalies (persistence requirement reduces false alarms)
* Capture both individual pressure deviations (SPP, AP) and cross-pressure patterns (common-mode, difference-mode)
* Provide interpretable z-scores that can be calibrated to operational risk tolerance

## Contact

* Author: Valdemar Børresen
* Supervisor: Behzad Elahifar (NTNU)
* Email: valdemar.borresen@ntnu.no
* Institution: NTNU

