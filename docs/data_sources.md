# Data Sources

Every input dataset used in `crop-failure-svi`, with download URLs and a
brief licence note. All are public US federal-government or CDC sources.

## 1. USDA Farm Service Agency — crop-acre eFOIA records

- **What it is**: per-county tabulation of planted, prevented-planting,
  and failed acres, by crop and by irrigation practice, for each crop
  year 2009 through 2023.
- **How we use it**: the **backbone** of the failure-rate panel.
- **Direct URLs**: see [`../data/fsa_acre_data_sources.md`](../data/fsa_acre_data_sources.md).
- **Format**: ZIP per year; inside, one `.xlsx` per year (older years
  use multi-sheet workbooks split by irrigation regime).
- **Licence**: public domain (US federal government work product, 17
  U.S.C. § 105). No attribution required, but cite USDA-FSA.

## 2. USDA NASS Quick-Stats — annual county yields

- **What it is**: county-level annual production statistics by
  commodity (planted area, harvested area, yield per acre).
- **How we use it**: target for the yield-anomaly model and a
  conditioning variable for the failure-rate calculation.
- **API**: https://quickstats.nass.usda.gov/api (a free API key is
  required; the rate limit is ~50 requests/minute).
- **Format**: JSON; the pipeline converts to a pandas DataFrame keyed
  by `(year, state_fips, county_fips, commodity)`.
- **Licence**: public domain.

## 3. CDC / ATSDR Social Vulnerability Index (SVI)

- **What it is**: a percentile-ranked composite of socioeconomic,
  household-composition, racial/ethnic-minority, and housing/transport
  vulnerability at census tract level. Published every 2-4 years.
- **How we use it**: the **social** axis of the analysis. Tract-level
  values are weighted up to counties by per-crop planted area (see
  `docs/methodology.md` §3).
- **URLs (2000, 2010, 2014, 2016, 2018, 2020)**:
  `https://svi.cdc.gov/Documents/Data/{YEAR}/db/states/SVI_{YEAR}_US.zip`
  (replace `{YEAR}` with the published year).
- **Format**: geodatabase or shapefile; we read with `geopandas`.
- **Licence**: public domain (CDC / ATSDR).

## 4. NOAA / National Drought Mitigation Center — US Drought Monitor

- **What it is**: weekly drought-severity polygons (D0 abnormally dry,
  through D4 exceptional drought).
- **How we use it**: per-county annual count of weeks intersecting a
  D2+ polygon = `drought_annual_freq` feature.
- **URL**: `https://droughtmonitor.unl.edu/data/shapefiles_m/USDM_YYYYMMDD_M.zip`
  (Tuesday snapshot date in `YYYYMMDD`).
- **Format**: zipped shapefile.
- **Licence**: public domain (NOAA/NDMC).

## 5. NOAA — heatwave-day records and US Climate Regions

- **What it is**: county-day-level temperature normals used to compute
  heatwave-day counts; static regional climate stratification.
- **How we use it**: `heatwave_annual_freq` feature and a
  `climate_region` one-hot.
- **URL**: NCEI public datasets (https://www.ncei.noaa.gov/products).
- **Licence**: public domain (NOAA).

## 6. USGS — county-level water-use estimates

- **What it is**: county-level surface-water and groundwater
  withdrawals for irrigation, published every 5 years.
- **How we use it**: `surface_water_usage_rate` and
  `groundwater_usage_rate` features, forward-filled to annual.
- **URL**: https://www.usgs.gov/mission-areas/water-resources/science/water-use-data.
- **Licence**: public domain (USGS).

## 7. USDA Risk Management Agency — crop-insurance Cause-of-Loss

- **What it is**: per-policy crop-insurance indemnification records
  tagged by primary cause of loss (Drought, Excess Moisture / Precipitation,
  Hail, Heat, Cold/Frost, Disease, etc.).
- **How we use it**: as **post-hoc** validation of failure-rate
  patterns and as an optional auxiliary target. *Not* used as an
  input to the failure ML, since the loss data are by definition only
  populated after failure occurs.
- **URL**: https://www.rma.usda.gov/SummaryOfBusiness/CauseOfLoss.
- **Licence**: public domain (USDA-RMA).

## 8. US Census TIGER — county boundaries

- **What it is**: vector geometries for every US county.
- **How we use it**: choropleth base layer for every map in the README.
- **URL**: https://www2.census.gov/geo/tiger/TIGER2024/COUNTY/tl_2024_us_county.zip
  (or the equivalent vintage for any prior year).
- **Licence**: public domain (US Census Bureau).

## 9. USDA CDL — tract-level crop area for SVI weighting

- **What it is**: the USDA Cropland Data Layer aggregated to census
  tracts for each year, giving per-tract per-crop planted area.
- **How we use it**: weighting field in §3 of `docs/methodology.md`,
  i.e. the multiplier that converts tract-level SVI to a per-crop
  county-level SVI.
- **URL**: https://nassgeodata.gmu.edu/CropScape/ (the CropScape REST API),
  with the per-tract aggregation handled internally.
- **Format**: GeoTIFF per year per state, aggregated by us to CSV.
- **Licence**: public domain (USDA-NASS).
