# chaselights-data-workers

The **public** data plane for [chaselights.app](https://chaselights.app): the
scheduled weather-grid producers live here so the private application repo can
stay private without burning its 2,000 Actions minutes.

## Contract

- This repo owns **`weathergrid/**`** in Cloudflare R2 (`chaselights-data`), served
  at `https://data.chaselights.app/weathergrid/…`.
- It **never commits its output**. Every producer ends with
  `python ops/pages/r2_sync.py --apply --only weathergrid --from-disk
  --files-from /tmp/changed.txt`, so the bucket is the only sink.
- The private repo owns **`weather_details/**`**. `r2_sync.py` fails closed if a
  path falls outside the `--only` scope, so neither side can half-write the
  other's tree.

## Workflows

| workflow | cadence | writes |
| --- | --- | --- |
| `update_weathergrid.yml` | 4×/day | GFS / ICON / CWA WRF compact bundles + PNG frames |
| `update_himawari9_weathergrid.yml` | 10 min | Himawari-9 Taiwan cloud bundle |
| `update_jma_msm_weathergrid.yml` | 3 h | JMA MSM Japan cloud bundle |
| `weathergrid_v2_jma_publish.yml` | 3 h + on `Update JMA MSM WeatherGrid` | V2 immutable tiles |
| `weathergrid_v2_cwa_publish.yml` | on `Update Live WeatherGrid Preview` | V2 CWA derived tiles |
| `cwa_low_cloud_calibration.yml` | 2×/day | CWA low-cloud calibration history |
| `b166_cams_aod_weathergrid.yml` | 2×/day | CAMS AOD / PM2.5 (tw, us) |

## Required secrets

`R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY` (S3 API token scoped to the
`chaselights-data` bucket). `CLOUDFLARE_ACCOUNT_ID` is only needed if the
endpoint is not the default.

## Why the marker step

Every job starts with `touch /tmp/job_start` and publishes
`find weathergrid -type f -newer /tmp/job_start`. Checkout mtimes are useless as
a discriminator — a fresh clone touches every file — so the job start time is
the only honest "what did this run produce" signal. A run that changes nothing
uploads nothing.