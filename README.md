# crop-failure-vulnerability

**US Crop Failure and the Social Vulnerability Index — mapping where
US crops fail, and who lives in the counties that bear it.**

*A multi-source 15-year pipeline tying USDA county-level crop-failure
records to climate hazards, groundwater dependence, crop insurance, and
the CDC Social Vulnerability Index — with interpretable machine-learning
attribution.*

---

## At a glance

| coverage |  |
|---|---|
| **Time span** | 15 crop years (2009–2023) |
| **Spatial unit** | ~3,100 US counties |
| **Crops** | 8 commodity crops (corn, soybeans, cotton, wheat, barley, oats, hay/alfalfa, sorghum) |
| **Irrigation regimes** | irrigated, non-irrigated, combined |
| **Primary outcome** | per-county per-crop `failure_share = Failed Acres / Planted Acres` |
| **Stressor inputs** | annual drought-area frequency, heatwave-day frequency, 9 NOAA climate-region one-hots, USGS groundwater + surface-water withdrawals, USDA RMA insurance loss-cause, NOAA temperature thresholds (`Thr0` … `Thr35`), 30/90-day SPI / SPEI / PDSI drought indices, VPD (vapour pressure deficit) features |
| **Distributional layer** | CDC Social Vulnerability Index — 4 themes + composite, weighted from tract level to county by per-crop planted area |
| **Models** | hyperparameter-tuned gradient-boosted classifier (failure / no-failure) + moving-window yield-anomaly regressor, both with SHAP attribution |

---

## The story in three acts

### Act I — Where and when US crops fail

**Where.** The mean per-county failure share over 2009–2023 picks out
three clear hotspots: the **Northern Plains** (the spring-wheat / sugar-
beet belt of the Dakotas and northern Minnesota), the **Mississippi
Delta**, and parts of the **Southern Plains** of Texas and Oklahoma.

![Mean crop failure share, US counties, 2009-2023](docs/figures/Map_Counties_Crop_Failure_meanFailShare.png)

**Which crops.** Each commodity has its own stress geography. Corn
concentrates in the rain-fed edges of the Eastern Corn Belt and the
central Plains; soybeans in the Mississippi corridor and the eastern
Great Plains; wheat across the Northern Plains, the Pacific Northwest,
and the Texas Panhandle.

| | | |
|:---:|:---:|:---:|
| ![Corn yield anomaly](docs/figures/Map_Counties_Yield_negAnomaly_CORN.png) | ![Soybeans yield anomaly](docs/figures/Map_Counties_Yield_negAnomaly_SOYBEANS.png) | ![Wheat yield anomaly](docs/figures/Map_Counties_Yield_negAnomaly_WHEAT.png) |
| Corn — fraction of years with negative yield anomaly | Soybeans | Wheat |

**When.** The annual time series of failure events overlaid on
drought-area and heatwave-area fractions shows the **2012 US megadrought**
spike (~4,750 failure events that year) and the secondary 2011 spike;
the corresponding land-area drought stack peaks at ~85 % of US ag
acreage in the same year.

![Annual failure-events line over drought + heatwave area stacked bars](docs/figures/AllCropsFailure_vs_DroughtArea_HeatwaveArea_TimeSeries.png)

### Act II — Why: climate, water, and the social fabric

**Climate.** Annual drought-frequency and heatwave-frequency at county
resolution have very different geographies. Drought concentrates in the
Southwest (Utah, Nevada, Arizona, New Mexico, West Texas); heatwaves
intensify across a broader Southern band, with the Plains a particular
hotspot.

| | |
|:---:|:---:|
| ![County drought annual frequency](docs/figures/Map_Counties_Drought_AnnualFreq.png) | ![County heatwave annual frequency](docs/figures/Map_Counties_HeatWave_AnnualFreq.png) |
| **Drought** annual frequency | **Heatwave** annual frequency |

**Water.** USGS withdrawal rates expose the **groundwater story**: the
High Plains Aquifer (Nebraska, Kansas, the Texas Panhandle), the Central
Valley, and the Mississippi Embayment carry by far the highest per-acre
groundwater dependence in the country. Crop-failure ML feature
importance (Act III, below) confirms `gwu_rate` (groundwater-usage rate)
as a top-tier predictor for several crops.

![Per-county groundwater usage rate](docs/figures/Map_Counties_GroundwaterUsageRate.png)

**Social.** The CDC SVI composite percentile shows the familiar
**Deep-South / Texas borderlands / Appalachia** pattern.

![SVI composite percentile per county](docs/figures/Map_Counties_SVI_RPL_THEMES.png)

### Act III — The cross-cuts: where climate meets vulnerability

**One image, all three axes.** A Sankey diagram of **crop type →
irrigation regime → SVI tier** captures the project in one figure.
Wheat-dominated rain-fed acreage flows disproportionately into the
medium-to-high SVI tiers; irrigated cotton and corn split more evenly.

![Sankey: crop -> irrigation -> SVI](docs/figures/Sankey_Crop_Irrigation_SVI.png)

**A second cross-cut: climate region → irrigation regime → SVI tier.**
The South-East (`SE`) and South-Central (`S`) regions carry the bulk of
flow into the High SVI tier via non-irrigated land; the Northern Great
Plains (`WNC`, `ENC`) split irrigated/non-irrigated more evenly but skew
toward Low-Medium SVI.

![Parallel categories: Climate Region × Irrigation × SVI](docs/figures/ParallelCategories_Climate_Irrigation_SVI.png)

**The headline distributional finding.** For every crop, the orange "high
failure share" density is **shifted toward the higher-SVI end of the
distribution** compared with the green "low failure share" density. That
is: the counties bearing the most acres-failed-per-acre-planted skew
toward higher social vulnerability. The pattern is consistent across
rain-fed and irrigated regimes and across all 8 crops.

![Density of failure share vs SVI percentile, by crop](docs/figures/InequalityPlot_Failure_SVI_Themes.png)

**The correlation matrix.** Underneath this distributional finding sits
a dense correlation structure between the SVI themes (the dark-red block
in the upper-left), the temperature thresholds (the dark-red block in
the lower-right), and the SPEI / SPI / PDSI drought indices in between.
Reading the correlation matrix is what tells you which features the ML
models latch on to next.

![Correlation heatmap: failure share, SVI themes, climate features](docs/figures/HeatMap_FailureShare_CorrelationMatrix.jpg)

---

## What the models learned

Per-crop gradient-boosted classifiers were fitted on the merged
county-year-crop feature panel, with stratified-grouped cross-validation
(group key = county) and Optuna hyperparameter search. The SHAP-summary
plots below rank the features by their mean absolute impact on the
classifier's predicted failure logit. The colour gradient (red = high
feature value, blue = low) shows whether high values of each feature
push the prediction toward "failure" or away from it.

| | | |
|:---:|:---:|:---:|
| ![SHAP CORN](docs/figures/SHAP_Summary_CORN.png) | ![SHAP SOYBEANS](docs/figures/SHAP_Summary_SOYBEANS.png) | ![SHAP WHEAT](docs/figures/SHAP_Summary_WHEAT.png) |
| **Corn** | **Soybeans** | **Wheat** |

Three patterns are visible across crops:

1. **Short-window drought indices** (`spei90d-1`, `spei30d-2`, `spi*`)
   are the top features for both wheat and corn — when the prior
   3-month SPEI is low (blue dots on the left of the strip), the model
   pushes strongly toward "failure".
2. **Vapour pressure deficit** (`VPD-1`, `VPD-Mean`) consistently
   ranks in the top 10 — high VPD (red dots on the right) drives the
   classifier toward "failure" for every crop tested.
3. **Groundwater usage rate** (`gwu_rate`) and the **SVI composite**
   (`RPL_THEMES`) both surface in the top 15-20 features — confirming
   that the model is picking up both the hydrology and the social-
   vulnerability layers we built into the panel.

---

## Pipeline

```mermaid
flowchart TD
    A["USDA FSA crop-acre records<br/>(per-county, 2009-2023)"]
    B["NASS Quick-Stats<br/>county yields"]
    C["CDC SVI tract-level<br/>(2000/2010/2014/2016/2018/2020)"]
    D["USDM weekly<br/>drought polygons"]
    E["NOAA US<br/>Climate Regions + temperature"]
    F["USGS water use<br/>(surface + ground)"]
    G["USDA RMA<br/>crop insurance"]

    P1["01-02. Unzip + tidy<br/>failed-acres table"]
    P2["03. Merge failure<br/>with yield"]
    P3["04-05. SVI prep<br/>(tract -> county weighted<br/>by planted area)"]
    P4["06. Build weather +<br/>drought + VPD index"]
    P5["07-08. Spatial join +<br/>area-outlier filter"]
    P6["12. Yield detrend<br/>(moving window)"]
    P7["modules_2024 05.<br/>Integrate RMA insurance"]

    M["Merged county-year-crop<br/>feature panel"]

    F1["09. Map plots"]
    F2["10. Density plots"]
    F3["11. Sankey / Heatmap /<br/>Parallel Categories"]
    F4["13-17. Final paper<br/>figures (5 sections)"]

    ML1["09 (2024). Tuned crop-<br/>failure classifier"]
    ML2["11 (2024). Moving-window<br/>yield-anomaly model"]

    OUT1["Per-crop failure maps"]
    OUT2["SVI inequality plots"]
    OUT3["SHAP feature ranking"]
    OUT4["Yield-anomaly maps"]

    A --> P1 --> P2
    B --> P2
    C --> P3
    D --> P4
    E --> P5
    F --> P5
    G --> P7
    P2 --> P5
    P3 --> P5
    P4 --> P5
    P5 --> P6
    P6 --> M
    P7 --> M
    M --> F1 --> OUT1
    M --> F2 --> OUT2
    M --> F3 --> OUT2
    M --> F4 --> OUT1
    M --> ML1 --> OUT3
    M --> ML2 --> OUT4

    classDef src fill:#fff7e6,stroke:#a86b00,color:#222
    classDef prep fill:#eef5ff,stroke:#1b4a7d,color:#222
    classDef ml fill:#e8f5e9,stroke:#2e7d32,color:#222
    classDef out fill:#ffeef2,stroke:#a02050,color:#222
    classDef hub fill:#f5f0ff,stroke:#5b2a86,color:#222,stroke-width:2px
    class A,B,C,D,E,F,G src
    class P1,P2,P3,P4,P5,P6,P7,F1,F2,F3,F4 prep
    class ML1,ML2 ml
    class M hub
    class OUT1,OUT2,OUT3,OUT4 out
```

---

## Repository Structure

```
.
├── README.md                                    # this file
├── docs/
│   ├── methodology.md                           # end-to-end method writeup
│   ├── data_sources.md                          # every input dataset with link + licence
│   └── figures/                                 # 16 publication-quality figures (~8 MB)
├── notebooks/
│   ├── README.md                                # full per-notebook map + execution order
│   ├── phase1_2023/                             # 17 notebooks — exploratory pass
│   │   ├── 01_crop_failure_data_preparation.ipynb
│   │   ├── 02_crop_yield_failure_preprocess.ipynb
│   │   ├── …
│   │   └── 17_final_plots_section_5_-_ground_water.ipynb
│   └── phase2_2024/                             # 11 notebooks — refined pass + ML
│       ├── 01_crop_area_weighted_svi_for_counties.ipynb
│       ├── …
│       ├── 09_crop_failure_machine_learning_final_-_tuning.ipynb
│       ├── 10_crop_failure_yield_machine_learning_final.ipynb
│       └── 11_yield_movwinave_anomaly_machine_learning.ipynb
└── data/
    ├── state_name_mapping.csv                   # FSA state-name lookup (USDA legacy)
    └── fsa_acre_data_sources.md                 # canonical USDA FSA URLs, 2009-2023
```

All 28 notebooks have been **sanitised** for public release — Google Colab
`drive.mount()` cells dropped, the original
`/content/drive/MyDrive/US_Crops/` data paths rewritten to a
`<DATA_ROOT>/` placeholder, cell-level `executionInfo.user`
`displayName` / `userId` redacted, and notebook-level `authorship_tag`
removed.

See [`notebooks/README.md`](notebooks/README.md) for the full per-notebook
map and the recommended execution order.

---

## Methodology in a nutshell

We start from the USDA Farm Service Agency's per-county, per-crop
**failed-acres** and **prevented-planting** records, which we harmonise
across the pre-2017 multi-sheet format and the post-2017 single-sheet
format into a tidy panel keyed by
`(year, state-county-FIPS, crop, irrigation_regime)`. We join NASS
Quick-Stats county yields, detrend them with a moving-window estimator,
and define the regression target as the residual.

The CDC Social Vulnerability Index is then **weighted from census-tract
level up to counties by per-crop planted area**, giving a per-crop SVI
that is specific to where each commodity is actually grown. An annual
drought-frequency index per county is derived from USDM polygons, a
heatwave-frequency index from NOAA temperature normals, surface- and
groundwater-withdrawal rates from USGS, and USDA RMA crop-insurance
loss-cause codes from the RMA Summary-of-Business API. The NOAA US
Climate Regions provide a stratification covariate.

The resulting county-year-crop feature panel feeds two interpretable
models: a **hyperparameter-tuned classifier** that predicts the binary
occurrence of crop failure, and a **moving-window yield-anomaly
regressor** that predicts the continuous detrended-yield deviation.
SHAP attribution exposes the relative contribution of climate,
water-resource, and social-vulnerability features. See
[`docs/methodology.md`](docs/methodology.md) for the full description.

---

## Data Sources

Every input dataset, with download URLs and licence notes, is enumerated
in [`docs/data_sources.md`](docs/data_sources.md). All sources are
public, US-federal-government or CDC, and free to use. For the USDA FSA
crop-acre eFOIA files specifically, the canonical 2009–2023 URLs are
kept in [`data/fsa_acre_data_sources.md`](data/fsa_acre_data_sources.md)
so the download step in the pipeline can be reproduced byte-for-byte.

---

## Reproducibility

The notebooks are written to be runnable on any Jupyter / Colab kernel
with the standard scientific-python stack (`pandas`, `geopandas`,
`numpy`, `scikit-learn`, `xgboost`, `shap`, `optuna`, `matplotlib`,
`seaborn`, `plotly`). They reference an external `<DATA_ROOT>/`
placeholder that should point at a working directory holding the input
data and intermediate parquet files. Setting up `<DATA_ROOT>` amounts to:

1. Mirror the directory structure documented in
   [`docs/data_sources.md`](docs/data_sources.md).
2. Download the FSA acre ZIPs listed in
   [`data/fsa_acre_data_sources.md`](data/fsa_acre_data_sources.md).
3. Pull the SVI shapefiles from `https://svi.cdc.gov/Documents/Data/`
   for years 2000, 2010, 2014, 2016, 2018, 2020.
4. Pull NASS county yields with the Quick-Stats API.

Run notebooks in the order documented in
[`notebooks/README.md`](notebooks/README.md).

---

## Citations and Acknowledgements

- **USDA Farm Service Agency** — crop-acre eFOIA data (2009–2023, public domain).
- **USDA NASS Quick-Stats** — county yield series.
- **CDC / ATSDR** — Social Vulnerability Index, 2000–2020.
- **NOAA NCEI** — climate regions, temperature normals, heatwave records.
- **National Drought Mitigation Center** — US Drought Monitor.
- **USGS** — county-level water-use estimates.
- **USDA Risk Management Agency** — crop-insurance loss-cause records.
- **US Census Bureau** — TIGER county boundaries.

---

## License

Code: MIT (unless otherwise noted in individual files).
Data: see the licence notes per source in `docs/data_sources.md`.
Documentation and figures: CC-BY 4.0.
