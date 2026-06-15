# Project Guidelines

## Overview

Oslofjord ocean simulation coupling hydrodynamic modeling (Oceananigans/FjordSim) with a biogeochemical OXYDEP model for oxygen depletion and simplified ecosystem dynamics. Dual-language: **Julia** for simulation, **Python** for preprocessing. Deployable via Docker to GCP Batch or run locally.

## Architecture

| Component | Language | Entry point | Purpose |
|-----------|----------|-------------|---------|
| Simulation | Julia | `simulation.jl` | Orchestrates hydro + biogeochemistry via `FjordSim.coupled_hydrostatic_simulation()` |
| OXYDEP model | Julia | `Oxydep.jl` | 6-tracer biogeochemistry (P, HET, NUT, DOM, POM, O₂) with redox switches |
| Scenario config | Julia | `scenarios.jl` + `scenarios.json` | Named scenario definitions mapping to grid/forcing/atmosphere files |
| Preprocessing | Python | `preprocess/` | Download NorKyst data, regrid, prepare bathymetry & forcing NetCDFs |
| Utilities | Python | `oslofjord_sim/stuff.py` | Regridding, coordinate transforms, gap-filling, OPeNDAP catalog |
| Deployment | Docker | `Dockerfile`, `entrypoint.sh`, `docker-compose.yml` | Container for local and GCP Batch execution |

**Tracers** (9 total): T, S, e, NUT, P, HET, POM, DOM, O₂.
**Units**: Biogeochemistry in N-units (mmol/m³); Redfield ratios for C/N conversion.

## Data Layout

All paths are relative to `PROJECT_ROOT` (env var, defaults to `/app` in Docker, repo root locally):

```
data/
  input/              # Grid, forcing, and atmosphere files
    bathymetry_*.nc
    forcing_*.nc
    JRA55/            # JRA55 atmospheric reanalysis files
  output/
    <scenario_name>/  # Per-scenario output directory
      snapshots_ocean*.nc
      simulation.log
```

## Scenario System

Scenarios are defined in `scenarios.json` with keys mapped to grid/forcing/atmosphere files:
```json
{
  "example": {
    "grid_file": "bathymetry_105to232.nc",
    "forcing_file": "forcing_105to232.nc",
    "atmospheric_forcing_dir": "JRA55",
    "hotstart_file": "snapshots_ocean.nc"
  }
}
```
`scenarios.jl` (`ScenarioConfigs` module) loads and resolves these to full paths under `data/input/` and `data/output/<scenario>/`.

## Build and Run

### Julia (local)
```bash
julia --project                  # Activate environment
# In Pkg REPL (press ]):
instantiate                      # Install from Project.toml/Manifest.toml
```
```bash
export PROJECT_ROOT=/path/to/repo
julia --project simulation.jl --scenario example --arch cpu
```
CLI args: `--scenario <name>` (key in `scenarios.json`), `--arch cpu|gpu`.

### Docker (local)
```bash
docker compose up --build        # Builds and runs with settings in docker-compose.yml
```
Configure via environment variables in `docker-compose.yml`: `SCENARIO`, `ARCH`, `JULIA_NUM_THREADS`.
Input/output volumes are mounted from `./data/input` and `./data/output`.

### Docker (GCP Batch / cloud)
Set `GCS_BUCKET` env var to enable cloud mode. `entrypoint.sh` handles:
- Downloading input from `gs://<GCS_BUCKET>/<GCS_INPUT_PREFIX>/`
- Running the simulation
- Uploading output to `gs://<GCS_BUCKET>/<GCS_OUTPUT_PREFIX>/<JOB_ID>/`

### Python
```bash
conda create --name oslofjord python=3.12
conda activate oslofjord
conda install -c conda-forge xesmf   # xesmf is conda-only, not pip
pip install -e .                      # Install oslofjord_sim package
conda env config vars set PROJECT_ROOT=/path/to/repo
```
Preprocessing notebooks (`bathymetry.ipynb`, `forcing.ipynb`) prepare grid and forcing files.

## Conventions

- **Scenario-driven config**: `simulation.jl` accepts `--scenario` and `--arch` CLI args. All file paths are resolved from `scenarios.json` relative to `PROJECT_ROOT`.
- **Data format**: NetCDF with zlib compression (complevel 5). Output snapshots every 6 hours.
- **Domain**: Oslo fjord, WGS84 coordinates (~58–60°N, 9–11°E).
- **Commented-out options**: Code uses `#¤` markers for disabled alternatives (KE closure, restart from snapshot, alternative advection). Preserve these when editing.
- **OXYDEP parameters**: ~30 tunable rate constants with defaults in the `OXYDEP()` constructor. Reference: Berezina et al. 2022.
- **`stuff.py` naming**: The utility module is `oslofjord_sim/stuff.py` — this is intentional, not a placeholder.

## Gotchas

- **GPU default**: Simulation targets NVIDIA/CUDA. Pass `--arch cpu` for CPU-only machines.
- **FjordSim dependency**: Pulled from GitHub via HTTPS (`https://github.com/NIVANorge/FjordSim.jl.git`, rev `main`). Requires access to the repo for `Pkg.instantiate()`.
- **xesmf**: Must be installed via conda, not pip (ESMF C library dependency).
- **Negative tracers**: Biogeochemistry can produce negative concentrations. `ScaleNegativeTracers` callback is enabled by default — do not remove it.
- **Memory**: 9 tracers on GPU can exhaust VRAM. Test with smaller grids first.
- **NorKyst OPeNDAP**: Downloads rely on THREDDS server availability (`thredds.met.no`).
- **PROJECT_ROOT**: Must be set as an env var. In Docker it defaults to `/app`; locally set it to the repo root.
