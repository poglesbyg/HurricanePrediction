# Hurricane Risk: Asheville, NC

> 🌪 **Live dashboard:** <https://paulgreenwood-usda.github.io/HurricanePrediction/>
> — rebuilt hourly by GitHub Actions from USGS, NWS, NHC, Open-Meteo, and NOAA CO-OPS feeds.

A Python toolkit (managed with [uv](https://docs.astral.sh/uv/)) that estimates
the risk of an Atlantic tropical cyclone — and the resulting **flood hazard** —
for **Asheville, NC**, by fusing climatology, seasonal outlooks, real-time
storm tracks, live river-gauge data, terrain-aware orographic rainfall
modeling, soil-moisture pre-conditioning, and a live web dashboard.

It is built around the lesson of Helene (Sep 2024): for an inland city in the
Blue Ridge, the danger is not landfalling wind but **decaying tropical systems
dumping orographic rainfall into a north-flowing mountain watershed on already
saturated soil**.

## What it does

1. **Climatology** — NOAA HURDAT2 best-track data (1851–2025, includes Helene)
   filtered to storms that came within a configurable radius of Asheville
   (default 150 mi).
2. **Seasonal context** — Colorado State University (CSU) Tropical
   Meteorology Project April 2026 outlook (extracted from *Helene PMP HUB.pdf*).
   Climatological frequency is scaled by `ACE_forecast / ACE_climo`.
3. **Real-time storm tracking** — National Hurricane Center *Active Storms*
   feed, with distance from each current Atlantic storm to Asheville.
4. **Terrain-aware orographic rainfall** — real DEM (Open-Meteo Elevation),
   the upslope wind component `V · ∇h` at the SE Blue Ridge escarpment,
   Kaplan–DeMaria inland wind decay with mountain enhancement, monthly PWAT
   climatology + tropical-moisture surge, and the fraction of each track that
   sits inside the French Broad watershed upstream of Asheville.
5. **Live river gauge + flood thresholds** — USGS NWIS for French Broad at
   Asheville (site 03451500), with NWS AHPS action / minor / moderate / major
   stage thresholds and Helene's preliminary record stage for reference.
6. **Active NWS alerts** — api.weather.gov alerts for the Asheville point and
   national-forest centroids.
7. **Soil moisture / antecedent precipitation** — Open-Meteo volumetric water
   content at multiple depths plus 7-day rainfall totals (the missing
   pre-conditioner that turned Helene from "wet" into "catastrophic").
8. **Surrounding context** — current weather + alerts for the four NC
   National Forests (Pisgah, Nantahala, Uwharrie, Croatan) and live tide /
   storm-surge observations from NOAA CO-OPS stations along the NC coast.
9. **Asheville Flood Index (AFI)** — a single 0–100 composite
   (CALM → ELEVATED → ALERT → WARNING → EMERGENCY) computed from the live
   gauge, soil saturation, antecedent rainfall, active alerts, and proximity
   of any active tropical system.
10. **Live web dashboard** — Flask app that renders all of the above with
    60-second caching; also publishable as a static snapshot to GitHub Pages
    and deployable to Azure App Service.

> Asheville is ~250 mi inland in the Blue Ridge mountains at ~2,134 ft. The
> hazard is almost always **rainfall and flooding from decaying tropical
> systems** (Helene 2024, Frances + Ivan 2004), not wind from a landfalling
> hurricane. The `P(hurricane-strength inside radius)` metric is therefore
> expected to be very low — that is physically correct.

## Setup

```powershell
uv sync
```

Python 3.12+ is required.

## CLI usage

The package installs a single console script, `hurricane-asheville`.

```powershell
# Climatology + 2026 CSU-adjusted seasonal probability
uv run hurricane-asheville risk

# List the historical storms that came within 150 mi (sorted by date)
uv run hurricane-asheville history --start-year 1990

# 10 closest passes ever
uv run hurricane-asheville history --top 10

# Different radius (e.g. 100 mi for direct rainfall hits)
uv run hurricane-asheville --radius 100 risk

# Real-time check of any active Atlantic storms
uv run hurricane-asheville active

# Plot all historical tracks that came within radius
uv run hurricane-asheville plot --output output/tracks.png

# Rank historical storms by terrain-aware orographic rainfall risk
uv run hurricane-asheville terrain --top 15
# (add --no-dem to skip the real DEM upslope calc and use the longitude proxy)

# Pre-download the regional elevation grid into data/dem.npz
uv run hurricane-asheville dem

# Live USGS gauge + NWS alerts for Asheville
uv run hurricane-asheville gauge

# Replay the Flood Index over history and validate it against Helene
uv run hurricane-asheville index-replay

# Run the live web dashboard (Flask) at http://127.0.0.1:5000
uv run hurricane-asheville dashboard
uv run hurricane-asheville dashboard --host 0.0.0.0 --port 8000 --debug
```

Global options (placed before the subcommand):

| Flag           | Default | Description                                          |
| -------------- | ------- | ---------------------------------------------------- |
| `--cache-dir`  | `data`  | Where HURDAT2 and the DEM grid are cached on disk.   |
| `--radius`     | `150`   | Search radius (mi) around Asheville for climatology. |
| `--start-year` | `1950`  | First year included in climatology / plots.          |

## Web dashboard

```powershell
uv run hurricane-asheville dashboard
```

Then open <http://127.0.0.1:5000>. The dashboard refreshes its underlying data
every 60 seconds (in-process cache) and shows:

- Asheville Flood Index (0–100) with label, shape-coded severity and breakdown
- Current French Broad stage vs that gauge's action / minor / moderate / major
- **Stage history since 2021** with the Helene crest and NWS record as
  reference lines, plus where today sits as a percentile *for this month*
- **Hourly rainfall timing** — the 72 h total cannot tell a wet week from a
  flash flood, so the page shows the hourly curve and the wettest 6 h / 24 h
- ML flood probability for the action stage, the only head that beats its
  naive baseline. The stage regression is **withheld** because it loses to
  "assume no change" at every horizon, and the card says so rather than
  quietly omitting it (see [Method notes](#method-notes))
- Active NHC Atlantic storms with their forecast track and cone of uncertainty
- Active NWS alerts, current weather, soil moisture, 7-day antecedent precip
- Live NOAA CO-OPS tide / surge observations for NC coastal stations
- Per-forest weather + alerts for Pisgah / Nantahala / Uwharrie / Croatan

The header states how old the data is and how often the page rebuilds. The
staleness dot changes colour when a rebuild is missed rather than pulsing
green regardless, and the page reloads only when a newer snapshot exists.

### Static snapshot (GitHub Pages)

```powershell
uv run python build_static.py
```

Renders the dashboard once and writes `site/index.html`, `site/state.json`
and `site/static/` (CSS + JS). Asset URLs are rewritten to relative paths
so the snapshot works under the Pages project subpath.

### Azure App Service

The repo ships an [azure.yaml](azure.yaml) and Bicep under [infra/](infra/) for
[`azd`](https://learn.microsoft.com/azure/developer/azure-developer-cli/):

```powershell
azd up
```

Oryx runs gunicorn against [wsgi.py](wsgi.py), which adds `src/` to
`sys.path` and exposes `app` from `hurricane_asheville.dashboard`.

## Method notes

- **Climatology probability** uses a Poisson model:
  $P(\ge 1 \text{ storm in a year}) = 1 - e^{-\lambda}$ where
  $\lambda$ = mean storms per year within the radius.
- **Seasonal scaling** multiplies $\lambda$ by `ACE_forecast / ACE_climo`
  (90 / 123 ≈ 0.73 for 2026). This is a deliberately simple adjustment — CSU
  itself notes seasonal forecasts have no skill at predicting *where* storms
  go, only basin-wide activity.
- **Inland wind decay** follows Kaplan & DeMaria (1995), with an additional
  mountain-enhancement term in [src/hurricane_asheville/terrain.py](src/hurricane_asheville/terrain.py).
- **Orographic factor** combines storm position / motion with the real terrain
  gradient sampled from a cached Open-Meteo elevation grid
  ([src/hurricane_asheville/dem.py](src/hurricane_asheville/dem.py),
  [src/hurricane_asheville/terrain.py](src/hurricane_asheville/terrain.py)).
  When no DEM is available it falls back to a longitude / motion heuristic.
- **Moisture factor** uses monthly PWAT climatology for ~35 N 82 W with a
  tropical-surge bonus when an Atlantic TC sits south of Asheville
  ([src/hurricane_asheville/moisture.py](src/hurricane_asheville/moisture.py)).
  Plug in real ERA5 via the `era5_pwat()` hook if you have a CDS API key.
- **Watershed weighting** intersects each track with a hand-drawn polygon
  approximating the French Broad watershed upstream of Asheville
  ([src/hurricane_asheville/watershed.py](src/hurricane_asheville/watershed.py)).
- **Flood Index** combines the live gauge percentile, topsoil saturation
  (≥ 0.40 m³/m³), 7-day antecedent precipitation, active flood / tropical
  alerts, and proximity of any active TC
  ([src/hurricane_asheville/index_score.py](src/hurricane_asheville/index_score.py)).
- **Flood thresholds are per gauge.** Each USGS site sits on its own datum, so
  every gauge is classified against its own NWS action / minor / moderate /
  major stages from
  [data/nws_flood_thresholds.json](data/nws_flood_thresholds.json). Gauges with
  no published NWS thresholds render as *no thresholds* rather than being
  measured against another river's numbers. Refresh the table with
  `uv run python scripts/refresh_flood_thresholds.py` (slow by design — NWPS
  allows 10 requests per 5 minutes).
- **Reservoirs report pool elevation**, USGS parameter `00062`, not river stage
  `00065` — Falls Lake reads ~249 ft against a 264 ft flood pool. NWS publishes
  those thresholds in the same elevation datum, so reservoirs are classified
  normally; the units travel with the number so a pool elevation is never shown
  as a river stage.
- **Gauge ids are pinned to their USGS station names** in
  [tests/test_gauge_registry.py](tests/test_gauge_registry.py). Verify any new
  site id against the USGS site service before adding it — a label and an id
  that disagree will otherwise show one river's data under another's name.
- **ML forecast honesty.** The regression predicts the **change** in stage,
  not the level. Predicting the level means predicting a quantity the model
  already holds as a feature: it spends its capacity reproducing its own
  input, and the level models lost to plain persistence by 2-4x. Against a
  rise target the naive baseline is exactly zero, so any explained variance
  is a real gain. Switching cut +6 h error 43% (0.121 -> 0.068 ft).

  Every head is scored against that baseline on the same walk-forward folds,
  and `serving.py` refuses to publish one that fails. Because 99% of hours
  the river does nothing — where zero is unbeatable and no model can do
  better than tie — the gate uses the rows where the river is **actually
  rising** (>= 0.5 ft). Judging on the full population would measure the calm
  regime almost exclusively.

  | head | rising rows | no-change | all hours | served? |
  | --- | --- | --- | --- | --- |
  | stage +6 h | MAE 1.08 ft | 1.30 ft | 0.07 vs 0.03 | yes |
  | stage +24 h | 1.00 ft | 1.31 ft | 0.16 vs 0.10 | yes |
  | stage +72 h | 1.21 ft | 1.53 ft | 0.50 vs 0.26 | yes |
  | P(> action 6.5 ft) | AUC 0.98 / 0.99 / 0.78 | 0.96 / 0.87 / 0.75 | — | yes |
  | P(> minor 9.5 ft), P(> moderate 13 ft) | AUC undefined | — | — | no |

  So the stage forecast is a *rising-river* forecast: useful when the stage
  is moving, redundant when it is flat, and worse than assuming no change if
  you average over every hour of the year. The card says exactly that rather
  than quoting only the flattering number.

  Gating on the rising regime rather than the overall average is a deliberate
  choice, and it is what admits the three stage heads — they lose on overall
  MAE and win when the river moves. Both verdicts are recorded
  (`beats_baseline`, `beats_baseline_overall`) so the trade-off stays visible,
  and `tests/test_models.py` pins the gate to the rising regime. Minor and moderate exceedance have
  been crossed once since 2021 (Helene), giving a positive example in a
  single fold and an undefined AUC, so those heads stay unserved.

- **The models had never seen rainfall.** `features.py` requested
  `precip_in_24h` while the store writes `wx_precip_in_24h` (history adds the
  `wx_` prefix), so the block was skipped without error and every model
  trained before this fix used river stage alone — 83 features, all of them
  self- or upstream-stage. Precipitation and ERA5 soil moisture are now wired
  in (111 columns, ~95% coverage). Forecast QPF is deliberately excluded: it
  covers ~4% of rows, all in the final fold, so it would teach a time signal
  rather than hydrology.
- **Helene's chart peak is a floor, not the crest.** The USGS gauge recorded
  18.47 ft before it stopped reporting; NWS carries the crest at 24.82 ft. The
  history card says so rather than presenting the recorded value as the peak.
- **The Flood Index has been validated against Helene.** `index-replay`
  reconstructs historical inputs and runs the *same* scorer the live page uses
  over 1,902 days (2021-05-27 to present). Result: Helene peaks at **96/100
  (EMERGENCY)** on 2024-09-27 with **3 days of ALERT-or-above lead time**, and
  only **12 of 1,902 days (0.63%)** ever reach ALERT — every one of them with a
  named tropical system in range (Fred, Henri, Ida, Ian, Nicole, Helene). No
  false positives in five years.

  The replay is deliberately conservative in three of four respects: stage is a
  daily mean rather than the crest, rate-of-rise is averaged over 24 h so a
  flash rise barely registers, and NWS alert state is not archived so that
  component is always zero. The fourth cuts the other way — rainfall uses what
  actually fell, so it assumes a perfect 72-hour forecast and real lead time
  would be shorter. Regenerate with
  `uv run hurricane-asheville index-replay`, which writes
  [data/index_validation.json](data/index_validation.json) for the dashboard.
- HURDAT2 is cached in [data/hurdat2.txt](data/hurdat2.txt) and the elevation
  grid in [data/dem.npz](data/dem.npz) after first download.

## Data sources

| Source                                  | Auth      | Used for                                  |
| --------------------------------------- | --------- | ----------------------------------------- |
| NOAA NHC HURDAT2                        | none      | Historical Atlantic best-tracks 1851–2025 |
| NOAA NHC CurrentStorms.json             | none      | Active Atlantic storms                    |
| NOAA NHC storm_graphics KMZ             | none      | Forecast track + cone of uncertainty      |
| USGS NWIS Instantaneous Values          | none      | French Broad gauge (site 03451500)        |
| api.weather.gov                         | none      | NWS active alerts                         |
| Open-Meteo Forecast API                 | none      | Current weather + soil moisture           |
| Open-Meteo Elevation API                | none      | DEM grid for orographic calc              |
| NOAA CO-OPS datagetter                  | none      | Live NC tide / surge observations         |
| NWS NWPS (api.water.noaa.gov)           | none      | Per-gauge flood thresholds (baked)        |
| Open-Meteo ERA5 archive                 | none      | 5-year precip / soil-moisture backfill    |
| CSU Tropical Meteorology Project (PDF)  | bundled   | 2026 seasonal forecast constants          |

All HTTP calls have timeouts and degrade gracefully when a service is offline;
the CLI and dashboard remain usable on cached climatology alone.

## Repository layout

```
azure.yaml                    azd manifest (App Service, Python)
build_static.py               render dashboard once for GitHub Pages
extract_pdf.py                pull text out of the bundled CSU PDF
wsgi.py                       gunicorn entry point for Azure App Service
infra/                        Bicep for App Service + supporting resources
scripts/
  refresh_flood_thresholds.py regenerate data/nws_flood_thresholds.json
site/                         static snapshot output (index.html, state.json)
data/                         hurdat2.txt + dem.npz cache
  nws_flood_thresholds.json   per-gauge NWS flood stages (from NWPS)
  index_validation.json       Flood Index replay summary (from index-replay)
output/                       generated plots (e.g. tracks.png)
src/hurricane_asheville/
  config.py        constants, Asheville lat/lon, CSU 2026 numbers, URLs
  geo.py           haversine distance
  hurdat.py        HURDAT2 download + parser
  risk.py          climatology + seasonal scaling
  active.py        NHC current-storms client
  terrain.py       orographic factor, inland wind decay, storm scoring
  dem.py           Open-Meteo elevation grid + grad(h) / upslope component
  moisture.py      PWAT climatology + tropical-surge factor
  watershed.py     French Broad upstream-of-Asheville polygon test
  gauge.py         USGS NWIS gauge + NWS alerts + flood thresholds
  weather.py       Open-Meteo current/forecast weather
  soil.py          soil moisture + 7-day antecedent precip
  forests.py       NC National Forests (Pisgah/Nantahala/Uwharrie/Croatan)
  tides.py         NOAA CO-OPS tide + meteorology stations
  index_score.py   Asheville Flood Index (AFI) composite 0-100
  dashboard.py     Flask app: collection, view model assembly, routes
  viewmodel.py     pure state -> render primitives (sparklines, bands, ML card)
  stage_history.py long-run stage series + month-of-year percentile
  storm_track.py   NHC forecast track / cone KMZ parsing
  index_replay.py  historical Flood Index replay + Helene validation
  templates/       dashboard.html + card partials
  static/          dashboard.css, dashboard.js
  cli.py           argparse entry point
tests/                        pytest suite (conftest stubs out network)
```

## Tests

```powershell
uv run pytest
```

Network-touching modules are stubbed out via [tests/conftest.py](tests/conftest.py)
so the suite runs offline.
