# data/raw

Raw, unprocessed acquisitions from federal primary sources for the Yakima River Basin prototype.
Every pull is recorded in `manifest.csv` for provenance.

**Authoritative specification:** [`docs/yakima_data_acquisition_spec.md`](../../docs/yakima_data_acquisition_spec.md)

## Sources

| Folder | Source | Endpoint | Content |
|---|---|---|---|
| `usgs_daily/` | USGS | `api.waterdata.usgs.gov/ogcapi/v0/collections/daily/items` | Daily mean discharge (00060/00003), ft³/s |
| `usgs_metadata.json` | USGS | `.../collections/monitoring-locations/items` | Site metadata per gage |
| `nrcs_snotel/` | NRCS | `wcc.sc.egov.usda.gov/awdbRestApi/services/v1/data` | Daily WTEQ/PREC/TMAX/TMIN (in, °F) |
| `nrcs_normals/` | NRCS | Report Generator (manual, frozen URLs) | 1991–2020 medians/averages |
| `usbr_daily/` | USBR | `pn-bin/daily.pl` | Reservoir AF/FB/QD/QU (daily) |
| `usbr_wygraph/` | USBR | `pn-bin/v1/wyreport.pl?format=csv-analysis` | Current/Previous year + 10-yr avg storage |
| `usbr_current/snapshots/` | USBR | `pn-bin/instant.pl` | 15-min current data snapshots |
| `usbr_status/` | USBR | `hydromet/yakima/yakstats.txt` | Daily system status + proration % |
| `twsa/` | USBR | news releases + basin pages | TWSA forecast records |

## Notes

- All USBR Hydromet data are **PROVISIONAL**; snapshot with retrieval timestamps.
- USBR reservoir inflow column `ID` is empty in the daily archive — use `QU` (unregulated flow).
- `manifest.csv` columns: `source,url,params,retrieved_at,n_rows,provisional`.