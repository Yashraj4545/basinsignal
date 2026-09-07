# Yakima Basin Data-Acquisition Specification

**Prepared:** 2026-09-06 · **Author:** BasinSignal research (Water Resource Management track)
**Target basin:** Yakima River Basin, Washington (HUC8s 17030001, 17030002, 17030003)
**Intended stack:** Python · Streamlit · modular backend · ML/data science · optimization

> **Method note.** Every endpoint, parameter name, unit, station ID, and example response below was
> captured from a **live request in the 2026-09-06 verification session** unless flagged
> **"Needs verification."** Nothing is invented. URLs marked VERIFIED returned real data when tested;
> anything inferred from page source or search results is explicitly labeled.

---

## 1. Executive Summary

Three federal agencies provide the primary, basin-scale inputs BasinSignal needs for the Yakima:

| Need | Source | Primary endpoint | Status |
|---|---|---|---|
| Daily streamflow (historic + current) | USGS | `api.waterdata.usgs.gov` OGC-API `daily` collection | VERIFIED |
| Snowpack / SWE / precip / temp | NRCS | AWDB REST (`wcc.sc.egov.usda.gov`) | VERIFIED |
| 1991–2020 normals | NRCS | Report Generator (not the REST API) | **Needs verification** (exact URL) |
| Reservoir storage / outflow | USBR | `pn-bin/daily.pl` (historical) + `pn-bin/instant.pl` (current) | VERIFIED |
| System water-year storage & avg | USBR | `pn-bin/v1/wyreport.pl?format=csv-analysis` | VERIFIED |
| Daily system status / proration % | USBR | `hydromet/yakima/yakstats.txt` | VERIFIED |
| TWSA proration forecast | USBR | News releases + yakima basin pages (official) | Partially verified (text); machine-readable table **Needs verification** |

**Key decisions for the prototype**
1. **USGS:** use the modern `api.waterdata.usgs.gov` OGC-API (legacy `waterservices.usgs.gov` is being
   retired in early 2027). No API key is required to start; keys are optional and raise rate limits.
2. **USBR reservoir inflow:** the archive's computed inflow column (`ID`) is **empty**; use the
   **unregulated-flow metric `QU`** (computed natural flow, cfs) as the naturalized inflow series plus
   `QD` (release/discharge, cfs), `AF` (storage, acre-feet), `FB` (forebay, ft).
3. **All USBR Hydromet data are PROVISIONAL** (subject to change). Archive as snapshots with timestamps.

---

## 2. Source × Endpoint × Variables Table

| # | Source | Endpoint / URL (base) | Variables | Units | Frequency | History | Auth / key | Format |
|---|---|---|---|---|---|---|---|---|
| 1 | USGS | `https://api.waterdata.usgs.gov/ogcapi/v0/collections/daily/items` | Daily mean discharge (`parameter_code=00060`, `statistic_id=00003`) | ft³/s | 1 day | 1904/1905/1908–present (per gage) | Optional free key (429 throttling without) | CSV (`f=csv`) or GeoJSON (`f=json`) |
| 2 | USGS | `.../collections/monitoring-locations/items` | Site metadata (name, location, drainage area, HUC) | – | static | current | none | GeoJSON |
| 3 | USGS | `.../collections/time-series-metadata/items` | Period of record (`begin`, `end`) per parameter/statistic | – | static | current | none | GeoJSON |
| 4 | NRCS | `https://wcc.sc.egov.usda.gov/awdbRestApi/services/v1/data` | `WTEQ` SWE, `PREC` precip (Oct–1-accumulated), `TMAX`, `TMIN` | in, in, °F (stored) / °C (original) | 1 day | WTEQ from 1978-10-01, TMAX/TMIN from 1988-09-30 | none | JSON |
| 5 | NRCS | `.../v1/stations` | Station list / triplets / elevations | – | static | current | none | JSON |
| 6 | NRCS | `.../v1/reference-data` | Valid DCOs, durations, elements, units | – | static | current | none | JSON |
| 7 | NRCS | Report Generator `https://wcc.sc.egov.usda.gov/reportGenerator/` | 1991–2020 normals (Medians official; + Avg/Max/Min/SD) | in, °F | monthly | 1991–2020 | none | HTML/CSV | 
| 8 | USBR | `https://www.usbr.gov/pn-bin/daily.pl` | `AF` storage, `FB` forebay, `QD` release, `QU` unregulated flow (ID empty) | AF, ft, cfs | 1 day | Reservoir archive (per reservoir) | none | CSV |
| 9 | USBR | `https://www.usbr.gov/pn-bin/instant.pl` | same variables at 15-min; incl. `yaksys af` system total | AF, ft, cfs | 15 min | ~current window (back in hours) | none | CSV / HTML |
| 10 | USBR | `https://www.usbr.gov/pn-bin/v1/wyreport.pl?format=csv-analysis` | Current Year, Previous Year, Average (10-yr) storage | AF | 1 day | ~2000–present | none | CSV |
| 11 | USBR | `https://www.usbr.gov/pn/hydromet/yakima/yakstats.txt` | Per-reservoir FB/AF/capacity/%/inflow/releases, diversion totals, river flows, **current proration % + TWSA date** | AF, cfs, % | 1 day (publish ~12:27) | rolling | none | plain text |
| 12 | USBR | `https://www.usbr.gov/newsroom/news-release/...` | Monthly TWSA / proration forecasts (senior & junior %) | % | monthly (Mar–Sep) | 2026 season doc'd; earlier via page | none | HTML/text |

---

## 3. Detailed Verified Specifications

### 3.1 USGS — daily mean streamflow

Modern endpoint (VERIFIED):
```
https://api.waterdata.usgs.gov/ogcapi/v0/collections/daily/items
   ?monitoring_location_id=USGS-12484500
   &parameter_code=00060
   &statistic_id=00003
   &datetime=2024-04-01/2024-04-10
   &limit=5
   &f=csv
```
Returns CSV columns incl. `time`, `value` (ft³/s), `unit_of_measure`. `limit` pages results
(`limit` required; paginate with `limit`/`offset` or per-year pulls). Passing `f=json` returns GeoJSON
features. **Do not** pass `fields=` (returned HTTP 400). Legacy `waterservices.usgs.gov/nwis/site` with
`huc`/`parameterCd`/`siteType=ST` returned 400 — use the modern collection instead; legacy service is
being decommissioned early 2027 anyway.

**API key (optional):** signup at `https://api.waterdata.usgs.gov/signup/` (api.data.gov-managed).
Without a key requests are throttled with HTTP 429s; obtain a key for bulk 120-year history pulls.
Exact throttle thresholds **Need verification** (see §7).

**Recommended active daily-flow gages (verified period of record via `time-series-metadata`):**

| Site ID | Name | Daily mean 00060 PoR | Drainage area | HUC |
|---|---|---|---|---|
| USGS-12484500 | Yakima River at Umtanum | 1908-10-01 → present (end ≥ 2026-09-04) | 1,594 mi² | 170300010705 |
| USGS-12510500 | Yakima River at Kiona | 1905-10-01 → present | 5,615 mi² | 170300031203 |
| USGS-12488500 | American River near Nile | 1909-04-25 → present | 78.9 mi² | 170300020107 |
| USGS-12502500 | Ahtanum Creek at Union Gap | 1904-05-01 → present | 173 mi² | 170300030105 |
| USGS-12492900 | Oak Creek above Tieton R nr Naches | begins **2026-05-09** (new gage) | – | 170300020307 |

**Discontinued daily gages** (appear only in `time-series-metadata` as historical; data ends listed):
12479500 Cle Elum (→1990-03-31), 12499000 (→1990), 12480000 Teanaway (→1973), 12494000 Naches below
Tieton (→1979); reservoir-inflow gages 12479000, 12474500, 12476000, 12488000, 12491500 (→1978-09-29);
12505001 Parker is a flow-model time series (no DV). These matter only if reconstructing long inflow
records; the USBR archive covers reservoir storage itself.

Station discovery: `monitoring-locations` collection with `monitoring_location_number=` or
`hydrologic_unit_code=17030001` (also /02, /03); period-of-record via `time-series-metadata` items with
`parameter_code=00060&statistic_id=00003&monitoring_location_id=USGS-...`.

### 3.2 NRCS — SNOTEL daily data (AWDB REST)

Open REST base `https://wcc.sc.egov.usda.gov/awdbRestApi/services/v1/` (no auth). Verified working:

- `/stations` — filters use camelCase: `stationTriplets`, `hucs`, `elements`, `stationNames`,
  `durations`, `activeOnly`.
- `/data` — VERIFIED example:
```
https://wcc.sc.egov.usda.gov/awdbRestApi/services/v1/data
   ?stationTriplets=375:WA:SNTL,642:WA:SNTL
   &elements=WTEQ,PREC,TMAX,TMIN
   &duration=DAILY
   &beginDate=2024-03-01
   &endDate=2024-03-04
```
  Returns JSON array of `{stationTriplet, data:[{stationElement:{elementCode, ordinal, durationName,
  storedUnitCode, originalUnitCode, beginDate, endDate}, values:[{date, value}]}]}`.
- Wrong names that 400: `stationIds`, `elementCds`, `elementCd`; correct names as above.
- `/reference-data?referenceLists=...` valid lists: `dcos, durations, elements, forecastPeriods,
  functions, instruments, networks, physicalElements, states, units`.

**Units (verified):** `WTEQ` in, `PREC` in (accumulated since Oct 1 — March values ~45 in),
`TMAX`/`TMIN` °F stored, °C as original units.

**Yakima SNOTEL stations (all triplet-form `NNN:WA:SNTL`, network SNTL, in the 3 HUCs):**

| Triplet | Name | Elev (ft) |
|---|---|---|
| 375:WA:SNTL | Bumping Ridge | 4,600 |
| 478:WA:SNTL | Fish Lake | 3,440 |
| 502:WA:SNTL | Green Lake | 5,920 |
| 507:WA:SNTL | Grouse Camp | 5,390 |
| 599:WA:SNTL | Lost Horse | 5,100 |
| 642:WA:SNTL | Morse Lake | 5,400 |
| 692:WA:SNTL | Pigtail Peak | 5,800 |
| 734:WA:SNTL | Sasse Ridge | 4,340 |
| 863:WA:SNTL | White Pass E.S. | – |

Also in-basin non-SNTL pillows (network `SNOW`): 21C36 Bumping Lake New, 21B04 Fish Lake,
21B14 Dommerie Flats, 21B08 Tunnel Avenue. **Filtering note:** `hucs=` alone does NOT restrict to a
network; pass `stationTriplets=*:*:SNTL` to get the SNOTEL network only.

Element PoR (verified for 375): WTEQ begins 1978-10-01; TMAX/TMIN begin 1988-09-30.

### 3.3 NRCS — 1991–2020 normals

Normals are **not** available through the AWDB REST API. Official normals come from the Report
Generator, `https://wcc.sc.egov.usda.gov/reportGenerator/`; the official normal is **Median (1991-2020)**,
with Average, % of Median, Max/Min also available. The exact self-service URL for the 9 stations'
SWE/precip/temp normals (e.g., median WTEQ by month, Oct 21–Aug 31) **Needs verification** — generate it
once manually and freeze the URL(s) in `data/raw/nrcs_normals/` (see §7).

### 3.4 USBR Yakima — historical reservoir data (`daily.pl`)

VERIFIED:
```
https://www.usbr.gov/pn-bin/daily.pl?format=csv&flags=false&description=true
   &station=BUM&year=2024&month=7&day=1&year=2024&month=7&day=3
   &pcode=AF&pcode=FB&pcode=QD&pcode=ID&pcode=QU
```
Returns:
```
DateTime,bum_af,bum_fb,bum_qd,bum_id,bum_qu
2024-07-01,30743.94,3423.68,186.00,,192.57
...
```
The duplicated `year`/`month`/`day` groups act as start (`first group`) and end (`second group`).
Same group = single day/range start. Example verified values (2024-10-01): KEE af=10401.00 fb=2433.27
qu=31.47 qd=77.41; KAC af=46166.28 fb=2209.34 qu=208.35 qd=634.53; CLE af=16130.00 fb=2118.06 qu=66.40
qd=202.00; RIM af=29169.88 fb=2827.15 qu=196.09 qd=1334.06; BUM af=8078.00 fb=3401.04 qu=36.91 qd=170.00.

**`ID` (computed reservoir inflow) is empty for all five reservoirs in the archive.** Use `QU`
(unregulated flow, "Estimated Daily Average", cfs) as the naturalized-inflow series.

Archive parameter codes (`hydromet_pcodes.html`): `AF` storage acre-feet · `FB` forebay ft · `Q`
discharge cfs · `QC` canal discharge cfs · `GH` gage height ft · `CH` canal gage height · `PC` cumulative
precip in · `OB` air temp °F · `SP` SWE in · `WF` water temp °F · `QU` unregulated flow cfs ·
`ID` reservoir inflow cfs (blank in archive; see §7 for realtime).

Reservoir station codes: **BUM** (Bumping), **CLE** (Cle Elum), **KAC** (Kachess), **KEE** (Keechelus),
**RIM** (Rimrock). Other basin codes from `yakwebarcread.html`: EASW, ELNW, KESW, NACW, PARW, RBDW,
KIOW, QSPW, CSPW, CPPW, MLSW, BRGW (some are SNOTEL mirrors).

### 3.5 USBR — current data (`instant.pl`)

VERIFIED:
```
https://www.usbr.gov/pn-bin/instant.pl?list=bum af,bum fb,rim q,rim af,yaksys af
   &format=csv&flags=false&description=false&interval=1&back=3
```
Returns 15-min rows:
```
DateTime,bum_af,bum_fb,rim_q,rim_af,yaksys_af
2026-09-06 21:00,14101.96,3408.38,1090.00,136000.00,
...
```
`yaksys` is the whole-Yakima-system pseudo-station (teacup total). Current storage is also displayed on
the teacup page `https://www.usbr.gov/pn/hydromet/yakima/yaktea.html` (total system capacity
1,065,670 AF, PROVISIONAL banner).

### 3.6 USBR — water-year graph CSV (`wyreport.pl`)

VERIFIED:
```
https://www.usbr.gov/pn-bin/v1/wyreport.pl?site=yaksys&parameter=af&format=csv-analysis
```
Returns `DateTime,Current Year,Previous Year,Average` (Average = 10-yr), from 2000/10/01. Works with
reservoir site codes too (`site=rim`). Useful for dashboard context without extra averaging.

### 3.7 USBR — daily system status & proration (`yakstats.txt`)

VERIFIED plain-text daily report:
```
https://www.usbr.gov/pn/hydromet/yakima/yakstats.txt
```
Contains per-reservoir FB/Content-AF/total-AF/% capacity/Inflow-cfs/Releases-cfs + system totals,
irrigation diversions by canal, river flows at all key gages, unregulated tributary flow above Parker,
storage % of 1991-2020 average, and the **current proration rate** ("The current proration rate for all
Junior Water Right Holders is 60% based on the SEP03, TWSA."). HTML sibling `yakstats.html`.
Parse this daily as a snapshot for the dashboard.

### 3.8 USBR — TWSA / proration forecast

Official monthly "Total Water Supply Available" forecasts are issued ~the 1st week of Mar–Sep via USBR
news releases (`https://www.usbr.gov/newsroom/news-release/[id]`; e.g., Sep 3 2026 release). Each states
senior/junior proration % for the season and cites the as-of date. 2026 season junior values ran 44%→52%
(Mar→Apr/May/June)→56% (Jul)→58% (Aug)→60% (Sep). The current junior proration % is also restated daily
in `yakstats.txt`. **Whether a machine-readable TWSA table (CSV) is posted on
`https://www.usbr.gov/pn/hydromet/yakima/` Needs verification** — the news-release text is the verified
fallback. Do **not** confuse this with the seasonal-runoff forecast (Apr–Sep %of-average); TWSA is the
allocation signal BasinSignal needs.

### 3.9 USBR — RISE API (secondary / not in v0 pipeline)

Reclamation's RISE API (`https://data.usbr.gov/rise-api/` — Swagger/OpenAPI UI; programmatic JSON:API
base `https://data.usbr.gov/rise/api/`) exposes `/location`, `/catalog-item`, `/catalog-record`,
`/result`, `/parameter`, `/model-run`, `/reclamation-region`. A `filter[state]=WA&filter[locationType]=RES`
location query returned HTTP 200 (JSON:API). Verified presence only; not load-bearing for the prototype.
Revisit for quality-controlled reservoir storage if Hydromet provisional data becomes limiting.

---

## 4. Download Checklist

Pipeline scripts should implement, in order:

1. **USGS daily mean flow** (`00060`/`00003`) full PoR per active gage (12484500, 12510500, 12488500,
   12502500, +optional 12492900) → `data/raw/usgs_daily/`.
2. **USGS site metadata** (name, drainage area, HUC, location) per gage → `data/raw/usgs_metadata.json`.
3. **NRCS SNOTEL daily** `WTEQ,PREC,TMAX,TMIN` for the 9 triplets, full PoR (WTEQ 1978→, TMAX/TMIN
   1988→) → `data/raw/nrcs_snotel/`.
4. **NRCS 1991–2020 normals** (median/avg SWE + precip + temp by month) → `data/raw/nrcs_normals/`
   (freeze URLs after manual generation).
5. **USBR reservoir daily archive** `AF,FB,QD,QU` for BUM/CLE/KAC/KEE/RIM, full range → `data/raw/usbr_daily/`.
6. **USBR system water-year CSV** `yaksys af` (+ per-reservoir `af`) → `data/raw/usbr_wygraph/`.
7. **USBR current snapshot** `instant.pl` fetch (15-min, last N hours) at each crawl → `data/raw/usbr_current/`.
8. **USBR daily status** `yakstats.txt` snapshot at each crawl → `data/raw/usbr_status/`.
9. **TWSA forecast records**: news-release text + proration % (manual/periodic) → `data/raw/twsa/`.
10. **Write/extend a manifest** (`data/raw/manifest.csv`: source, endpoint, params, retrieval datetime,
    row count, provisional flag) for provenance and re-crawl.

Suggested cadence: historical bulk (steps 1–6) once on setup; daily crawl for steps 7–8; monthly for 9
during Mar–Sep; normals regenerated never (static).

---

## 5. `data/raw/` Folder Structure

```
data/
└── raw/
    ├── README.md                  # manifests, fetch recipes, provenance notes
    ├── manifest.csv               # every pull: source, url, params, retrieved_at, n_rows, provisional
    ├── usgs_daily/
    │   ├── USGS-12484500_00060_00003.csv   # one file per gage, full PoR
    │   ├── USGS-12510500_00060_00003.csv
    │   ├── USGS-12488500_00060_00003.csv
    │   ├── USGS-12502500_00060_00003.csv
    │   └── USGS-12492900_00060_00003.csv   # optional (PoR starts 2026-05-09)
    ├── usgs_metadata.json
    ├── nrcs_snotel/
    │   ├── 375_WA_SNTL_WTEQ.csv  ...     # one per station × element
    │   ├── 375_WA_SNTL_TMAX.csv
    │   └── ...
    ├── nrcs_normals/
    │   └── 1991-2020_median_<element>_<triplet>.csv
    ├── usbr_daily/
    │   ├── BUM_af_fb_qd_qu.csv   ├── CLE_...   ├── KAC_...   ├── KEE_...   └── RIM_...
    ├── usbr_wygraph/
    │   └── yaksys_af_wateryear.csv   (+ rim_af_wateryear.csv ...)
    ├── usbr_current/
    │   └── snapshots/ inst_YYYYMMDD_HHMM.csv
    ├── usbr_status/
    │   └── yakstats_YYYYMMDD.txt
    └── twsa/
        └── news_<date>.txt + twsa_series.csv   # senior/junior % x issue date
```

---

## 6. Example Commands (copy-paste; PowerShell 5.1 / .NET webclient shown)

```powershell
# 1. USGS daily mean flow, one gage, one year (page with limit)
$u='https://api.waterdata.usgs.gov/ogcapi/v0/collections/daily/items?monitoring_location_id=USGS-12484500&parameter_code=00060&statistic_id=00003&datetime=2024-01-01/2024-12-31&limit=1000&f=csv'
Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 120 -OutFile 'data/raw/usgs_daily/USGS-12484500_2024.csv'

# 2. NRCS SNOTEL daily data, one call, multiple triplets/elements
$u='https://wcc.sc.egov.usda.gov/awdbRestApi/services/v1/data?stationTriplets=375:WA:SNTL,642:WA:SNTL&elements=WTEQ,PREC,TMAX,TMIN&duration=DAILY&beginDate=2024-03-01&endDate=2024-03-04'
(Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 60).Content | Out-File -Encoding utf8 'data/raw/nrcs_snotel/375_642_daily.json'

# 3. USBR historical reservoir daily archive, CSV
$u='https://www.usbr.gov/pn-bin/daily.pl?format=csv&flags=false&description=true&station=BUM&year=2024&month=7&day=1&year=2024&month=7&day=3&pcode=AF&pcode=FB&pcode=QD&pcode=QU'
Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 60 -OutFile 'data/raw/usbr_daily/BUM_demo.csv'

# 4. USBR current 15-min data (whole-system total + reservoir storage)
$u='https://www.usbr.gov/pn-bin/instant.pl?list=yaksys af,bum af,bum fb,rim q&format=csv&flags=false&description=false&interval=1&back=24'
Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 60 -OutFile 'data/raw/usbr_current/inst_$(Get-Date -f yyyyMMdd_HHmm).csv'

# 5. USBR system water-year graph CSV (Current/Previous/Average)
$u='https://www.usbr.gov/pn-bin/v1/wyreport.pl?site=yaksys&parameter=af&format=csv-analysis'
Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 60 -OutFile 'data/raw/usbr_wygraph/yaksys_af_wateryear.csv'

# 6. USBR daily system status snapshot (contains current proration %)
$u='https://www.usbr.gov/pn/hydromet/yakima/yakstats.txt'
Invoke-WebRequest -UseBasicParsing -Uri $u -TimeoutSec 60 -OutFile "data/raw/usbr_status/yakstats_$(Get-Date -f yyyyMMdd).txt"
```

---

## 7. Manual Verification Checklist (cannot be fully scripted)

1. **USGS API-key throttling(429)** exact thresholds with/without a key; decide whether
   `https://api.waterdata.usgs.gov/signup/` registration is needed for the 1904–present bulk pulls.
2. **NRCS normals exact URLs** from Report Generator for the 9 triplets × (Median 1991-2020,
   plus 1991-2020 Average), including SWE median by month and any TMAX/TMIN normals; capture and freeze.
3. **USBR TWSA machine-readable output**: confirm whether a TWSA table (junior %, senior %, TWSA AF)
   is posted at `https://www.usbr.gov/pn/hydromet/yakima/` or if news-release parsing is required;
   archive 2020–2025 releases for back-testing proration %.
4. **`daily.pl` range semantics**: verify behavior over multi-year ranges (e.g., 2000-10-01→2026-09-30
   in one call) vs chunked yearly requests; confirm end date defaults to today; confirm timezone (all
   outputs appear Pacific/local; `instant.pl` is local time).
5. **`instant.pl` realtime inflow**: confirm whether computed inflow (`id`) is populated for reservoirs
   in realtime vs the empty `ID` in the daily archive; confirm max acceptable `back` hours and 15-min gap
   handling.
6. **Capacity discrepancy**: teacup total capacity 1,065,670 AF vs `yakstats.txt` total 1,065,400 AF
   (five-reservoir summations differ slightly); pick the authoritative total for %-full display.
7. **Kiona gage continuity**: confirm 12510500 (PoR 1905→) remains the end-of-basin daily-flow gage or
   whether USGS-12505001 Parker (flow-model TS) is the preferred lower-basin series.
8. **New gage USGS-12492900 Oak Creek**: confirm coverage cadence since 2026-05-09 and any datum/code
   quirks before including it.
9. **SNOTEL White Pass E.S. (863)** observations start date & whether network code is `SNTL` for all nine
   triplets in the normals generator.
10. **RISE reservoir records**: confirm `filter` field names (`filter[locationType]=RES` worked) and
    whether Yakima reservoir storage appears in `/result` for quality-controlled cross-checks.

---

## 8. Provenance & License (as documented by providers)

- **USGS**: public domain; no license/citation requirement beyond site attribution.
- **NRCS/AWDB**: U.S. Government work, public domain; datasets are provisional until final QC; normals
  are official products.
- **USBR Hydromet**: public; **all realtime and archive data are PROVISIONAL** and subject to change —
  snapshot with retrieval timestamps and do not claim finality for operations.
- **TWSA news releases**: Federal public records, public domain.