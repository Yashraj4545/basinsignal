# Drought.gov Washington: Deep Analysis for BasinSignal

**Prepared:** 2026-09-05 · **Author:** BasinSignal research (Water Resource Management hackathon track)
**Primary page analyzed:** <https://www.drought.gov/states/washington>
**Target basin:** Yakima River Basin (Columbia Basin as comparison/fallback)
**Intended stack:** Python · Streamlit · modular backend · ML/data science · optimization

> **Note on method.** Quotations, units, update frequencies, and source attributions below were captured directly from the live Washington page (fetched 2026-09-05) and from official product pages (NOAA/NIDIS, NCEI, USGS, NRCS, NASA, USBR, WA Ecology). Anything that could not be confirmed from an authoritative source is explicitly flagged **"Needs verification."** No APIs, datasets, values, licenses, or capabilities have been invented. Data-valid dates (e.g., U.S. Drought Monitor "09/01/26") are snapshot-dependent and will change.

---

## 1. Executive Summary

Drought.gov is the **U.S. Drought Portal**, operated by NOAA's **National Integrated Drought Information System (NIDIS)**, a multi-agency program within NOAA's Climate Program Office / Office of Oceanic and Atmospheric Research (<https://www.drought.gov/about>, <https://www.drought.gov/about/who-we-are>). It is not a data producer itself; it is a **data aggregator and visualization layer** that consolidates and reformats indicators from primary producers — NOAA/NCEI, NOAA/CPC, NOAA/NWS, USDA/NRCS, USGS, NASA, NDMC, UC Merced/Climate Engine, and the Washington State Department of Ecology — into web- and GIS-ready formats (<https://www.drought.gov/data-download>).

For BasinSignal, Drought.gov is best classified as:

- **A supporting source** for derived, drought-focused indices (USDM, SPI/SPEI, MIDI, EDDI) that are cheap to acquire and useful as **model features** and **dashboard context**.
- **A discovery/pointer layer** for the primary, basin-scale data that the ML and optimizer actually need (USGS streamflow, NRCS SNOTEL, USBR Yakima Project operations), which are almost always better obtained **from the upstream provider directly**.
- **Not** a primary source for model-training hydrology at basin scale, because its strongest products (USDM) are deliberately expert-blended national maps (quarter-degree-era county/"local observer" resolution) and its gridded products (GridMET) are best pulled from the originating Climatology Lab / Climate Engine / Google Earth Engine.

**Net recommendation:** Use Drought.gov for context, SPI/EDDI/MIDI feature rasters, and weekly USDM percent-area time series for the Yakima (obtainable via the USDM REST statistics API at HUC8 resolution). Build the actual BasinSignal pipeline on primary sources: USGS NWIS streamflow, NRCS/AWDB SNOTEL snowpack, USBR Hydromet reservoir storage + TWSA forecasts, GridMET/nClimGrid meteorology, and USDA/NASS water-use context. See §5, §10.

---

## 2. Website Inventory

Items below are the data products/features visible on the Washington page (<https://www.drought.gov/states/washington>) and its immediate sub-pages (the state data page <https://www.drought.gov/states/washington/data>, the Data Catalog <https://www.drought.gov/data-maps-tools>, and the Data Download page <https://www.drought.gov/data-download>).

Page snapshot statistics (header, data-valid 09/01/26): 3.2 M Washington residents in areas of drought; drought area change 0.0% vs prior week; July ranked 30th driest on record since 1895 (0.37 in. precipitation, −0.53 in. from normal); January–July ranked 27th driest since 1895 (19.62 in., −3.19 in.).

### 2.1 U.S. Drought Monitor (USDM) — the centerpiece map

- **What it measures:** Location and intensity of drought across five classes — D0 (Abnormally Dry, precursor) and D1–D4 (Moderate → Exceptional). Combines physical indicators with input from local observers; used by USDA for disaster-declaration and loan triggers. Produced by rotating authors from **NDMC (Univ. of Nebraska–Lincoln), NOAA, USDA, NASA** (<https://droughtmonitor.unl.edu/About/WhatistheUSDM.aspx>).
- **Unit/category:** Categorical D0–D4 (+ "No drought"); also 1-week change map (−3 to +3 class change).
- **Spatial resolution:** Vector polygons delineated by authors at county- to multi-county scale for decision purposes; a 500 m re-gridded display product exists but is explicitly **not** official for decisions (<https://droughtmonitor.unl.edu/DmData/Metadata.aspx>). Approx. 4-mile digitization error in pre-2004 archives.
- **Temporal resolution / update:** Weekly. Valid 07:00 ET Tuesday; released Thursday morning.
- **Historical coverage:** 2000–present (weekly). Pre-2004 shapefiles digitized from image archive.
- **Upstream provider:** NDMC / USDA / NOAA / NASA (consortium).
- **Programmatic access:** Yes — GIS data (shapefile, KMZ/KML, GML, WMS, GeoJSON, grid) at <https://droughtmonitor.unl.edu/DmData/GISData.aspx>; **REST statistics API** for % area by D0–D4 per week at state, county, and **HUC2/4/6/8** level, returning CSV/XML/JSON: <https://droughtmonitor.unl.edu/DmData/DataDownload/WebServiceInfo.aspx>. Drought.gov mirrors weekly USDM GeoJSON/TIFF at <https://www.ncei.noaa.gov/pub/data/nidis/geojson/us/usdm/> (GCS mirror: `https://storage.googleapis.com/noaa-nidis-drought-gov-data/...`).
- **Direct official link:** <https://droughtmonitor.unl.edu/> · <https://www.drought.gov/data-maps-tools/us-drought-monitor>
- **BasinSignal relevance:** High as a **contextual feature and dashboard layer**; usable at HUC8 basin scale via the statistics API for the Yakima. Do **not** use its category as the allocation ground truth (§4).

### 2.2 USDM 1-Week Change Map

- **What it measures:** Where drought improved/no-change/worsened (−3…+3 class change) relative to the prior week. Same source, cadence, resolution as USDM. Downloadable per week with USDM GIS data.
- **BasinSignal relevance:** Low; useful only as a trend context widget.

### 2.3 Short-Term and Long-Term Multi-Indicator Drought Index (MIDI)

- **What it measures:** Objective, computer-generated blends of indices. Short-term MIDI ≈ previous 3 months (combines PDSI, Palmer Z-Index, 1-mo and 3-mo SPI); Long-term MIDI ≈ up to 5 years. Method follows NOAA/NWS CPC methodology; categories D0–D4 are percentile-based (e.g., D1 = drier than 80–90% of conditions in the 1979–present reference period), plus W0–W4 wet side (<https://www.drought.gov/data-maps-tools/multi-indicator-drought-index-midi>).
- **Unit/category:** Categorical D0–D4 / W0–W4.
- **Spatial resolution:** 4 km GridMET grid.
- **Temporal / update:** Updated every **5 days with a 4–5 day delay** (data-valid 08/28/26 in snapshot). Reference period **1979–present**.
- **Upstream provider:** **UC Merced**, computed via **Climate Engine**, from **GridMET**.
- **Programmatic access:** Yes — raster (COG/GeoTIFF/TIF tiles) under NCEI/GCS "current-conditions" downloads; e.g., CE-GRIDMET-derived indices at <https://www.drought.gov/data-download>. Underlying GridMET directly from Climatology Lab / Climate Engine / Earth Engine.
- **Labeled:** "Experimental."
- **Direct official link:** <https://www.drought.gov/data-maps-tools/multi-indicator-drought-index-midi>
- **BasinSignal relevance:** Medium as a **feature** (integrated moisture-deficit) with clear latency; better than USDM for ML because it is objective; weaker than direct GridMET variables because it is an aggregation.

### 2.4 Precipitation conditions

- **What it measures:** 7-day total (inches), 30-day % of normal, 60-day % of normal; normal = 1991–2020 average.
- **Unit:** inches; percent of normal (class borders `<25%, 25–50, 50–75, 75–100, 100–150, 150–200, 200–300, >300%`).
- **Spatial / temporal / update:** ~4 km GridMET grid; daily update with 3–4 day delay.
- **Upstream provider:** UC Merced (GridMET).
- **Programmatic access:** Yes — GridMET NetCDF from <https://www.climatologylab.org/gridmet.html>, Earth Engine collection `IDAHO_EPSCOR/GRIDMET`, Climate Engine API, plus drought.gov-ready COG versions.
- **BasinSignal relevance:** Core MVP feature (basin precipitation anomaly from GridMET/PRISM).

### 2.5 Temperature conditions

- **What it measures:** 7-day average max temperature (°F); 7-day anomaly and 30-day anomaly (departure from 1991–2020 normal, °F).
- **Spatial / temporal / update:** ~4 km; daily with 3–4 day delay. Same source/access as §2.4.
- **BasinSignal relevance:** Core MVP feature (heat → evaporative demand → melt timing/soil moisture).

### 2.6 Streamflow conditions

- **What it measures:** 1-day average streamflow expressed as **percentile vs. historical same-day conditions** at each gauge: Record Low / Much Below Normal (<10th) / Below Normal (10–25th) / Normal (25–75th) / Above Normal (75–90th) / Much Above Normal (>90th) / Record High.
- **Spatial resolution:** Point (station). **Only gauges with ≥30 years of record are ranked.**
- **Temporal / update:** Daily (yesterday's mean daily discharge, cfs); map from USGS.
- **Upstream provider:** USGS (WaterWatch logic — see §9: WaterWatch is being decommissioned in favor of the National Water Dashboard / HYSWAP statistics).
- **Provisional:** "These streamflow data are provisional and subject to revision" (<https://waterdata.usgs.gov/provisional-data-statement/>).
- **Programmatic access:** Yes — primary data via USGS Water Data APIs at <https://api.waterdata.usgs.gov/> (daily-values collection) and legacy services at <https://nwis.waterservices.usgs.gov/docs/dv-service> (parameter 00060 = discharge, cfs; statCd 00003 = daily mean).
- **BasinSignal relevance:** **Core MVP target + feature.** Compute your own percentiles with USGS's `HYSWAP` package (<https://doi-usgs.github.io/hyswap/>) rather than depending on the map layer.

### 2.7 Soil moisture conditions

Two widgets on the page (data valid 09/04/26 and 09/01/26):

- **SPoRT-LIS (NASA):** 0–100 cm soil moisture **percentile** classes (0–2, 2–5, 5–10, 10–20, 20–30, 30–70, 70–80, 80–90, 90–95, 95–98, 98–100) from the Noah land-surface model run at **~3 km (0.03°)**; climatology **1981–2013**, real-time run since April 2015, updated every 6 h. Page text also gives "1981–2013" baseline — *the legend's generic "historical measurements for this day of the year" baseline is **Needs verification*** (<https://www.drought.gov/data-maps-tools/nasa-sport-lis-soil-moisture-products>). GeoTIFFs available, e.g., `vsm_percentiles` at <https://geo.nsstc.nasa.gov/SPoRT/modeling/lis/conus3km/geotiff/vsm_percentiles/>; archive via NASA GHRC DAAC, DOI 10.5067/SPORT/LIS/DATA101.
- **Crop-CASMA (NASA SMAP + USDA NASS + George Mason Univ.):** soil moisture **anomaly** (−70%…+70% breakpoints) derived from NASA SMAP remote sensing; baseline **2015–present**; updated daily with a 3-day delay (<https://www.drought.gov/data-maps-tools/crop-condition-and-soil-moisture-analytics-tool-crop-casma>).
- **Also available via the Soil Moisture product dashboard** (not on the state page): CPC Leaky-Bucket monthly soil-moisture percentile (GeoJSON at GCS), NationalSoilMoisture.com RK/NLDAS 50 cm blends, and GRACE groundwater-storage anomaly (updated daily, 4-day delay) (<https://www.drought.gov/indicators/soil-moisture-product-dashboard/data>).
- **BasinSignal relevance:** Strong enhancement. SPoRT-LIS percentile is an excellent feature and dashboard state indicator; Crop-CASMA/SMAP contributes remote-sensing cross-checks. GRACE is landscape-scale (≈300 km) — too coarse for basins *except as regional context* ("Needs verification" for how much weight BasinSignal gives it).

### 2.8 Snowpack and snow-drought information

Not a map section on the state page (the markdown capture has no SWE/snow-drought map — **Needs verification** against the interactive layer picker). What is confirmed:

- Narrative: "Approximately three-quarters of the runoff from the Cascades originates as snowpack" (page text).
- Snow drought topic page provides: **SNOTEL % of median SWE** (1991–2020 median, HUC6 basins) and **SNODAS SWE % of 2004–2021 average** (NOAA NOHRSC, ~1 km, daily) (<https://www.drought.gov/topics/snow-drought>, <https://www.drought.gov/data-maps-tools/nohrsc-national-snow-analyses>).
- Snow-drought taxonomy: *dry snow drought* (below-normal cold-season precipitation) vs *warm snow drought* (near-normal precipitation falling as rain / early melt).
- NRCS/Washington Snow Survey is linked from the resources list on the page.
- **Access:** SNOTEL via NRCS AWDB REST/SOAP APIs; basin graphs for HUC4 1703 (Yakima) at <https://nwcc-apps.sc.egov.usda.gov/awdb/basin-plots/POR/WTEQ/assocHUC4/1703_Yakima.html>; SNODAS via NSIDC <https://nsidc.org/data/g02158> and Climate Engine.
- **BasinSignal relevance:** **Core MVP** for a snowmelt-driven basin. SWE % of normal at Yakima SNOTEL sites is arguably the single most predictive seasonal variable.

### 2.9 Reservoirs and groundwater references

- **Reservoirs:** No reservoir layer appears on the Washington state page in the capture (**Needs verification** for the interactive widget). Drought.gov does maintain a **USBR Hydromet Reservoir Storage** product page linking the **Yakima teacup diagram** — five reservoirs (Keechelus, Kachess, Cle Elum, Bumping, Rimrock; total storage capacity ≈ 1,065,670 AF per the teacup page) — at <https://www.usbr.gov/pn/hydromet/yakima/yaktea.html> (<https://www.drought.gov/data-maps-tools/usbr-hydromet-reservoir-storage>).
- **Groundwater:** Present only as boilerplate ("reservoirs, groundwater... key to monitoring") plus the GRACE product on the soil-moisture dashboard (§2.7). No Washington groundwater-well layer on the page.
- **Programmatic access:** USBR Hydromet daily/15-min data via web forms and USBR **RISE** open data (time-series query + API) at <https://data.usbr.gov/>. **All Hydromet data are labeled PROVISIONAL.**
- **BasinSignal relevance:** Reservoir storage (AF) and USBR's official **Total Water Supply Available (TWSA)** forecast are **core MVP data** for the Yakima optimizer (§5, §7).

### 2.10 Outlooks and forecasts

- **7-day Quantitative Precipitation Forecast (QPF):** NWS Weather Prediction Center; liquid precip inches; updated once daily, valid from 07:00 ET; tiles/GIS data at <https://www.ncei.noaa.gov/pub/data/nidis/tile/noaa-qpf-5km-2d/>.
- **8–14-day precipitation and temperature outlooks:** CPC probability of above/near/below normal (33–40 … >90% classes); updated daily.
- **Monthly Drought Outlook** (CPC): last day of calendar month; **Seasonal Drought Outlook** (CPC): third Thursday of the month.
- **EDDI subseasonal forecasts (1–4 week):** evaporative-demand drought index forecast tiles; source NOAA/PSL + WRCC/Climate Engine.
- **Not visible here:** 3-month (seasonal) CPC precipitation/temperature probability maps are *not* on the Washington state page (only 8–14-day) — **Needs verification** on the national outlooks pages.
- **Upstream providers:** NWS WPC, NOAA CPC.
- **BasinSignal relevance:** Dashboard scenario controls (use outlook probabilities to perturb supply scenarios). For true seasonal water-supply probabilistic guidance at Yakima scale, prefer **NWRFC ESP** water-supply forecasts (§5) — the page links NWRFC natural/forecast products.
- **Direct links:** <https://www.drought.gov/forecasts>, <https://www.nwrfc.noaa.gov/natural/index.html>, <https://www.nwrfc.noaa.gov/ws/ws_info.php>.

### 2.11 Historical conditions

- **USDM (2000–present, weekly):** described in §2.1.
- **SPI from nClimGrid-Monthly (1895–present, monthly):** 9-month SPI widget shown; base period 1895–2014; timescales 1–72 months; netCDF at <https://www.ncei.noaa.gov/pub/data/nidis/indices/nclimgrid-monthly/>.
- **Paleoclimate (Living Blended Drought Atlas):** Palmer Modified Drought Index (PMDI) reconstructed from **tree rings**, June–August values, back to year 0 for some areas; through ~2017; yearly (<https://www.drought.gov/data-maps-tools/living-blended-drought-product-lbdp-historical-drought-information>).
- **Aggregated historical tool:** <https://www.drought.gov/historical-information>.
- **BasinSignal relevance:** SPI monthly from 1895 is the best long archive for the Yakima (pre-dates gauge records); use for backtesting context rather than training (stationarity caveats).

### 2.12 Additional maps / external services referenced

- **Data download page (GIS/web-ready):** JSON/GeoJSON, COG/TIFF, and XYZ map tiles for most products, plus foundational temperature/precipitation datasets (nClimGrid-Daily 1951–present; nClimGrid-Monthly 1895–present; ACIS Interpolated Grid; CPC Unified CONUS 1948–present; CPC Unified Global 1979–present) — all mirrored to public Google Cloud Storage buckets (no login for public reads) (<https://www.drought.gov/data-download>).
- **Topic pages that carry products not serialized on the state page** (**Needs verification** for exact contents): **Snow Drought** (§2.8), **Soil Moisture** (<https://www.drought.gov/topics/soil-moisture>) , **Water Supply** (<https://www.drought.gov/topics/water-supply>), **Vegetation/Fire** (VegDRI/VHI, fire-danger products), **Evaporative Demand (EDDI)** (<https://www.drought.gov/data-maps-tools/evaporative-demand-drought-index-eddi>).
- **Washington-specific linked resources (all official):**
  - WA Dept. of Ecology Statewide Conditions: <https://ecology.wa.gov/Water-Shorelines/Water-supply/Water-availability/Statewide-conditions>
  - Washington State Drought Contingency Plan (2018): <https://apps.ecology.wa.gov/publications/documents/1811005.pdf>
  - Washington State Climate Office: <https://climate.washington.edu/>
  - USGS Washington Water Conditions: <https://waterdata.usgs.gov/state/Washington/>
  - NWS NW River Forecast Center: <https://www.nwrfc.noaa.gov/rfc/>
  - WSU AgWeatherNet: <http://weather.wsu.edu/>
  - USDA Northwest Climate Hub (incl. "Washington & the U.S. Drought Monitor" process PDF): <https://www.climatehubs.usda.gov/hubs/northwest>
  - Pacific Northwest DEWS: <https://www.drought.gov/dews/pacific-northwest>

### 2.13 What is NOT on the page

No EDDI/VegDRI/fire map, no reservoir table, no groundwater map, and no explicit snow-analyses layer are serialized in the 2026-09-05 capture of the state page. These live behind the interactive map layer picker or on topic pages (links above). **Needs verification** by manually toggling the map layers in a browser.

---

## 3. Data Suitability Matrix

| Dataset/Product | Source | Variable | Resolution | Update Frequency | Historical Data | Access Method | Best Use in BasinSignal | Limitations |
|---|---|---|---|---|---|---|---|---|
| USGS daily streamflow (NWIS) | USGS | Discharge, cfs (param 00060, stat 00003) | Point (gauges) | Daily (~80% within hours; provisional) | Decades; many ≥30 yr | Water Data APIs / `dataretrieval` (pip) / NWIS legacy | **Core MVP — model target + supply signal** | Provisional values revised; regulations/diversions mask natural flow — use gauges on unregulated reaches |
| USBR Yakima Hydromet | US Bureau of Reclamation | Reservoir storage (AF), flows (QD/QU cfs), precip, SWE | Point (basin ops network) | 15-min & daily; reports by ~06:45 MT | Long term (project ops archive; RISE) | RISE API / web forms | **Core MVP — reservoir + TWSA supply for optimizer** | PROVISIONAL; historical archives need web-forms/CSV scraping; not a true API on the classic site |
| USBR Yakima TWSA/proration forecast | US Bureau of Reclamation | % of entitlement for junior ("proratable") rights | Basin (5-reservoir system) | Monthly Apr–Sep | News releases; forecast archive **Needs verification** | Press releases + Hydromet pages | **Core MVP — allocation ground truth / label** | Small sample (~1 value/season); not an open machine API |
| NRCS SNOTEL (AWDB) | USDA/NRCS NWCC | SWE, precip accum., Tmax/Tmin, soil moisture | Point (site triplets) | Daily (midnight) / hourly | Varies; many from 1980s | AWDB REST + SOAP API | **Core MVP — snowpack feature, seasonal predictor** | Mountain point sites only; provisional & revised; outliers in WTEQ |
| NRCS basin SWE % of normal / forecasts | USDA/NRCS | SWE % of median (1991–2020); water-supply forecast | Basin (reports), HUC6 | Daily (SWE), monthly-issuance forecasts | 30-yr normals; forecast archive | WCSS reports/CSV; AWDB | **Core MVP feature + baseline forecast** | Forecasts are regression-based (good baseline); access via web/report CSV |
| GridMET | UC Merced Climatology Lab | Precip, Tmax/Tmin, ETo/ETr, VPD, radiation, PDSI/EDDI inputs | ~4 km, daily | Daily (+3–4 day formal; early/provisional/permanent statuses) | 1979–present | climatologylab.org, Earth Engine (`IDAHO_EPSCOR/GRIDMET`), Climate Engine, USGS STAC | **Core MVP — meteorological features** | Semi-operational/pro-bono; no formal DOI SLA; late data replaced by provisional/permanent versions |
| PRISM precipitation & normals | Oregon State Univ. / PRISM | Monthly & daily precip/T, 1991–2020 normals | ~800 m–4 km | Monthly/daily | 1895–present | prism.oregonstate.edu | Enhancement — high-res normals for %-normal features | Widely used; download via FTP/HTTP pages |
| nClimGrid SPI/SPEI | NOAA/NCEI | SPI/SPEI gamma & Pearson-III, 1–72 mo | ~5 km, monthly | Monthly (next-month release) | 1895–present | netCDF at NCEI/NIDIS dir; SPI-daily (1951–) | Feature + long backtest archive | Base periods differ (1895–2014 monthly vs 1991–2020 daily SPI) |
| USDM categories | NDMC/NOAA/USDA/NASA | D0–D4 spatial categories + %-area | Vector polygons; HUC2–8 stats | Weekly (Thu) | 2000–present | GIS data + REST stats API (state, county, HUC) | **Context/feature at HUC8; dashboard** — not allocation truth | Expert-blended; coarse boundaries; gridded version unofficial; pre-2004 ~4 mi digitization error |
| NASA SPoRT-LIS soil moisture percentile | NASA/MSFC | VSM percentile 0–100 cm | ~3 km (0.03°), 6-hourly | Daily on drought.gov | Climatology 1981–2013; real time 2015– | GeoTIFF (geo.nsstc.nasa.gov), GHRC DAAC | **Enhancement — soil-moisture feature/label** | Noah-modeled, not observed; expanded-boundary caveats; baseline text inconsistencies |
| NASA SMAP / Crop-CASMA | NASA + USDA NASS + GMU | Root-zone/surface soil moisture anomaly | ~9 km (SMAP) / CASMA grids | Daily (3-day delay) | 2015–present | Drought.gov/GEE/ASF | Enhancement — remotely sensed cross-check | Short record; depth limited; latency |
| NOHRSC SNODAS | NOAA/NWS NOHRSC | SWE, snow depth (physically blended) | ~1 km, daily | Daily | 2003–present (NSIDC g02158) | NSIDC, Climate Engine | Enhancement — gridded snow context | Analysis/model blend; regional artifacts |
| NWFS/NWRFC ESP water-supply forecasts | NOAA/NWS NW River Forecast Center | Natural/regulated seasonal flow vol., % of normal, exceedance | Forecast points (incl. Yakima subbasins) | Daily ESP runs; water-supply season | Archive via XM.LINK/CSV | NWRFC ESP maps; Forecast Report CSV; NWPS/HEFS APIs | **Enhancement — probabilistic supply input to scenarios** | Probability = weather uncertainty only; model-state error excluded (per NWRFC docs) |
| CPC outlooks & QPF | NOAA/CPC, NWS/WPC | 7-day QPF; 8–14-day, monthly/seasonal drought outlooks | Grid (tiles), ~25 km+ | QPF daily; monthly/seasonal issuances | Archives via NCEI/NIDIS tiles; CPC pages | https + tiles | Dashboard scenario controls | Coarse; probabilities, not forecasts; no long custom histories |
| Washington DOE drought declarations / WSAC | WA Dept. of Ecology | Binary drought declarations; WSAC minutes; explanations | Watershed (e.g., Upper/Lower Yakima, Naches) | Event-driven (seasonal) | Declarations 2023–2026 (Yakima); drought plan 2018 | Ecology pages/orders (PDFs) | **Ground truth for risk threshold (75% of normal)** + policy context | Not a dataset API; qualitative; threshold uses 1991–2020 median supply |
| USDA FSA drought disaster designations | USDA Farm Service Agency / USDM | Administrative disaster areas | County | Weekly | Archive | GeoJSON at NCEI/NIDIS dir | Context only | Policy signal lags; not physical |

---

## 4. Critical Assessment

**Can we use Drought.gov itself as our model-training data source?** Partially, and only for its derived indices. Drought.gov is a reformatter/aggregator: it does not produce streamflow, snowpack, or reservoir measurements. For training hydrology at basin scale you want the primary producers: USGS NWIS (streamflow), NRCS AWDB (snow), USBR (storage), GridMET/nClimGrid (meteorology). Drought.gov-hosted COGs are convenient copies of GridMET/nClimGrid/SPI that you can legitimately use for features, but you should treat them as convenience mirrors of the originals, and use USDM category data only as described below.

**Which products can serve as model features?**
- GridMET precipitation/temperature/ET/VPD (daily, 1979–) — primary meteorological features.
- SPI/SPEI from nClimGrid (multiple accumulation periods) — drought-frequency features.
- SNOTEL SWE (% of median) and basin SWE index — snowpack features.
- SPoRT-LIS soil-moisture percentile — antecedent moisture feature.
- Antecedent streamflow at unregulated gauges + reservoir storage (as of forecast date) — system-state features.
- Short-/Long-Term MIDI — compact integrated moisture-deficit features (optional).

**Which products can serve as labels or ground truth?**
- **Observed streamflow** (USGS daily values) at a natural/unimpaired reach — the most defensible regression target.
- **USBR TWSA / proration %** (junior rights allocation) — the most decision-relevant target for allocation, though the historical sample is tiny (~1 value/season; systematic archive **Needs verification**).
- **Washington drought declarations** (2023–2026 Yakima) and the state's statutory threshold (supply < 75% of normal + undue hardship, RCW 43.83B) — a weak, few-sample *policy* ground truth for a risk-score calibration, not a training target.
- USDM category is acceptable as a *coarse* label for a "drought-status" demonstration classifier, clearly framed as a blend, but it is expert-subjective (§below).

**Which products are only suitable for dashboard context?**
- USDM D0–D4 and change maps (as a map layer and %-area time series for the basin).
- Monthly/Seasonal Drought Outlooks, 8–14-day outlooks, HEFS/NWPS flood outlooks — scenario narrative.
- GRACE groundwater anomaly and paleoclimate reconstructions — long-context panels.
- USDA FSA disaster-designation maps — policy context.

**What risks exist if we rely too heavily on drought categories?**
- USDM categories are a **blend of indicators plus local-observer judgment**, redrawn weekly, with boundaries that are not physically precise; the 500 m gridded product is explicitly non-authoritative (<https://droughtmonitor.unl.edu/DmData/Metadata.aspx>).
- Categories are **ordinal bins** (D0–D4), discarding magnitude; two regions with the same class can have very different absolute water availability.
- Washington's own drought determination is **not** USDM-based — it is supply-vs-75%-of-normal + hardship (RCW 43.83B; <https://ecology.wa.gov/water-shorelines/water-supply/water-availability/statewide-conditions/drought-response>). A model trained on USDM category alone will therefore disagree with the state's actual decision trigger.
- Using categories as the dependent variable yields poor calibration for allocation volumes and can look spuriously good (autocorrelated, slow-moving categories).

**Why is statewide data insufficient for basin-level allocation?**
- Water availability is institutionally and hydrologically local: allocation in the Yakima is governed by **prior appropriation**, the 2019 *Acquavella* adjudication (**first in time, first in right**; 1905 cutoff separates senior/proratable/junior classes), a five-reservoir system with 1.07 MAF storage, and per-reach instream-flow obligations (<https://ecology.wa.gov/water-shorelines/water-supply/water-availability/yakima-basin>). A statewide precip/soil-moisture aggregate cannot resolve the Upper-Yakima/Naches/Lower-Yakima divergence that drives the two statutory thresholds (Ecology's 2025 order covered Upper Yakima, Lower Yakima, and Naches watersheds separately: <https://ecology.wa.gov/getattachment/b0c273fb-4bf9-4691-b8b9-4b61142d7593/Drought-2025-April-Declaration-Order.pdf>).
- Snow varies dramatically across the basin's elevation gradient; **only ~15% of the basin drains through a reservoir** (Ecology blog, <https://ecology.wa.gov/blog/may-2026/yakima-basin-water-supply-update-fourth-year-of-drought>), so snowmelt timing, not a state average, sets summer supply.
- Municipal demand, irrigation-district entitlements, and fish flows are all place-specific; none are represented by statewide drought categories.

**What additional data must be collected for a credible Yakima prototype?**
1. USGS daily streamflow at a small set of basin gauges (e.g., verified example: **USGS 12484500 Yakima River at Umtanum, WA**; select the rest from the Washington site inventory <https://waterdata.usgs.gov/state/Washington/>) — focus on natural/unregulated reaches and an index gauge for the valley.
2. SNOTEL SWE timeseries for Yakima HUC4 1703 sites via AWDB REST.
3. USBR reservoir storage (5 reservoirs, daily AF) + the official monthly **TWSA forecasts** (junior-right %) with their issuance dates.
4. GridMET (or PRISM) precipitation/temperature/ET for the basin polygon; SPI/SPEI from nClimGrid monthly as long-history features.
5. Minimum instream-flow and entitlement/demand parameters from published sources (Yakima Basin Integrated Plan, USBR project data, Ecology) — with documented assumptions where unpublished (§7).
6. Optionally: SPoRT-LIS soil moisture, SNODAS SWE, WSU AgWeatherNet ag stations.

---

## 5. BasinSignal Data Pipeline Recommendation

Recommended minimally-viable pipeline (all sources official; all links verified in §2–§4). Latencies are as documented by each provider.

```text
Source → Variables → Frequency → Storage Format → BasinSignal Module → Purpose
```

### Core MVP data

```text
USGS NWIS daily values (api.waterdata.usgs.gov / dataretrieval) 
  → discharge (cfs, param 00060), gage height 
  → daily (provisional ~same-day) 
  → Parquet (time-series store) 
  → data_fetchers/usgs.py; forecasting module 
  → model target + supply index + dashboard trend
```

```text
NRCS SNOTEL via AWDB REST (wcc.sc.egov.usda.gov/awdbRestApi) 
  → SWE (WTEQ, in), precip accum (PREC), Tmax/Tmin 
  → daily 
  → Parquet + site metadata 
  → data_fetchers/nrcs.py; feature builder 
  → snowpack feature + %-of-normal seasonal predictor
```

```text
USBR Yakima Hydromet (data.usbr.gov RISE time-series; classic web forms as fallback) 
  → reservoir storage (AF), daily mean flows (QD), computed natural flow (QU) 
  → 15-min / daily 
  → Parquet 
  → data_fetchers/usbr.py; supply model 
  → reservoir storage feature + optimizer supply input
```

```text
USBR monthly TWSA / proration forecasts (usbr.gov news + hydromet pages; archive Needs verification) 
  → junior-right allocation % (senior = 100%) 
  → monthly Apr–Sep 
  → CSV (curated registry of issuances) 
  → allocation module 
  → optimizer ground truth + scenario anchor
```

```text
GridMET (climatologylab.org / Google Earth Engine IDAHO_EPSCOR/GRIDMET / USGS STAC) 
  → precip (mm/d), Tmax/Tmin (K), ETo/ETr, VPD 
  → daily, 1979–present, ~4 km 
  → Cloud-Optimized GeoTIFF / zarr 
  → data_fetchers/gridmet.py; zonal stats via rioxarray 
  → meteorological features (basin aggregates)
```

```text
Washington DOE (ecology.wa.gov) statutory + declaration context 
  → drought declarations (2023–2026 Yakima), 75%-of-normal rule, WSAC minutes 
  → event-driven 
  → CSV/JSON (curated) 
  → risk expression 
  → policy threshold for risk score + demo credibility
```

### Strong enhancement data

```text
NOAA/NCEI nClimGrid SPI/SPEI (netCDF) → SPI-3/6/12/24 → monthly, 1895–present → netCDF/Parquet → feature builder → long-history drought features + backtest context
```

```text
NASA SPoRT-LIS soil moisture percentile (GeoTIFF) → 0–100 cm VSM percentile → daily (6-h model cycle) → COG/Parquet → soil layer → soil-moisture feature + dashboard state
```

```text
NWRFC ESP water-supply forecasts (XML permalink + Forecast Report CSV) → seasonal natural/regulated volumes, % of normal, exceedance → daily ESP / seasonal → CSV → scenario engine → probabilistic supply scenarios for optimizer
```

```text
USGS HYSWAP (Python) → computed daily percentiles/exceedance probs → on-the-fly → Parquet → stats module → replaces WaterWatch percentile computation, avoids deprecated dependenc 
```

### Nice-to-have data

```text
SNODAS (NSIDC g02158 / Climate Engine) → gridded SWE/snow depth → daily → COG → snow context layer (overwintering snow-model cross-check)
```

```text
CPC drought outlooks + QPF (tiles) → probability classes → 7-day/8–14-day/monthly/seasonal → GeoJSON/tiles → dashboard scenarios → narrative controls ("next 30 days drier")
```

```text
USDM HUC8 statistics (REST API) → D0–D4 %-area of Yakima HUCs → weekly → CSV → dashboard context panel (NOT a model label)
```

```text
USDA FSA disaster designations (GeoJSON) → county-level disaster status → weekly → JSON → context layer (agri-drought policy)
```

---

## 6. Modeling Implications

**Target variable (first ML model).** Recommend **season's April–September naturalized runoff volume** at a Yakima index gauge (natural/computed flow, e.g., computed natural flow from Hydromet, or a long-record USGS gauge on an unregulated reach), expressed as a volume (or percent of 1991–2020 normal to mirror state usage). Alternative/parallel target with direct decision value: **USBR TWSA-based junior-right allocation %** — but the sample (annual values, ~2023–2026 for extreme years) is far too small to train on; use it only as a calibration anchor. A pragmatic first deliverable is a **weekly or monthly streamflow forecast** (next 1–3 months) at the index gauge — enough data (~30+ years), decision-relevant, and easy to validate against observed flow.

**Candidate input features** (as-of-date, censored to avoid leakage):
- SNOTEL basin SWE, SWE % of 1991–2020 median, peak-SWE date.
- GridMET/NRCS water-year-to-date precipitation and % of normal for the basin.
- Winter/spring temperature anomalies (GridMET Tmax) — melt timing.
- Antecedent flow (30/90-day mean discharge) at the index gauge.
- Reservoir storage (AF and % capacity, May 1 and July 1).
- SPI-3/6/12 from nClimGrid; optionally MIDI/SWE percentiles.
- ETo/EDDI-type evaporative-demand aggregate for the summer block (for refill/peaking constraints).
- Only data known by the forecast date may be used (respect the documented latencies: GridMET +3–4 days; nClimGrid ~month-end; SNOTEL next-day).

**Simple baseline.** The **NRCS April–September water-supply forecast** (regression-based, official; <https://www.nrcs.usda.gov/programs-initiatives/sswsf-snow-survey-and-water-supply-forecasting-program>) — score your model against it. A pure-stats fallback is "climatological median for the season" or the classic **SWE-on-April-1 linear regression to April–Sep volume** (the historical basis of yield forecasting). Adding "persistence" (last month's flow) completes a three-baseline set.

**Stronger model to try later.** Gradient-boosted trees (LightGBM/XGBoost) with lagged and seasonal features, then a small recurrent/sequential model (e.g., a compact LSTM or a quantile-gradient-boosting ensemble) for quantile/probabilistic bands. Use **quantile regression** so the output can feed the optimizer's scenarios directly. Avoid an LLM as the forecaster.

**Evaluation metrics.** Streamflow volume: NSE, KGE, RMSE/MAE (and % error vs normal), percent-bias (important for allocation), and P10–P90 interval coverage for quantile outputs. Category/declaration framing: accuracy vs the state's 75%-of-normal rule, with PR-oriented metrics at the threshold. Report each metric on water years and "drought vs wet" subsets.

**Backtesting avoiding time leakage.** Strict **walk-forward, origin-based (expanding-window) evaluation**: for each forecast origin `t`, train only on data with labels ≤ `t − k` (k = horizon + full lead/latency), predict the future, move on. Never shuffle; never standardize with full-period statistics; feature matrices must be constructed from data-as-known-at-origin (e.g., don't mix in late-revised final GridMET values). Consider purged/embargoed CV if horizons overlap. Validate season-by-season (Apr–Sep blocking).

**How drought-monitor categories should and should not be used.**
- **Should:** USDM D0–D4 % area (HUC8) as *one* contextual feature; as a dashboard status layer; as a sanity check against your risk score.
- **Should not:** as the sole training label; as allocation ground truth; as a substitute for measured supply in the optimizer's constraints; as the trigger logic for "shortage" (the state uses 75% of normal supply, not USDM).

---

## 7. Optimization Implications

The available data map directly onto a **water-allocation optimizer** (LP/MILP or a scenario parser over the proration formula). Recommended formulation ingredients:

- **Supply estimate.** Reservoir storage (USBR Hydromet, AF) + modeled/observed natural inflow (USBR computed natural flow `QU`, or USGS gauge flow on unregulated reaches) + snowmelt contribution implied by SWE. The whole-system supply anchor is USBR's **TWSA forecast** (April–September available water), which directly determines junior-right % via proration (<https://www.usbr.gov/pn/hydromet/yakima/>; Ecology's prorationing explanation at <https://ecology.wa.gov/water-shorelines/water-supply/water-availability/yakima-basin>).
- **Demand estimates.** (a) Irrigation: district entitlements — Roza, Kittitas Reclamation, Wapato, Sunnyside, etc. (see the *Acquavella*-based distribution described by Yakima Basin Integrated Plan, <https://yakimabasinintegratedplan.org/drought/>). (b) Municipal: city of Yakima and valley communities. (c) Environmental: minimum instream-flow targets (e.g., the Yakima River at Parker flow requirement that triggers storage control per Ecology). **Most demand figures will need reasonable, documented assumptions** — see the explicit list at the end of this section.
- **Minimum environmental-flow constraint.** Enforce minimum flows at key reaches (Parker, Umtanum, Prosser/Kiona) as hard constraints derived from NWS/state flow targets; the state operational trigger is that reservoir releases must sustain the Yakima at Parker minimum flows (Ecology).
- **Municipal priority constraint.** Model municipal demand as high-priority senior (pre-1905) supply that must be satisfied before proratable supply — consistent with prior appropriation.
- **Agricultural allocation.** Allocate within the proratable pool; model the documented rule that **shortage is shared equally across junior/proratable rights** after senior (and Yakama Nation) entitlements are fully served (USBR practice, confirmed in multiple releases, e.g., <https://www.usbr.gov/newsroom/news-release/5315>).
- **Fairness / shortage penalty.** Optimize a lexicographic or weighted objective: satisfy seniors → min environmental-flow shortfall → minimize the *maximum* curtailment rate among remaining users → minimize total volume shortage; add a fairness metric (e.g., deviation from equal-share proration, or a Gini/Cohen-vacha-type index over districts). Emphasize the trade-off frontier, not a single optimum.
- **Uncertainty / scenario analysis.** Re-optimize under (i) different TWSA forecast percentiles, (ii) NWRFC ESP exceedance scenarios (wet/normal/dry), (iii) historical analog years, and (iv) user-set demand multipliers. Output is a set of allocations with ranges — matching how the message "seniors 100% / juniors 52–58%" is framed in real USBR releases.

**Demand values that will require reasonable documented assumptions for a hackathon prototype** (label these clearly in the UI):
- District-level irrigation entitlements and consumptive use (published totals exist in USBR/YBIP materials but not as a clean open API — pull what you can, assume the rest).
- Municipal water demand by month.
- Environmental instream-flow targets (numbers exist in NWS/Ecology documents but are not in a single machine-readable API).
- Return flows (irrigation efficiency → return to river) — must be assumed.
- Crop/acreage demand curves by month (WSU AgWeatherNet evaporative data can support ET-based estimates as an enhancement).

---

## 8. Dashboard and Demo Implications

Recommended Streamlit components, each backed by a real, verifiable source:

- **Basin map (interactive).** Plotly/folium layer: USGS gauges (from the NWIS site inventory filtered to the Yakima), the 5 USBR reservoirs, and SNOTEL sites (AWDB metadata). Color-code gauge markers by current USGS percentile you compute with HYSWAP.
- **Drought-risk score.** Compose a transparent index (e.g., weighted blend of: reservoir storage % capacity, basin SWE % of median, SPI-6, SPoRT-LIS soil-moisture percentile, USDM HUC8 %-in-D1+), each normalized to 0–100, with the weight scheme shown next to the score. Calibrate the score's "alert" band to Ecology's 75%-of-normal logic, not to the USDM.
- **Streamflow and rainfall trends.** Dual-pane time series: observed daily discharge vs computed normal/percentile band (HYSWAP), plus GridMET basin precipitation anomaly bars with 1991–2020 normal line.
- **Soil-moisture status.** SPoRT-LIS 0–100 cm percentile dial per sub-basin polygon (zonal mean from GeoTIFF).
- **Snowpack panel.** Basin SWE % of median at SNOTEL sites with a date slider (water-year envelope from AWDB period-of-record data).
- **Scenario controls.** Sliders for precipitation/precip-scenario, snowpack multiplier, TWSA percentile (from NWRFC ESP or USBR forecast), and demand multiplier → rerun the optimizer.
- **Allocation recommendation.** Bar chart per user group (senior, junior/proratable, municipal, Yakama Nation; or per district) showing **entitlement vs recommended allocation vs shortage %**, with the equal-share proration line.
- **Explanation of trade-offs.** Trade-off frontier: x = total agricultural shortage (AF), y = environmental deficit (AF below instream target), point = municipal reliability; hover over a point to see the allocation.
- **Data-source transparency panel.** For every chart: dataset name, provider, update date, latency, provisional flag, and a link to the official source; plus a "data valid" footer that mirrors drought.gov's own convention.

---

## 9. Legal, Licensing, and Reliability Notes

- **USGS.** Public domain (U.S. Government work). Required attribution nominally "U.S. Geological Survey" (standard citation practice). **Caveats:** streamflow data are **provisional** and subject to revision (<https://waterdata.usgs.gov/provisional-data-statement/>); regulated reaches embed reservoir/diversion effects; WaterWatch (the percentile engine behind the drought.gov streamflow map) is being **decommissioned** — compute percentiles yourself with `HYSWAP` (<https://waterdata.usgs.gov/blog/wdfn-stats-delivery/>). Reliability high; suitable for operational decision support as a *measured* input, but never treat provisional current-year values as final.
- **NOAA/NIDIS (Drought.gov).** Data disseminated under the **NOAA Open Data Dissemination (NODD)** program; publicly accessible, anonymous HTTPS reads from Google Cloud Storage with no login (<https://www.drought.gov/data-download>). NOAA disclaimer applies (<https://www.noaa.gov/disclaimer>). Reliability high for context; tapes/flags like "Experimental" (MIDI) must be honored. Suitable for dashboards and features; treat derived indices as products of a model chain.

- **USDM (NDMC/USDA/NOAA/NASA).** Free to use; **cite NDMC, USDA, and NOAA** (<https://droughtmonitor.unl.edu/DmData/GISData.aspx>). Caveats: expert-blended; gridded 500 m version is unofficial; pre-2004 archives carry ~4-mile digitization error. Reliable as a **situational awareness** product, not as a water-budget measurement.
- **NRCS SNOTEL/AWDB.** USDA public data; reports are stamped **"Provisional data, subject to revision"** (<https://wcc.sc.egov.usda.gov/reports/UpdateReport.html?report=Washington>). Sensor artifacts (e.g., midwinter melt, sensor freeze) appear in WTEQ. Suitable for operational planning used with the caveat that current-season values are provisional.
- **GridMET (UC Merced / Climatology Lab).** Free, **pro-bono, semi-operational**; cite Abatzoglou (2013): <https://doi.org/10.1002/joc.3413>. Data carry `early → provisional → permanent` status tiers in Earth Engine; near-real-time values can be superseded. Reliable for features with the owner's own caution about operational guarantees.
- **NASA products (SPoRT-LIS, SMAP/Crop-CASMA).** Open data under **EOSDIS Data Use guidance**; cite the product DOI (e.g., 10.5067/SPORT/LIS/DATA101). Modeled/assimilated, not purely observational; short records (2015–). Best used as enhancement, not sole evidence.
- **USBR Hydromet / RISE.** Public data; all Hydromet values are **"PROVISIONAL — SUBJECT TO CHANGE"** (<https://www.usbr.gov/pn/hydromet/yakima/yaktea.html>). TWSA/proration **forecasts** are operational products revised monthly — treat as forecasts with real uncertainty. Suitable for prototype demos and student use; verify against published releases before any operational claim.
- **WA Department of Ecology & WSAC.** Public records; drought determinations under RCW 43.83B are legal/policy artifacts (orders are PDFs). Reliability high for policy ground truth; not a measurement stream.
- **Overall:** every source above is a government or academic product with terms permissive for a hackathon prototype. **Student prototype, not operational-commitment use.** Mark all provisional data in the UI; withhold any claim of "operational" status.

---

## 10. Final Recommendation

1. **Should BasinSignal use Drought.gov?** Yes — but as **aggregated context, derived-index features, and a discovery layer**, not as the primary hydrologic data backbone. The backbone must be the primary producers.
2. **Exactly how to use it.** (a) Pull USDM D0–D4 %-area at HUC8 for the Yakima via the REST statistics API (weekly, for dashboard and one feature); (b) use its nClimGrid SPI/SPEI netCDFs and GridMET-derived COG tiles for features; (c) mine its "Washington" resource page and PNW DEWS portal to keep the state/national outlook narrative current; (d) never compute an allocation decision directly from a drought category.
3. **Top 3 external datasets to acquire next.**
   1. **USGS daily streamflow** at 6–10 curated Yakima gauges (incl. USGS 12484500 at Umtanum) via `dataretrieval` — target + supply.
   2. **NRCS SNOTEL SWE + basin SWE index** for Yakima HUC4 1703 via AWDB REST — seasonal predictor.
   3. **USBR Hydromet reservoir storage (5 reservoirs) + the official monthly TWSA/proration numbers** — optimizer constraints and ground truth.
4. **Best first implementation scope (hackathon).** A Streamlit MVP that (i) fetches the three datasets above daily into Parquet, (ii) computes a transparent basin drought-risk score, (iii) produces a 1–3-month streamflow forecast with a walked-forward baseline vs NRCS, and (iv) runs a senior-first / equal-share-proration advisor with scenario sliders and a trade-off plot.
5. **Single highest-risk technical assumption.** That **2019–2026-era "drought" behavior at the Yakima — a short, extreme, snow-drought-dominated record — generalizes enough to train a model usable for allocation advice.** Mitigate by centering targets on % of normal, embedding the 75%-of-normal rule, and presenting quantile bands rather than point forecasts.

---

## Action Plan for the Next 48 Hours

**Hours 0–6: data acquisition spike**
1. Register for the free **USGS Water Data API key** (<https://api.waterdata.usgs.gov/signup/>); install `dataretrieval` and `hyswap`.
2. Pull daily flow (00060/00003) for USGS 12484500 + 4–6 other Yakima gauges (pick via <https://waterdata.usgs.gov/state/Washington/>); save to `data/raw/usgs/`.
3. Pull SNOTEL WTEQ/PREC for 8–12 Yakima sites via AWDB REST; save normals (1991–2020) and period-of-record.
4. Scrape USBR Hydromet daily reservoir storage (5 reservoirs) via RISE/classic pages; assemble the 2023–2026 junior-right % release numbers into a tiny CSV.

**Hours 6–18: pipeline and baseline**
5. Build `src/data_fetchers/` (usgs, nrcs, usbr, gridmet) with a common `fetch→validate→parquet` interface; add `tests/` smoke tests for non-empty, monotonic-index checks.
6. Compute basin zonal means from GridMET (or PRISM % of normal) and SPI-3/6/12 from the nClimGrid NetCDFs.
7. Implement the baseline (climatological median + SWE-Apr-1 regression) and the walk-forward harness; compute NSE/RMSE/Pbias on out-of-sample seasons.

**Hours 18–30: optimizer + risk score**
8. Model the proration logic: seniors→Yakama→juniors equal-share, with Parker/valley minimum-flow constraints; output recommendation and a Gini-type fairness metric.
9. Wire the transparency panel (every chart shows source, data-valid, provisional flag, link).

**Hours 30–42: Streamlit MVP**
10. Stand up the radar pages (basin map, risk score, flow/snow/soil panels, scenario sliders, allocation bars, trade-off plot) with the fetched data.

**Hours 42–48: verify, label, demo**
11. Re-check every claim/citation against the official pages listed in this report; mark all provisional and "Needs verification" items visibly; rehearse the 3-minute demo (risk score → scenario → allocation → caveats).

*End of report.*