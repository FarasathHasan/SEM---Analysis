# Urban Luminosity Morphology Integration Framework (ULMIF)
## Composite Urban Growth Score — Six-Indicator Pipeline

A fully data-driven pipeline that computes a **Composite Urban Growth Score (CUGS)** across functional urban areas by integrating six spatial indicators derived from GHSL land-use rasters and optional population layers. All normalization bounds, indicator weights, and architectural parameters are learned from the data itself — no hardcoded thresholds.

---

## Overview

The pipeline processes two epochs of classified land-use rasters (2017 and 2024, 10 m resolution, urban class = 1) and derives six urban growth structure indicators for all pixels that transitioned to urban between the two time points. These indicators are then fused into a single composite sprawl score using a three-stage learning procedure: data-driven weight learning (PCA + entropy + CV ensemble), unsupervised Variational Autoencoder (VAE) training, and spatially explicit score modulation.

### Six Indicators

| Script | Indicator | Abbrev. | Core Concept |
|---|---|---|---|
| `gm_analysis.py` | Growth Magnitude | GM | Gross new urban area (absolute + relative), MCDA-weighted |
| `gvf_analysis.py` | Growth Velocity Factor | GVF | Mean expansion speed of new urban pixels (m/year) |
| `lce_analysis.py` | Land Consumption Efficiency | LCE | New urban area per unit population increase (ha/person) |
| `lum_analysis.py` | Land Use Mix | LUM | Shannon Evenness Index change across new growth pixels |
| `sd_analysis.py` | Spatial Dispersion | SD | Integrated Ripley K/L + Moran I + NNI + VMR (CDF-based) |
| `smc_analysis.py` | Spatial Morphological Complexity | SMC | Fractal dimension, lacunarity, wavelet energy, shape metrics |

### Composite Score (`sprawl_index.py`)

The composite framework fuses all six indicator rasters into a single per-pixel and city-level sprawl index. Weight learning uses an ensemble of PCA loadings, Shannon entropy, and coefficient of variation. A VAE trained on the joint indicator distribution provides a reconstruction-error modulation factor (alpha) that captures structural complexity not captured by linear weighting.

---

## Repository Structure

```
.
├── gm_analysis.py            # Growth Magnitude
├── gvf_analysis.py           # Growth Velocity Factor
├── lce_analysis.py           # Land Consumption Efficiency
├── lum_analysis.py           # Land Use Mix
├── sd_analysis.py            # Spatial Dispersion
├── smc_analysis.py           # Spatial Morphological Complexity
├── sprawl_index.py           # Composite score integration (VAE + weights)
│
├── Processed data/
│   ├── 2017_cleaned.tif      # Classified land-use raster, 2017
│   ├── 2024_cleaned.tif      # Classified land-use raster, 2024
│   ├── pop2017.tif           # Population density raster, 2017 (for LCE)
│   └── Pop2024.tif           # Population density raster, 2024 (for LCE)
│
├── outputs_growth_magnitude/ # GM raster + summary CSV + learned params JSON
├── outputs_gvf_2017_2024_corrected/  # GVF speed raster
├── LCE_results/              # LCE index raster + decoupling JSON/CSV
├── outputs_lum/              # LUM change raster + cross-city CSV
├── outputs_urban_sprawl_dispersion/  # SD raster + figures + report
├── outputs_smc/              # SMC proxy raster + composite JSON
└── sprawl_index_results/     # Final sprawl index TIFF + report JSON
```

---

## Input Data Requirements

All rasters must be **pre-processed to the same spatial grid** before running any script. The composite framework (`sprawl_index.py`) enforces strict alignment checks and will raise an error if CRS, affine transform, width, height, or bounds differ between any two inputs.

| Requirement | Specification |
|---|---|
| Format | GeoTIFF (.tif) |
| Coordinate Reference System | Any projected CRS (UTM recommended) |
| Pixel resolution | 10 m (pipeline is resolution-aware; cell size is read from the raster transform) |
| Land-use class scheme | 1 = Urban, 2 = Vegetation, 3 = Water, 4 = Agriculture |
| NoData encoding | −9999, −32767, NaN, or values < −1×10¹⁰ (all detected automatically) |
| Population rasters | Same grid as land-use rasters; units = persons per pixel |
| Temporal coverage | Two epochs: 2017 and 2024 (configurable per script) |

### Running Order

The scripts must be run in the order below because each one produces rasters consumed by the next:

```
1. gm_analysis.py
2. gvf_analysis.py
3. lce_analysis.py
4. lum_analysis.py
5. sd_analysis.py
6. smc_analysis.py
7. sprawl_index.py   ← reads all six output rasters
```

---

## Installation

Python 3.9 or later is required.

```bash
git clone https://github.com/your-org/ulmif-sprawl-index.git
cd ulmif-sprawl-index
pip install -r requirements.txt
```

---

## Requirements

See `requirements.txt` for pinned versions. The key dependencies are:

```
numpy>=1.24
pandas>=2.0
rasterio>=1.3
scipy>=1.11
scikit-learn>=1.3
scikit-image>=0.21
PyWavelets>=1.4
torch>=2.0          # CPU build is sufficient; GPU optional
tqdm>=4.65
matplotlib>=3.7
```

> **Note on PyTorch:** The VAE in `sprawl_index.py` uses PyTorch. A CPU-only installation is sufficient for cities up to ~200 000 new-urban pixels. For larger cities install the CUDA build following the official instructions at https://pytorch.org.

> **Note on GDAL/rasterio:** On Windows, install rasterio via the pre-built wheel from https://www.lfd.uci.edu/~gohlke/pythonlibs/ or via `conda install -c conda-forge rasterio` to avoid GDAL dependency conflicts.

---

## Usage

### Running a single indicator

```bash
python gm_analysis.py
```

Each script reads input paths from constants near the top of the file. Edit the `INPUT_FILE_*` and `OUTPUT_DIR` variables (or the `DataLoader` path arguments) to match your directory layout before running.

### Running the composite index

After all six indicator scripts have completed successfully:

```bash
python sprawl_index.py
```

Edit the six `*_PATH` constants at the top of `sprawl_index.py` to point to the output rasters produced by the individual indicator scripts.

### Per-city configuration

Scripts that write cross-city CSV files expose a `CITY_NAME` constant near the top of the file. Set this string before each run so that results are appended correctly to the cross-city comparison tables.

---

## Outputs

| File | Description |
|---|---|
| `sprawl_index_results/sprawl_index_spatial_map.tif` | Per-pixel composite sprawl index, 0–1 |
| `sprawl_index_results/sprawl_index_report.json` | City-level score, learned weights, normalization bounds, VAE parameters |
| `outputs_growth_magnitude/gm_summary_2017_2024.csv` | GM metrics per period |
| `outputs_gvf_2017_2024_corrected/gvf_speed_*.tif` | GVF expansion speed raster (m/year) |
| `LCE_results/land_consumption_efficiency_*.tif` | LCE index raster |
| `cross_city_lum_new_growth.csv` | Cross-city LUM comparison table |
| `outputs_urban_sprawl_dispersion/composite_spatial_dispersion_score.json` | SD composite score |
| `outputs_smc/composite_smc_new_growth.json` | SMC composite score |
| `cross_city_smc_new_growth.csv` | Cross-city SMC comparison table |

Score interpretation: **0 = compact / no sprawl**, **1 = maximum sprawl / dispersed**.

---

## Methodological Notes

**Growth Magnitude (GM)** uses a two-pass design: the first pass collects gross new urban area values across all time intervals to learn normalization maxima; the second pass applies learned MCDA weights (PCA + entropy + CV ensemble on absolute and relative growth rates) and a spatially explicit modifier derived from distance-to-urban-core, local growth density, and edge proximity.

**Growth Velocity Factor (GVF)** computes KD-tree nearest-neighbor distances from each new urban pixel to the nearest 2017 urban pixel, then divides by the time span to yield expansion speed in meters per year.

**Land Consumption Efficiency (LCE)** uses a sliding window convolution to compute local new urban area and local population increase, then takes their ratio. Pixels with negligible population increase are flagged as inefficient and assigned a penalized value. A data-driven decoupling index (ratio of population growth rate to land growth rate) is also computed.

**Land Use Mix (LUM)** computes the Shannon Evenness Index (SHEI) at 2017 and 2024 using a data-driven window size learned from patch-size statistics of the classified raster. The composite score for cross-city comparison is the mean SHEI change over new growth pixels.

**Spatial Dispersion (SD)** integrates four independent spatial statistics — Ripley's L, Moran's I (quadrat-based), Nearest Neighbor Index, and the Variance-to-Mean Ratio — each expressed as an empirical CDF probability against a CSR null distribution generated from Monte Carlo simulations restricted to the valid study area. Confidence-weighted integration combines the four scores into a final dispersion value.

**Spatial Morphological Complexity (SMC)** computes fractal dimension (box-counting), lacunarity (gliding box), wavelet energy, shape index, compactness, edge density, and fragmentation on the new-growth mask. Normalization ranges (5th–95th percentile) and entropy-based weights are learned from random 100 × 100 pixel patches sampled from the same mask.

**Composite integration** applies a PCA + entropy + CV ensemble weighting to the normalized six-indicator feature matrix, then trains an unsupervised VAE on that matrix. The reconstruction error coefficient of variation determines alpha, the modulation strength. Final score = clamp(weighted_sum × (1 + alpha × normalized_reconstruction_error), 0, 1).


## License

MIT License. See `LICENSE` for details.
