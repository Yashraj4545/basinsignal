# BasinSignal Master Plan

This is the single end-to-end implementation guide for BasinSignal. An agent joining this
repository should read this file first, then follow the phases in order. It defines what we are
building, the first release scope, data, model, user experience, code structure, validation, and
deployment.

## 1. Product in one sentence

Build a transparent decision-support dashboard for the **Yakima River Basin, Washington, USA**
that uses snow, weather, river-flow, and reservoir data to forecast near-term river flow, show
water-shortage risk, and compare water-allocation scenarios.

This is a prototype for exploration and planning. It must never present a recommendation as an
official water-release order or legal allocation decision.

## 2. Who it is for and the first geographic scope

Primary users are Yakima Basin water managers, irrigation districts/farmers, reservoir planners,
and local/environmental planners.

The MVP covers the Yakima River Basin only. Its first USGS river checkpoints are:

```text
Mountain snow and tributaries
    -> Umtanum (USGS-12484500)
    -> downstream diversions / tributaries / reservoir operations
    -> Kiona (USGS-12510500)
    -> Columbia River

American River near Nile (USGS-12488500) and
Ahtanum Creek at Union Gap (USGS-12502500)
are tributary signals.
```

## 3. MVP outcomes

The first usable website must let a user:

1. See latest snowpack, river flow, reservoir storage, and a clear provisional-data notice.
2. Compare current conditions with historical normal conditions.
3. Forecast weekly/monthly flow at one selected USGS gauge 1–3 months ahead.
4. View uncertainty and compare the forecast with simple baselines.
5. Change supply/demand scenario inputs and see a transparent allocation result.
6. Inspect source, retrieval time, units, and limitations for every chart.

Do not attempt an all-Washington product, a legal-rights database, real-time reservoir control, or
a complex deep-learning model in the MVP.

## 4. Data: what to collect and why

| Priority | Dataset | Provider | What is stored | Exact role |
|---|---|---|---|---|
| 1 | Daily streamflow | USGS | date, gauge, daily mean discharge (`00060/00003`) in cfs | Forecast target and observed supply signal |
| 2 | Snow/weather stations | NRCS SNOTEL/AWDB | SWE, precipitation, Tmin, Tmax by station/day | Predict seasonal melt and runoff |
| 3 | Reservoir operations | USBR Yakima Hydromet | storage AF, forebay, releases QD, unregulated flow QU | Estimate managed water supply |
| 4 | Normals | NRCS | 1991–2020 monthly median/average SWE, precip, temperature | Calculate percent-of-normal indicators |
| 5 | Basin weather | GridMET | daily precipitation, Tmin/Tmax, evapotranspiration, VPD | Weather coverage away from stations |
| 6 | TWSA/proration | USBR | forecast issue date, supply forecast, senior/junior allocation % | Compare scenario output with official context |

### Existing USGS status

Raw source files belong in `data/raw/usgs_daily/` and are never edited. Already collected:

| Gauge | File | Coverage |
|---|---|---|
| Yakima River at Umtanum | `USGS-12484500_00060_00003.csv` | 1908-10-01 to 2026-09-08 |
| Yakima River at Kiona | `USGS-12510500_00060_00003.csv` | 1905-10-01 to 2026-09-08 |
| American River near Nile | `USGS-12488500_00060_00003.csv` | 1909-04-25 to 2026-09-08 |

Collect Ahtanum Creek next. The full endpoint and storage rules are documented in
[`yakima_data_acquisition_spec.md`](yakima_data_acquisition_spec.md).

### Data storage rules

```text
data/
  raw/                 # Exact upstream downloads; do not clean or overwrite
  interim/             # Parsed and standardized tables; reproducible from raw
  processed/           # Feature tables, predictions, and allocation inputs
  models/              # Versioned trained model artifacts and metrics
```

Keep a `data/raw/manifest.csv` recording source URL, request parameters, retrieval timestamp, row
count, and whether the source is provisional. Never commit credentials or API keys.

## 5. Architecture

Use one Python repository with a Streamlit application and modular domain code. There is no need
for a separate REST backend in the first MVP: Streamlit calls Python service functions directly.

```text
Official providers -> fetchers -> raw CSV/JSON -> validation/standardization
                                                   -> Parquet feature tables
                                                   -> forecast + allocation services
                                                   -> Streamlit dashboard
```

Create this layout as implementation begins:

```text
app/
  Home.py
  pages/
    1_Basin_Overview.py
    2_Flow_Forecast.py
    3_Allocation_Scenarios.py
    4_Data_Transparency.py
src/
  config.py
  data_fetchers/       # usgs.py, nrcs.py, usbr.py, gridmet.py
  data_processing/     # validate.py, normalize.py, features.py
  forecasting/         # baselines.py, train.py, predict.py, evaluate.py
  allocation/          # inputs.py, rules.py, optimizer.py
  services/            # dashboard-facing orchestration functions
  visualization/       # reusable charts/maps
tests/
  test_fetchers.py
  test_processing.py
  test_forecasting.py
  test_allocation.py
```

Use `pandas`, `pyarrow`, `scikit-learn`, `plotly`, `streamlit`, `requests`, and `pytest` first.
Add geospatial libraries only when the basin map requires them.

## 6. Data pipeline process

1. Fetch raw source data without modifying it.
2. Validate schema, units, date range, duplicate dates, monotonic dates, and missing values.
3. Convert timestamps to a consistent daily date index and units to project standards (cfs, AF,
   inches or millimetres—choose once and document it).
4. Save standardized tables as Parquet under `data/interim/`.
5. Create a daily basin feature table under `data/processed/`.
6. Train/evaluate only from data available as of each historical forecast date.
7. Save forecasts with model version, training cutoff, target gauge, horizon, and creation time.

The first feature table should include: date, target gauge flow, lagged flows (1, 7, 14, 28 days),
rolling flow means, day-of-year, water-year, SNOTEL SWE/precip/temp features, GridMET weather
features, and reservoir storage/release features when collected.

## 7. Forecasting model

### First target

Predict **weekly or monthly mean streamflow 1–3 months ahead** at Umtanum first. Kiona is a
secondary downstream target. Do not start by predicting TWSA/proration: there are too few annual
examples for a trustworthy machine-learning target.

### Algorithms, in order

1. **Seasonal climatology baseline:** historical median flow for the same week/day of year.
2. **Persistence baseline:** recent flow continues forward.
3. **Regularized linear regression (Ridge):** transparent baseline using snow, weather, and lagged
   flow features.
4. **HistGradientBoostingRegressor:** non-linear candidate after the baselines work.

Choose a model only if it improves out-of-sample metrics over the baselines and remains explainable.
Do not add neural networks in the MVP.

### Training and evaluation rules

- Split by time, never randomly shuffle.
- Use expanding-window / walk-forward backtesting. For every forecast origin, train only on dates
  earlier than that origin and predict the future horizon.
- Respect data latency: do not leak final revisions or future GridMET/SNOTEL data into old forecasts.
- Report MAE, RMSE, and percentage error beside each baseline.
- Show prediction intervals using residual quantiles or quantile regression; label them as estimates.
- Keep a simple model card in `data/models/` with target, features, dates, metrics, limitations, and
  training command.

## 8. Allocation scenario engine

The allocation component is a transparent rule/optimization tool, not a learned model.

Inputs: usable storage, predicted inflow, minimum instream-flow reserve, senior demand, junior/
proratable demand, and optional municipal/environmental demand.

Initial rules:

1. Reserve the configured environmental minimum.
2. Serve senior/priority demand up to available supply.
3. Divide remaining proratable supply equally by percentage across junior demand.
4. Show unmet demand, allocation percentage, and every assumption.

Implement this deterministic version first. Only then add a linear-programming scenario solver
(`scipy.optimize.linprog` or PuLP) for user-selected trade-offs. Never claim that the output is an
official legal allocation.

## 9. Website pages

| Page | User sees | Backing data/service |
|---|---|---|
| Basin Overview | map, current flow, snowpack percent of normal, reservoir storage, risk card | latest processed tables |
| Flow Forecast | target-gauge selector, historical line, forecast line/band, baseline comparison | forecasting service |
| Allocation Scenarios | supply/demand sliders, allocation bars, shortages, assumptions | allocation service |
| Data Transparency | source links, refresh time, units, provisional flags, quality checks | manifest + metadata |

Design requirements: plain language, no unexplained acronyms, accessible color choices, source links
on each chart, visible units, and a permanent "prototype—not an operational allocation decision"
notice.

## 10. Delivery phases and acceptance checks

### Phase A — data foundation

- Finish four primary USGS raw files and metadata.
- Download selected NRCS SNOTEL and USBR data.
- Build fetch/validate/Parquet modules and manifest logging.
- Pass tests for non-empty output, expected columns, valid dates, and no duplicate daily records.

### Phase B — forecast MVP

- Build a reproducible feature dataset for Umtanum.
- Implement climatology, persistence, and Ridge baselines.
- Run walk-forward evaluation and save a metrics table.
- Only add gradient boosting if it beats both baselines.

### Phase C — dashboard and allocation MVP

- Build all four Streamlit pages using real processed data.
- Implement transparent scenario rules and assumption controls.
- Add chart/source/provisional status requirements.

### Phase D — deploy and operate

- Add automated tests and a simple scheduled data-refresh job.
- Deploy a staging site, smoke test it, then share it for review.

## 11. Deployment and hosting

Recommended MVP hosting: **Streamlit Community Cloud** with GitHub for the dashboard, because it is
the fastest public prototype path. Keep historical raw data out of the repository if it becomes too
large; publish compact processed Parquet/CSV artifacts or load from an object store.

For a more durable deployment, containerize the app with Docker and host it on Render, Google Cloud
Run, or Azure App Service. Store data in an object store (S3/GCS/Azure Blob) and run a scheduled
fetch job daily. Put keys in host-managed secrets, never `.env` files committed to Git.

Before deploying, provide: `requirements.txt`, a Streamlit entry point, a `.gitignore`, tests, clear
environment-variable documentation, and a public-data attribution/provisional-data notice.

## 12. Definition of done for the first demo

The demo is complete when it is deployed at a shareable URL and a user can select Umtanum or Kiona,
see real historical and latest data, generate a 1–3 month forecast with baseline comparison, run an
allocation scenario, and inspect all data sources/limitations.

## 13. Source documents

- [`yakima_data_acquisition_spec.md`](yakima_data_acquisition_spec.md): exact sources, endpoints,
  parameter codes, gauge list, and raw-data conventions.
- [`drought_gov_washington_analysis.md`](drought_gov_washington_analysis.md): research rationale,
  suitable enhancements, model safeguards, and policy context.

