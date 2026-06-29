# Notebooks

This repository ships **all 28 notebooks** from the analysis pipeline,
split into two phases:

- **Phase 1 (2023)** — `phase1_2023/` — 17 notebooks. The first
  exploratory pass through the data: download, harmonise, build the SVI
  and weather indices, produce the cross-cut plots and the five paper
  sections of figures.
- **Phase 2 (2024)** — `phase2_2024/` — 11 notebooks. The refined pass:
  per-crop SVI weighting, RMA insurance integration, and the
  machine-learning models (classification + yield-anomaly regression).

Every notebook has been **sanitised** for public release:

- Google Colab `drive.mount()` cells were dropped.
- The original `/content/drive/MyDrive/US_Crops/` data paths were rewritten
  to a `<DATA_ROOT>/` placeholder.
- Cell-level `executionInfo.user` `displayName` / `userId` fields were
  redacted.
- The notebook-level `authorship_tag` was removed.

To reproduce, set `<DATA_ROOT>` to a local working directory mirroring
the original Drive layout (see [`../docs/data_sources.md`](../docs/data_sources.md))
and run the notebooks in the order below.

---

## Phase 1 (2023) — `phase1_2023/`

| # | file | role |
|---|---|---|
| 01 | `01_crop_failure_data_preparation.ipynb` | Pull the USDA FSA crop-acre `.zip`s for 2009-2023; unzip and standardise the per-county tables. |
| 02 | `02_crop_yield_failure_preprocess.ipynb` | Filter to the 8 target crops, harmonise irrigation labels, write a tidy per-county-year `failed-acres` table. |
| 03 | `03_crop_yield_failure_merge.ipynb` | Merge the failure table with NASS Quick-Stats county yield. |
| 04 | `04_svi_data_preparation.ipynb` | Pull the CDC Social Vulnerability Index (SVI) at census-tract level for 2000 / 2010 / 2014 / 2016 / 2018 / 2020 and join to FIPS. |
| 05 | `05_svi_data_preparation_2_-_svi_failure_plot.ipynb` | Second SVI prep step + initial SVI ↔ failure plots. |
| 06 | `06_wheather_index.ipynb` | Build the per-county weather index (drought area + heatwave area + USDM). |
| 07 | `07_join_counties_cilmate_regions.ipynb` | Join counties to NOAA US Climate Regions. |
| 08 | `08_remove_very_small_very_big_crop_areas.ipynb` | Drop outlier counties (very small or very large planted area). |
| 09 | `09_map_plots.ipynb` | Per-county failure-share choropleth maps. |
| 10 | `10_svi_-_crop_failure_density_plot.ipynb` | SVI ↔ failure density plots (one per crop). |
| 11 | `11_sankey_diagram_heatmap_group_bar_chart_spider_diverging.ipynb` | Sankey, heatmap, spider, and diverging plots tying crop × irrigation × SVI. |
| 12 | `12_yield_detrend.ipynb` | Yield detrending (moving-average) for the anomaly target. |
| 13 | `13_final_plots_section_1.ipynb` | Final paper figures, section 1 (counties / failure baseline). |
| 14 | `14_final_plots_section_2.ipynb` | Final paper figures, section 2 (climate drivers). |
| 15 | `15_final_plots_section_3.ipynb` | Final paper figures, section 3 (cross-cuts, Sankey). |
| 16 | `16_final_plots_section_4.ipynb` | Final paper figures, section 4 (inequality plots). |
| 17 | `17_final_plots_section_5_-_ground_water.ipynb` | Final paper figures, section 5 (groundwater dimension). |

## Phase 2 (2024) — `phase2_2024/`

| # | file | role |
|---|---|---|
| 01 | `01_crop_area_weighted_svi_for_counties.ipynb` | Tract-level SVI weighted to county scale by the per-crop planted area (the refined SVI used by the ML models). |
| 02 | `02_crop_failure_preprocess_-_new_modified.ipynb` | Refined preprocessing of the FSA failure table (better state-name harmonisation, irrigation taxonomy). |
| 03 | `03_final_plots_section_1.ipynb` | Updated final-figure builder for section 1. |
| 04 | `04_final_plots_section_2.ipynb` | Updated final-figure builder for section 2. |
| 05 | `05_insurance_data_process.ipynb` | USDA Risk Management Agency (RMA) crop-insurance integration. |
| 06 | `06_crop_failure_machine_learning.ipynb` | First classification pass — un-tuned baseline ML on the merged panel. |
| 07 | `07_crop_failure_machine_learning.ipynb` | Second pass — feature pruning + cross-validation (small notebook, follow-on tweaks). |
| 08 | `08_crop_failure_machine_learning_final.ipynb` | "Final" classification with held-out evaluation. |
| 09 | `09_crop_failure_machine_learning_final_-_tuning.ipynb` | **Hyperparameter-tuned final classifier** (the SHAP plot in the main README is from this notebook). |
| 10 | `10_crop_failure_yield_machine_learning_final.ipynb` | Yield-failure regression model trained on the same panel. |
| 11 | `11_yield_movwinave_anomaly_machine_learning.ipynb` | **Moving-window yield-anomaly model** with windowed features. |

---

## Recommended execution order

For a clean reproduction from scratch:

1. Phase 1 notebooks 01 → 12 (data ingest, indices, detrend).
2. Phase 2 notebooks 01 → 02 (refined SVI + preprocess) and 05
   (insurance) to update the merged feature panel.
3. Phase 2 notebooks 06 → 09 in sequence for the classification model
   (each builds on the previous one's saved artefacts).
4. Phase 2 notebooks 10 and 11 for the yield-failure and yield-anomaly
   regression models.
5. Either Phase 1 notebooks 13 → 17 or Phase 2 notebooks 03 → 04 for the
   final figures (the Phase 2 ones supersede the Phase 1 sections 1–2).
