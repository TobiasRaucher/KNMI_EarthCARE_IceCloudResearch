# KNMI_EarthCARE_IceCloudResearch — EarthCARE Ice Cloud Characterisation

**Bachelor's thesis | Tobias Raucher | KNMI · 2026**

A Python research repository for characterising the **global distribution and physical properties of ice clouds** using the EarthCARE satellite's ATLID lidar and CPR radar. The study spans January 2025 – December 2025, with a focus period of December 2025, and includes a validation comparison against the CALIPSO satellite record.

## Repository Structure

```
KNMI_EarthCARE_IceCloudResearch/
├── final_data/         Gridded NetCDF inputs for all plotting notebooks (tracked, 71 MB)
├── notebooks/
│   ├── processing/     Six data ingestion & gridding notebooks (run on KNMI work PC)
│   └── plotting/       Five analysis & visualisation notebooks (run locally)
├── ectools/            earthcarekit Python package (pip install -e .)
├── external/           XUEMEI_earthcare_analysis companion repository
├── presentations/      KNMI Preliminary Results presentation
└── data/ output/       Raw HDF5 files and intermediate outputs — not tracked
```

---

## Data Products

| Product | Instrument | Content | Ice identifier |
|---------|-----------|---------|----------------|
| `ATL_TC__2A` | ATLID | Target classification + temperature | code 3 |
| `AC__TC__2B` | ATLID + CPR | Synergetic cloud-aerosol classification | codes 3, 13–15, 19, 21 |
| `ATL_EBD_2A` | ATLID | Extinction, backscatter, depolarization ratio, lidar ratio | masked via ATL_TC |
| `ATL_ICE_2A` | ATLID | Ice water content, effective radius | masked via ATL_TC |

Raw data: EarthCARE L2 HDF5 files delivered as ZIP archives, not tracked in this repository.
Remote storage: `/net/pc230016/nobackup_1/users/zadelhof/EarthCARE_DATA/L2/` (KNMI work PC only).

---

## Figure Data Index

All files listed below are in `final_data/` (53 files, 71 MB). Each row maps one thesis figure to its input files.

| Figure | Description | Files in `final_data/` |
|--------|-------------|------------------------|
| **Fig. 1** | Methodological flowchart | *(diagram only — no data)* |
| **Fig. 2** | Annual mean ice cloud occurrence, 2025 | `ATL_TC_2A_1.0deg_2025????_2025????_occurrence_{latlon,latheight}.nc` × 24 (12 months × 2 formats)<br>`AC_TC_2B_1.0deg_2025????_2025????_occurrence_{latlon,latheight}.nc` × 24 |
| **Fig. 3** | Temperature regime, December 2025 | `ATL_TC_2A_temperature_0.5deg_20251003_20260228_latheight.nc`<br>`combined_all_1.0deg_20251201_20251231_latheight.nc` |
| **Fig. 4** | Zonal-mean microphysics, December 2025 | `ATL_ICE_2A_v2_1.0deg_20251201_20251231_latheight.nc`<br>`combined_ice_1.0deg_20251201_20251231_latheight.nc` |
| **Fig. 5** | Zonal-mean optical properties at 355 nm, December 2025 | `ATL_EBD_2A_v2_1.0deg_20251201_20251231_latheight.nc`<br>`combined_all_1.0deg_20251201_20251231_latheight.nc` |
| **Fig. 6** | EarthCARE vs CALIPSO, December 2025 vs 2016 | `ATL_TC_2A_1.0deg_20251201_20251231_occurrence_{latlon,latheight}.nc` (subset of Fig. 2 files)<br>CALIPSO: `CAL_LID_L3_Ice_Cloud-Standard-V1-00.2016-12A.hdf` — available from [NASA ASDC](https://asdc.larc.nasa.gov) |

> `combined_all` and `combined_ice` are merged outputs containing ice cloud occurrence, temperature, and sample counts for December 2025; used to provide occurrence contours and quality masks in Figures 3–5.

---

## Processing Pipeline

All six processing notebooks follow a shared pattern:

```
Discover ZIPs → Stage locally → Extract HDF5 → Build ice mask → Grid & accumulate → Save NetCDF
```

Key design decisions:

- **ATL_TC ice mask** (code 3) applied to EBD and ICE products before accumulation
- **`np.bincount`** for vectorised histogram accumulation across 5+ months of data
- **Height grid**: 0–20 km, 100 m spacing, 201 levels
- **Horizontal grid**: 1.0° lat × 1.0° lon (180 × 360 cells)
- **Quality filter**: `quality_status` ≤ 1 (Good + Likely Good)
- **Minimum samples**: 10 pixels per grid cell before statistics are reported
- Orbit matching between product pairs uses time-overlap (robust to differing orbit numbering)

### Output file naming

```
{PRODUCT}_{RESOLUTION}deg_{START}_{END}_{FORMAT}.nc
```

| Format suffix | Dimensions | Description |
|---|---|---|
| `_3d` | lat × lon × height | Full spatial grid |
| `_latheight` | lat × height | Zonal mean (pixel-weighted) |
| `_latlon` | lat × lon | Height-collapsed occurrence |

---

## Notebooks

### `notebooks/processing/` — Data Ingestion & Gridding

> Run on the **KNMI work PC** (remote L2 data access required).
> All six notebooks are independent and can be run in parallel.

---

#### `Ice_Cloud_ATL_TC__2A_processing.ipynb`

Grids ice cloud **occurrence** from the ATLID target classification product onto a global 1° × 100 m grid for the full mission period (January 2025 – February 2026).

| Setting | Value |
|---|---|
| Input | `ATL_TC__2A` (51,274 ZIP files) |
| Ice class | code 3 only |
| QC filter | `quality_status` ≤ 1 |
| Exclude | ground (−2), missing (−3), optionally noise (−1) |
| Interpolation | Nearest-neighbor to target height grid |

**Outputs:** `ATL_TC_2A_1.0deg_{date_range}_occurrence_{3d,latlon,latheight}.nc`

---

#### `Total_Cloud_ATL_TC__2A_processing.ipynb`

Grids **all tropospheric cloud** occurrence from `ATL_TC__2A` — warm liquid (code 1), supercooled liquid (code 2), and ice (code 3) — providing a total-cloud baseline for comparison against the synergetic product.

| Setting | Value |
|---|---|
| Input | `ATL_TC__2A` (51,274 ZIP files) |
| Target classes | 1, 2, 3 (all tropospheric cloud types) |
| QC filter | `quality_status` ≤ 1 |

**Outputs:** `ATL_TC_2A_v2_1.0deg_{date_range}_{3d,latlon,latheight}.nc`

---

#### `Ice_Cloud_AC_TC_2B_processing.ipynb`

Grids ice cloud **occurrence** from the synergetic (ATLID + CPR) classification product, capturing ice clouds that the lidar alone misses due to attenuation or low signal.

| Setting | Value |
|---|---|
| Input | `AC__TC__2B` (42,625 ZIP files) |
| Ice classes | 3, 13, 14, 15, 19, 21 (all ice-related synergetic codes) |
| QC filter | `quality_status` ≤ 1 |
| Interpolation | Nearest-neighbor to target height grid |

**Outputs:** `AC_TC_2B_1.0deg_{date_range}_occurrence_{3d,latlon,latheight}.nc`

---

#### `Total_Cloud_AC_TC_2B_processing.ipynb`

Grids **all tropospheric cloud** occurrence from `AC__TC__2B` — codes 7–21, covering all cloud types from possible liquid to deep convective ice — for direct comparison with the ATL_TC total-cloud product.

| Setting | Value |
|---|---|
| Input | `AC__TC__2B` (42,625 ZIP files) |
| Target classes | 7–21 (all tropospheric clouds; excludes precipitation codes 5–6) |
| QC filter | `quality_status` ≤ 1 |

**Outputs:** `AC_TC_2B_v2_1.0deg_{date_range}_{3d,latlon,latheight}.nc`

---

#### `ATL_EBD_2A_all_processing.ipynb`

Grids four **optical property** retrievals from `ATL_EBD_2A`, restricted to ice-classified pixels (ATL_TC code 3). Applies an SNR filter (signal/error > 3) and physical plausibility bounds per variable.

| Variable | Description | Range |
|---|---|---|
| `lidar_ratio_355nm` | Lidar ratio at 355 nm | 0–200 sr |
| `particle_linear_depol_ratio_355nm` | Linear depolarization ratio | 0–1 |
| `particle_extinction_coefficient_355nm` | Extinction coefficient | 0–0.1 m⁻¹ |
| `particle_backscatter_coefficient_355nm` | Backscatter coefficient | 0–10⁻³ m⁻¹ sr⁻¹ |

Orbits are matched to `ATL_TC__2A` via time-overlap. Output files include mean, standard deviation, and sample count per cell.

**Outputs:** `ATL_EBD_2A_v2_1.0deg_{date_range}_{3d,latheight}.nc`

---

#### `ATL_ICE_2A_processing.ipynb`

Grids **microphysical retrievals** (IWC, effective radius) from `ATL_ICE_2A`, restricted to ice-classified pixels. A `d > 0` fill-value filter removes ATL_ICE fill zeros that contaminated the v1 output.

| Variable | Description | Units |
|---|---|---|
| `ice_water_content` | Ice water content | mg m⁻³ |
| `ice_effective_radius` | Particle effective radius | μm |

**Outputs:** `ATL_ICE_2A_v2_1.0deg_{date_range}_{3d,latheight}.nc`

---

### `notebooks/plotting/` — Analysis & Visualisation

> Run **locally** against NetCDF outputs copied from the work PC.
> Global maps use Robinson projection (Cartopy). All statistics are pixel-count weighted throughout.

---

#### `Q1_distribution.ipynb` → Figure 2

**Spatial distribution of EarthCARE ice clouds, January–December 2025**

Aggregates the 12 monthly ATL_TC and AC_TC occurrence files and produces maps and zonal cross-sections for the lidar-only product, the synergetic product, and their difference.

| Panel | Description |
|---|---|
| Row 1 | ATL_TC (lidar-only) ice cloud occurrence — global map + zonal cross-section |
| Row 2 | AC_TC (synergetic) ice cloud occurrence — global map + zonal cross-section |
| Row 3 | Occurrence difference (AC_TC − ATL_TC) — global map + zonal cross-section |

Includes zonal statistics broken down by latitude band (Tropics / Subtropics / Extratropics) and height tier (High >8 km / Mid 4–8 km / Low <4 km).
Smoothing: median filter (size 3) + Gaussian (σ = 1.5 for maps, σ = 1.0 for cross-sections).

**Data:** 24 monthly ATL_TC files + 24 monthly AC_TC files from `final_data/`

---

#### `Q2_temperature.ipynb` → Figure 3

**Temperature regime of ice clouds, December 2025**

Combines the full-atmosphere temperature field (October 2025 – February 2026, 0.5° grid, no ice mask) with December 2025 ice cloud data to contextualise ice detections within the thermal structure of the atmosphere.

| Figure | Description |
|---|---|
| T1 | Zonal-mean temperature field with ice occurrence contours overlaid |
| T1b | Retrieval error map: ice detections above 0 °C (1.6% of pixels) |
| T2 | Weighted histogram of ice cloud temperature (2 K bins; p25/p50/p75 marked) |
| T3 | Categorical lat-height cross-section coloured by temperature zone |
| T4 | Histogram with zone shading + cumulative ice occurrence fraction |
| T4.5 | Cumulative temperature distribution curve |
| T5 | Zonal-mean ice water path (IWP, column-integrated, occurrence-weighted) |

Temperature zone breakdown (by ice-pixel count):

| Zone | Temperature range | EarthCARE fraction |
|---|---|---|
| Retrieval error | > 0 °C | 0.004% |
| Warm / unlikely | 0 to −20 °C | 4.0% |
| Mixed-phase | −20 to −38 °C | 35.1% |
| Pure ice | −38 to −60 °C | 40.6% |
| Deep convective | < −60 °C | 20.4% |

Weighted mean ice temperature: **−45.1 °C** (IQR: −56.7 to −31.1 °C; central 90%: −78.9 to −21.0 °C).

**Data:** `ATL_TC_2A_temperature_0.5deg_20251003_20260228_latheight.nc` + `combined_all_1.0deg_20251201_20251231_latheight.nc`

---

#### `Q2_microphysics.ipynb` → Figure 4

**Microphysical properties of ice clouds, December 2025**

| Figure | Description |
|---|---|
| M1v2 | IWC lat-height cross-section (log scale, 10⁻⁶ to 1 mg m⁻³) with occurrence contours |
| M2v2 | Effective radius lat-height cross-section (linear scale, 0–120 μm) |
| M_profiles_v2 | Vertical profiles by latitude band (Tropics / Midlatitudes / High latitudes), ±1 std shaded |

Key statistics (pixel-count weighted, min 10 samples/cell):

| Variable | Weighted mean | Range |
|---|---|---|
| Ice water content | 0.024 mg m⁻³ | 8 × 10⁻⁶ – 0.99 mg m⁻³ |
| Effective radius | 70.26 μm | 0.7 – 154 μm |

**Data:** `ATL_ICE_2A_v2_1.0deg_20251201_20251231_latheight.nc` + `combined_ice_1.0deg_20251201_20251231_latheight.nc`

---

#### `Q2_optics.ipynb` → Figure 5

**Optical properties of ice clouds at 355 nm, December 2025**

| Figure | Description |
|---|---|
| M3 / M3b | Linear depolarization ratio: quantile-normalised and linear scale |
| M4 / M4b | Lidar ratio: quantile-normalised and linear scale |
| M4c–M4e | Extinction (v1.5 + v2 SNR-filtered) and backscatter coefficient (log scale) |
| M4f | Extinction vs backscatter scatter (log-log regression: slope 0.845, r = 0.749) |
| M5 | Vertical profiles by latitude band: depol ratio and lidar ratio ±1 std |
| M6 | Depol ratio vs lidar ratio scatter (Pearson r = 0.366, Spearman ρ = 0.361) |
| M7 | Depol vs lidar ratio coloured by temperature regime (5 zones) |
| M8 | Depol vs lidar ratio coloured by latitude band (20° bins) |

Key statistics (min 50 samples/cell, T < 260 K mask applied):

| Variable | Weighted mean | Std | Range |
|---|---|---|---|
| Depolarization ratio | 0.373 | 0.038 | 0.13 – 0.58 |
| Lidar ratio | 27.25 sr | 1.71 sr | 14.4 – 62.5 sr |

**Data:** `ATL_EBD_2A_v2_1.0deg_20251201_20251231_latheight.nc` + `combined_all_1.0deg_20251201_20251231_latheight.nc`

---

#### `Q3_calipso_comparison.ipynb` → Figure 6

**EarthCARE vs CALIPSO, December 2025 vs December 2016**

Reads CALIPSO L3 ice cloud data in HDF4 format (85 lat × 144 lon × 172 altitude levels, December 2016) and reproduces parallel occurrence maps and temperature statistics alongside the EarthCARE December 2025 results.

| Figure | Description |
|---|---|
| CALIPSO occurrence (lat-lon) | Global map of December 2016 ice cloud occurrence |
| CALIPSO occurrence (lat-height) | Zonal mean cross-section |
| Temperature zone comparison | EarthCARE vs CALIPSO zone fractions |

Comparison of key statistics (December 2025 vs December 2016):

| Variable | EarthCARE (Dec 2025) | CALIPSO (Dec 2016) |
|---|---|---|
| Weighted mean temperature | −45.1 °C | −48.9 °C |
| IWC weighted mean | 0.024 mg m⁻³ | 0.0118 g m⁻³ |
| Deep convective fraction | 20.4% | 23.4% |
| Pure ice fraction | 40.6% | 49.4% |
| Mixed-phase fraction | 35.1% | 24.1% |
| Warm fraction | 4.0% | 3.1% |

**Data:** `ATL_TC_2A_1.0deg_20251201_20251231_occurrence_{latlon,latheight}.nc` (from `final_data/`) + CALIPSO L3 `CAL_LID_L3_Ice_Cloud-Standard-V1-00.2016-12A.hdf`, available from [NASA ASDC](https://asdc.larc.nasa.gov) (Winker et al., 2024).

---

## Grid Settings

| Parameter | Standard processing | Temperature map |
|---|---|---|
| Horizontal resolution | 1.0° | 0.5° |
| Height range | 0–20 km | 0–20 km |
| Height spacing | 100 m (201 levels) | 100 m (201 levels) |
| Min samples per cell | 10 | 10 |
| Quality filter | `quality_status` ≤ 1 | `quality_status` ≤ 1 |

---

## Setup

```bash
# Install the earthcarekit package
pip install -e ectools/

# Core dependencies
pip install numpy scipy xarray netCDF4 matplotlib cartopy h5py pyhdf
```
