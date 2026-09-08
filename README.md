# DRM Pulsed-Flow KPI Pipeline

Automated data pipeline for **Dry Reforming of Methane (DRM)** catalyst testing on a
pulsed-flow reactor equipped with mass-spectrometer detector. It replaces hours of manual
spreadsheet processing with a single script: it discovers raw instrument files,
cleans the pulsing/noisy mass-spec signal, computes reaction KPIs, and builds an
interactive dashboard to compare catalysts side by side.

---

## Description

The reactor feed is periodically pulsed (reactant gas / inert sweep), so every raw
mass-spec channel (`Sig:_MCD_CH4`, `CO2`, `Ar`, `H2`, `CO`, `H2O`) oscillates in a
sawtooth pattern rather than sitting at a steady value. On top of that, the raw data
contains occasional pressure-spike artifacts (most visible right after a gas
switch-over, and more frequently once the reactor is hot). This pipeline:

1. Detects the real pulse period directly from the data (no hardcoded timing),
2. Smooths every species with a rolling average sized to a **whole number of pulse
   cycles**, so the periodic pulsing cancels out into a clean trend line instead of
   just being blurred, and pressure spikes get diluted rather than distorting the
   result,
3. Computes all reaction KPIs from the smoothed signal, referenced against a
   bypass-run baseline,
4. Outputs a per-catalyst Excel workbook and a combined interactive Plotly
   dashboard for cross-catalyst comparison.

---

## Repository structure

```
├── DRM_PULSING_DATA_SCRIPT.ipynb        # main pipeline notebook (run this)
├── README.md
│
├── Raw data files/                      # input: instrument exports (see naming below)
│   ├── Catalyst_XX_bypass.dat
│   ├── Catalyst_XX_reaction.dat
│   └── Catalyst_XX_parameters.dat
│
├── Output combined results/             # per-catalyst KPI + master outputs
│   ├── Catalyst_XX_integrated_results.xlsx
│   └── Master_DRM_LookerStudio_Data.csv
│
├── overview pics of averaged raw data/  # QC plots: raw vs rolling-average, per catalyst
│   ├── Catalyst_XX_averaging_qc_bypass.png
│   └── Catalyst_XX_averaging_qc_reaction.png
│
├── DRM_Interactive_Dashboard.zip        # compiled Plotly dashboard (unzip -> open .html)
└── Master_DRM_LookerStudio_Data.csv     # combined KPI table for all catalysts
```

---

## Input file naming convention

Drop any number of catalyst triplets into the raw data folder. Files are matched by
a shared `<ID>` prefix - no code changes needed to add more catalysts:

```
<ID>_bypass.dat        Pfeiffer PrismaPro export, feed gas measured with no catalyst bed
<ID>_reaction.dat       Same export format, measured over the catalyst bed
<ID>_parameters.dat     Process log: reactor temperatures (TIR-1, TIR-2), pressure
                        (PIR-1), and mass-flow controller setpoints (MFC-1..8)
```

The pipeline auto-discovers every complete `<ID>` triplet in the input folder and
processes them all in one run.

---

## Pipeline stages

**1. Read** - parses each `.dat` export (auto-detects the header row rather than
assuming a fixed line count, so it's robust to small export-format changes).

**2. Merge** - the parameters log (typically ~5s sampling) is interpolated along
its own timeline and matched onto the reaction file's native timestamps
(`merge_asof`, nearest match). This only *attaches* temperature/pressure/flow
values to each reaction row - **the reaction file's own mass-spec values are never
modified**, and the parameter values themselves are **never smoothed** (they're
merged as-measured, so real process behavior like pulsing-synced temperature
fluctuations stays visible for QC).

**3. Rolling average** - for every catalyst, the true pulse period is auto-detected
by peak-picking the Ar signal (median gap between consecutive pulse peaks, robust
to a few noisy/split peaks). The **bypass file's period is used to size the
rolling window for both the bypass and reaction smoothing** of that catalyst -
bypass is the cleaner reference signal (no catalytic reaction / no reaction-driven
pressure spikes), so it gives the more reliable period estimate. The window spans
2 full pulse cycles, which is what lets the up/down pulsing cancel out into a flat
trend line rather than just a thickened, still-pulsing envelope. The reaction
file's own period is still checked as a diagnostic; a large disagreement between
the two is printed as a warning.

**4. QC plots** - generated *before* any KPI is calculated, so smoothing quality
can be visually verified first:
- Bypass: full run (raw vs rolling average, all 6 species)
- Reaction: start / middle / end zoomed windows (the run is too long to inspect at
  full resolution in one panel)

**5. KPIs** - computed from the **rolling-averaged** signals (not raw), referenced
against the bypass baseline (mean of the smoothed bypass signal):
- CH4 / CO2 conversion
- H2 / CO / H2O yield
- H2 / CO selectivity
- H2:CO ratio (syngas quality)
- Gas-phase carbon balance

**6. Dashboard** - a single compiled HTML file with:
- QC images embedded per catalyst (smoothing check)
- **Time-dependent plots** - every KPI vs. reaction time, with reactor temperature
  on a secondary (right-hand) axis
- **Temperature-dependent plots** - every KPI vs. reactor temperature, single axis
  (best read during the ramp-up region; once steady-state is reached, temperature
  oscillates slightly with the pulse cycle, so points at nearly the same
  temperature can come from very different times - the time-dependent view is the
  reliable one for steady-state comparison)

All catalysts found in the input folder are overlaid together (one colored line
per catalyst) in every KPI plot, for direct comparison.

---

## Outputs

| File | Contents |
|---|---|
| `<ID>_integrated_results.xlsx` | 3 sheets: combined raw+merged+smoothed data, bypass raw+smoothed data, calculated KPIs (incl. reactor temperature) |
| `<ID>_averaging_qc_bypass.png` / `_reaction.png` | Raw vs. rolling-average, per species |
| `Master_DRM_LookerStudio_Data.csv` | All catalysts' KPIs combined, ready for Looker Studio / Power BI / further analysis |
| `DRM_Interactive_Dashboard.html` (zipped) | Interactive Plotly comparison dashboard |

---

## Requirements

```
pandas
numpy
scipy
matplotlib
plotly
openpyxl
```

Install with:
```
pip install pandas numpy scipy matplotlib plotly openpyxl
```

## How to run

1. Place your `<ID>_bypass.dat`, `<ID>_reaction.dat`, `<ID>_parameters.dat` files
   in the raw data folder (any number of catalysts).
2. Open `DRM_PULSING_DATA_SCRIPT.ipynb` and update `INPUT_FOLDER` / `OUTPUT_FOLDER`
   at the top of the script to point at your folders.
3. Run all cells. Per-catalyst Excel files, QC plots, the master CSV, and the
   dashboard HTML are written to `OUTPUT_FOLDER`.

## Configuration reference

A few constants near the top of the script control the pipeline's behavior:

| Constant | Meaning |
|---|---|
| `N_CYCLES` | How many pulse periods make up one rolling-average window (default: 2) |
| `MIN_PEAK_GAP_SEC` | Minimum spacing enforced between detected pulse peaks - only needs to be smaller than your actual pulse period (rule of thumb: ~30-50% of it) |
| `ZOOM_WIDTH_FRAC` | Fraction of the reaction run shown in each start/middle/end QC zoom window |

## Notes / known limitations
- The bypass run should be long enough to give a stable, representative baseline;
  a very short bypass run gives the startup transient more relative weight in the
  baseline average.
- Included as additional file is a user **manual pdf** guide for running this script
