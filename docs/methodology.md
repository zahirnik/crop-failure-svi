# Methodology

End-to-end description of the analytic pipeline used in `crop-failure-svi`.
This document complements [`../notebooks/README.md`](../notebooks/README.md)
(which lists the 28 source notebooks) and
[`./data_sources.md`](./data_sources.md) (which enumerates the raw data).

## 1. Crop-failure backbone

The first move is to convert the USDA Farm Service Agency's annual
**crop-acre eFOIA** records into a county-year panel of failure rates.
For each year 2009–2023 the corresponding `fsa_acres_*.zip` is downloaded
from the canonical URL (see `data/fsa_acre_data_sources.md`), unzipped,
and the per-county tables are extracted.

The pre-2017 files split records across three sheets (`Total`,
`Irrigated`, `Nonirrigated`), each with different column-header
conventions; the post-2017 files use a single `county_data` sheet with
an explicit `Irrigation Practice` column. Both flavours are harmonised
into a tidy table:

| column | meaning |
|---|---|
| `Year` | calendar year |
| `State`, `County`, `FIPS` | political geography (state and county FIPS) |
| `Crop` | one of {CORN, SOYBEANS, COTTON, WHEAT, BARLEY, OATS, ALFALFA/HAY, SORGHUM} |
| `Crop Type` | sub-type within `Crop` (e.g. ELS vs Upland for COTTON) |
| `Irrigation Practice` | I / N / ALL |
| `Planted Acres` | acres planted |
| `Prevented Acres` | acres on which planting was prevented |
| `Failed Acres` | acres planted but failed |

State names are normalised against `data/state_name_mapping.csv`, which
records the USDA's legacy non-standard state abbreviations
(`MASACUSET`, `PENSLVANA`, …) and maps them to the conventional full
names.

The failure rate is computed as:

```
failure_share = Failed Acres / Planted Acres
```

with corresponding **prevented_share**, **failure_count**, and
**prevented_count** aggregates available at the county-year level.

## 2. NASS Quick-Stats yield join

Annual county-level yields (Quick-Stats API, USDA NASS) are pulled for
the same eight crops. The series is **de-trended** with a rolling
moving-window estimator to expose negative anomalies that are
independent of the long-run yield-improvement trend driven by genetics
and management. The detrended residual is what feeds the yield-anomaly
ML model (`notebooks/10_ml_yield_anomaly.ipynb`); the **fraction of years
2009-2023 with a negative anomaly** per county is what is rendered in
the per-crop yield-anomaly maps in the README.

## 3. SVI county-weighted construction

The CDC SVI is published at census-tract level, with separate
geodatabases for years 2000 / 2010 / 2014 / 2016 / 2018 / 2020. To get
a per-county SVI value that is **specific to the crop being analysed**,
we use the per-crop planted-area table at tract resolution
(`Crop-SVI-YYYY-YYYY.csv` files in the source folder, derived from the
2008 NLCD-Cropland Mosaic and CDL annual layers) and compute a
weighted-mean SVI:

```
SVI_county(crop) = Σ_tracts(SVI_tract × area_planted_to_crop_tract) /
                   Σ_tracts(area_planted_to_crop_tract)
```

This is computed independently for each of the four SVI themes
(`RPL_THEME1` = socioeconomic, `RPL_THEME2` = household composition &
disability, `RPL_THEME3` = minority status & language, `RPL_THEME4` =
housing type & transportation) and the overall composite
(`RPL_THEMES`). Negative sentinel values in the SVI input
(used by CDC for suppressed data) are masked to NaN before averaging.

The 2020 SVI release renamed `EPL_POV` to `EPL_POV150`; this is
reverse-mapped so the time series is comparable across the six
SVI vintages.

## 4. Weather and water indices

For each (year, county) cell we compute:

- **`drought_annual_freq`** — number of weeks in the year that the
  county centroid intersected a USDM D2+ drought polygon (computed
  weekly, summed annually);
- **`heatwave_annual_freq`** — county-day-aware heatwave-day count
  using NOAA's NCEI temperature normals and the standard
  3-or-more-consecutive-day-90th-percentile-anomaly definition;
- **`surface_water_usage_rate`** and **`groundwater_usage_rate`** —
  per-county withdrawal divided by total irrigated area, from the USGS
  national water-use estimates (released roughly every 5 years; we
  forward-fill to annual);
- **`climate_region`** — categorical assignment to one of NOAA's nine
  US climate regions, used as a stratifier in the ML models.

## 5. Crop-insurance integration (RMA)

USDA Risk Management Agency's Cause-of-Loss data are joined on
`(year, state, county_FIPS, crop)`. The fields retained are
**`loss_amount_per_acre`** by cause-of-loss bucket (Drought, Excess
Moisture / Precipitation, Hail, Heat, Cold/Frost, Disease, etc.),
which act as **post-hoc** target labels — they are *not* used as
inputs to the ML failure classifier, since they are by definition only
present when failure has already occurred.

## 6. ML feature panel

After the joins above we hold one feature row per
`(year, county_FIPS, crop, irrigation_regime)`. The feature columns
break down as:

- **Climate**: drought_annual_freq, heatwave_annual_freq, climate_region
  one-hot.
- **Hydrology**: surface_water_usage_rate, groundwater_usage_rate.
- **Vulnerability**: SVI_THEME1..4 (crop-area-weighted), SVI_THEMES.
- **Crop area**: planted_acres (log-transformed), share of county
  agricultural land planted to this crop.
- **Geography**: latitude/longitude of county centroid.

Targets:

- **Classification target** (failure ML): `1` if `failure_share > 0.05`
  for that (county, year, crop), else `0`. The threshold is set high
  enough to remove single-acre noise but low enough to keep prevalence
  in the 5-25% range across crops.
- **Regression target** (yield-anomaly ML): the detrended yield
  residual (Z-scored within crop).

## 7. Models

**Classifier** (`notebooks/09_ml_failure_tuning_final.ipynb`):
gradient-boosted trees (XGBoost) with stratified-grouped cross-validation
where the grouping unit is the county (so the model is never trained and
validated on the same county-year pair). Hyperparameters tuned with
Optuna over `max_depth`, `n_estimators`, `learning_rate`,
`min_child_weight`, `gamma`, `subsample`, `colsample_bytree`,
`reg_alpha`, `reg_lambda`. Evaluation metrics: ROC AUC, F1, per-class
precision and recall.

**Yield-anomaly regressor** (`notebooks/10_ml_yield_anomaly.ipynb`): same
underlying booster, regression objective, evaluated with R² and MAE on
the held-out years.

**SHAP attribution** (`shap.TreeExplainer`) ranks features by mean
absolute contribution to the classifier's logit. Per-feature summary
plots like the one in the README's headline section are saved per crop
under `Final_Exports_2024_09/Shap_Summary_All_Causes_All_Features_V1/`
in the original Drive layout.

## 8. Plotting stack

All maps use `geopandas` + `matplotlib` with a `cartopy` PlateCarree
projection clipped to CONUS. Choropleths use a custom 6-stop yellow→red
quantile ramp tuned to highlight the right tail (high-failure /
high-vulnerability counties) without compressing the bulk of the
distribution. Sankey diagrams are rendered with `plotly`. SVI density
plots use `seaborn.kdeplot` with a fixed bandwidth across crops for
visual comparability.

## 9. Reproducibility notes

- Set `<DATA_ROOT>/` to a working folder before running any notebook.
- The pre-2017 FSA files load slowly (~30 s each) because of the
  three-sheet split; consider pickling intermediate parquet files
  before the merge step in step 6.
- The 2020 SVI tract-level geodatabase has a `> 100-part polygon`
  warning from Fiona that is safe to ignore.
- Stratified grouping by county prevents the most common form of
  spatial-leakage in cross-validation; do not relax it to ordinary
  K-fold without a re-analysis.
