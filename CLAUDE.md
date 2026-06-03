# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

This is a doctoral research project implementing the data pipeline and analysis from the paper **"Data-Driven Prediction of Battery Cycle Life Before Capacity Degradation"** (Severson, Attia et al., 2019, Nature Energy). The repository contains code to load, convert, and explore the associated lithium-ion battery cycling dataset (124 cells across 3 experimental batches). The predictive modeling code is under academic license — contact braatz@mit.edu — and is **not** included here.

## Environment Setup

```bash
conda create --name battery-life python=3.7.10
conda activate battery-life
conda install --file requirements.txt
jupyter notebook
```

Key packages: `h5py==2.8.0`, `numpy==1.22.0`, `scipy==1.6.1`, `matplotlib==3.3.4`, `pandas==1.2.3`.

## Data

Raw `.mat` files must be placed in a `./Data/` subdirectory (not tracked by git). Download from [https://data.matr.io/1/](https://data.matr.io/1/).

| File | Batch |
|------|-------|
| `2017-05-12_batchdata_updated_struct_errorcorrect.mat` | Batch 1 (46 cells) |
| `2017-06-30_batchdata_updated_struct_errorcorrect.mat` | Batch 2 (48 cells) |
| `2018-04-12_batchdata_updated_struct_errorcorrect.mat` | Batch 3 (45 cells after cleaning) |

## Data Pipeline Architecture

The pipeline has two stages:

**Stage 1 — Convert `.mat` → `.pkl`** (one notebook per batch):
- `BuildPkl_Batch1.ipynb` → `batch1.pkl`
- `BuildPkl_Batch2.ipynb` → `batch2.pkl`
- `BuildPkl_Batch3.ipynb` → `batch3.pkl`

Each notebook reads raw HDF5-formatted `.mat` files via `h5py`, extracts per-cell data, and serializes to a Python pickle. The pkl files use nested dicts with cell keys `b1cN`, `b2cN`, `b3cN`.

**Stage 2 — Load & analyze** (`Load Data.ipynb` / `LoadData.m`):
- Merges the three batches, applies batch-specific cleaning (described below).
- Defines the train/test split identical to the paper.

## Data Structure (Python pkl)

```python
bat_dict['b1c0'] = {
    'cycle_life': int,          # number of cycles to 80% capacity (EOL)
    'charge_policy': str,       # e.g. "3.6C(80%)-4C"
    'summary': {
        'cycle': array,         # cycle index
        'QD': array,            # discharge capacity (Ah)
        'QC': array,            # charge capacity (Ah)
        'IR': array,            # internal resistance (Ohm)
        'Tavg': array,          # avg temperature (°C)
        'Tmin': array,
        'Tmax': array,
        'chargetime': array,    # time to reach 80% SOC (min)
    },
    'cycles': {
        '0': {                  # cycle number as string key
            'I': array,         # current (A)
            'Qc': array,        # charge capacity (Ah)
            'Qd': array,        # discharge capacity (Ah)
            'Qdlin': array,     # discharge capacity, linearly interpolated
            'T': array,         # temperature (°C)
            'Tdlin': array,     # temperature, linearly interpolated
            'V': array,         # voltage (V)
            'dQdV': array,      # discharge differential capacity
            't': array,         # time (min)
        }
    }
}
```

## Batch Cleaning Rules (critical for reproducibility)

These are applied in `LoadData.m` and must be replicated in Python analysis:

1. **Batch 1 ↔ Batch 2 overlap**: 5 cells in Batch 1 continued cycling into Batch 2. Their Batch 2 cycles are appended to Batch 1 records (`batch2_idx = [8,9,10,16,17]` in MATLAB 1-indexed).
2. **Batch 2**: After merging, remove the 5 overlap cells from Batch 2.
3. **Batch 3**: Remove channel 46 (index 37, 0-indexed) upfront. Remove cells where final `QDischarge > 0.885 Ah` (did not reach EOL). Remove noisy cells at indices 3, 40, 41 (0-indexed after prior removal).
4. **Final combined batch**: Optionally remove 5 Batch 1 cells that did not finish cycling (indices `[9,11,13,14,23]` in the combined array, 1-indexed MATLAB).

## Train/Test Split (from paper)

```matlab
test_ind = [1:2:(numBat1+numBat2), 84];   % every other cell + cell 84
train_ind = setdiff(1:(numBat1+numBat2), test_ind);
secondary_test_ind = (numBat-numBat3+1):numBat;  % all Batch 3 cells
```

## MATLAB Utilities

- `EC_dQdV.m` — computes dQ/dV from raw (Q, V) vectors with configurable voltage step (default 4 mV); used to compute the `dQdV` feature.
- `EC_dVdQ.m` — analogous dV/dQ computation.
- `Figure4_mac.m` — reproduces Figure 4 from the paper.

## Output Variable

Cycle life (EOL) is defined as the first cycle where `QDischarge < 0.88 Ah`. If a cell never drops below this threshold, EOL = total cycles + 1.
