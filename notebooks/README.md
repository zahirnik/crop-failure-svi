# Notebooks

The full pipeline used in the analysis spans 28 Jupyter / Colab notebooks
across two phases (initial exploratory pass in 2023, refined ML pass in
2024). The two shipped in this repository are the **most representative**
of the ML stage of the project; the remaining notebooks were
preprocessing / plotting helpers that produced the figures embedded in
the [main README](../README.md) and the [methodology doc](../docs/methodology.md).

## Notebooks in this repository

| file | what it does |
|---|---|
| `09_ml_failure_tuning_final.ipynb` | Hyperparameter-tuned crop-failure classifier (XGBoost / Random Forest variants) trained on the merged county-year feature table. Loads the feature panel, runs a stratified grouped train/test split, fits and tunes the classifier, evaluates with ROC AUC / F1 / per-class precision and recall, and writes a SHAP-summary feature ranking. |
| `10_ml_yield_anomaly.ipynb` | Detrended yield-anomaly regressor. Uses a moving-window detrending of the NASS county-yield series and predicts negative-anomaly events from the same SVI + climate + insurance feature panel. |

Both notebooks reference an external `<DATA_ROOT>/` directory that should
point at a working folder mirroring the original Google Drive layout. To
reproduce, materialise the parquet / CSV files documented in
[`docs/data_sources.md`](../docs/data_sources.md) and rebind `<DATA_ROOT>`
to that directory.

## Full pipeline (notebooks not shipped here)

For completeness, the broader pipeline was structured as follows. Notebook
numbering matches the original ordering on disk; the brief description
lists the one-liner each notebook contributed.

### Phase 1 — exploratory pass (2023)

| step | role |
|---|---|
| 1 | Pull the USDA FSA crop-acre `.zip`s for 2009-2023; unzip and standardise the per-county tables |
| 2 | Filter to the 8 target crops, harmonise irrigation labels, write a tidy per-county-year `failed-acres` table |
| 3 | Merge the failure table with NASS Quick-Stats county yield |
| 4–5 | Pull the CDC Social Vulnerability Index (SVI) at census-tract level for 2000 / 2010 / 2014 / 2016 / 2018 / 2020 and join to FIPS |
| 6 | Build the per-county weather index (drought area + heatwave area + USDM) |
| 7 | Join counties to NOAA US Climate Regions |
| 8 | Drop outlier counties (very small or very large planted area) |
| 9 | Per-county failure-share choropleth maps |
| 10 | SVI ↔ failure density plots (one per crop) |
| 11 | Sankey / heatmap / spider / diverging plots tying crop × irrigation × SVI |
| 12 | Yield detrending (moving-average) |
| 13–17 | Final-figure builders for the five paper sections (counties / drivers / cross-cuts / inequality / groundwater) |

### Phase 2 — refined pass (2024)

| step | role |
|---|---|
| 1 | Tract-level SVI weighted to county scale by the per-crop planted area |
| 2 | Refined preprocessing of the FSA failure table (better state-name harmonisation, irrigation taxonomy) |
| 3–4 | New final-figure builders for Section 1 and Section 2 |
| 5 | USDA Risk Management Agency (RMA) crop-insurance integration |
| 6 | First classification pass — un-tuned baseline ML on the merged panel |
| 7 | Second pass — feature pruning + cross-validation |
| 8 | "Final" classification with held-out evaluation |
| 9 | **Tuned final classifier** *(shipped here as `09_ml_failure_tuning_final.ipynb`)* |
| 10 | Yield-failure regression model trained on the same panel |
| 11 | **Moving-window yield-anomaly model** *(shipped here as `10_ml_yield_anomaly.ipynb`)* |
