# Coding Conventions

**Analysis Date:** 2026-06-03

## Project Nature

This is a data science / research codebase composed primarily of **Jupyter notebooks** and **MATLAB scripts**. There is no standalone Python module structure or formal code organization beyond the notebooks.

## Naming Patterns

**Notebooks:**
- Pattern: `[Action][Batch].ipynb` or `[Purpose].ipynb`
- Examples:
  - `BuildPkl_Batch1.ipynb` - Data pipeline stage 1 (MAT → PKL conversion)
  - `BuildPkl_Batch2.ipynb` - Batch 2 equivalent
  - `Load Data.ipynb` - Data pipeline stage 2 (load, merge, clean)

**Variables (Python):**
- Lowercase with underscores: `numBat1`, `summary_IR`, `batch_combined`
- Short descriptive names: `f` for file handles, `i`/`j` for loop indices
- Prefixed by function: `summary_` for summary data, `batch_` for batch structures
- Examples from code:
  - `bat_dict` - battery dictionary (main data structure)
  - `batch1`, `batch2`, `batch3` - batch-specific data
  - `test_ind`, `train_ind` - index arrays for splits
  - `summary_QD` - discharge capacity summary

**Variables (MATLAB):**
- CamelCase: `numBat1`, `batch_combined`, `bat_label`
- Descriptive abbreviations: `IR` (internal resistance), `QD` (discharge capacity), `QC` (charge capacity)
- Examples:
  - `batch_combined` - merged all three batches
  - `endcap3` - end-of-life capacity for batch 3
  - `rind` - row indices to remove
  - `nind` - noise indices to remove

**Data Keys (Nested Dictionaries):**
- Cell identifiers: `b1c0`, `b1c43` (batch `b{N}` + cell `c{ID}`)
- Abbreviations for measurements:
  - `QD` - discharge capacity (Ah)
  - `QC` - charge capacity (Ah)
  - `IR` - internal resistance (Ohm)
  - `Tavg`, `Tmin`, `Tmax` - average/min/max temperature (°C)
  - `dQdV` - differential capacity (Ah/V)
  - `Qdlin` - linearly interpolated discharge capacity
  - `Tdlin` - linearly interpolated temperature

## Code Style

**Formatting:**
- No automated formatter configured (no `.prettierrc`, `.black`, etc.)
- Python notebooks use ad-hoc spacing and indentation
- MATLAB uses standard indentation (4 spaces)

**Linting:**
- No linting tool configured (no `.eslintrc`, `.pylintrc`, etc.)
- Code follows basic Python/MATLAB conventions without enforced rules

**Comments:**
- MATLAB: Header comments with `%` lines for section demarcation:
  ```matlab
  %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
  % Script to show an example of loading the dataset...                  %
  %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
  ```
- Python notebooks: Markdown cells separate logical sections
- Inline comments explain non-obvious logic (e.g., batch overlap corrections)

## Import Organization

**Python Notebooks:**
- Standard library first: `pickle`, `numpy`, `scipy`
- Data processing: `pandas`, `h5py`, `scipy.io`
- Plotting: `matplotlib.pyplot`
- Typical order (from notebooks):
  ```python
  import numpy as np
  import matplotlib.pyplot as plt
  import pickle
  import h5py
  import scipy.io
  ```

**MATLAB:**
- Loaded with `load()` (no import-like mechanism)
- Added to path with `addpath()` for utility functions
- Example from `Figure4_mac.m`:
  ```matlab
  addpath('./Code_Utils')
  ```

## Data Structures

**Primary Container (Python):**
- Dictionary of dictionaries: `bat_dict[cell_key] = {...}`
- Cell key format: `'b1c0'`, `'b2c43'`, `'b3c19'`
- Structure per cell:
  ```python
  {
      'cycle_life': int,           # cycles to EOL (80% capacity)
      'charge_policy': str,        # e.g. "3.6C(80%)-4C"
      'summary': {                 # per-cycle summary data
          'cycle': array,          # cycle indices
          'QD': array,             # discharge capacity
          'QC': array,             # charge capacity
          'IR': array,             # internal resistance
          'Tavg': array,           # average temperature
          'Tmin': array,
          'Tmax': array,
          'chargetime': array,     # minutes to 80% SOC
      },
      'cycles': {                  # per-cycle detailed data
          '0': {                   # cycle number as string key
              'I': array,          # current
              'Qc': array,         # charge capacity
              'Qd': array,         # discharge capacity
              'Qdlin': array,      # linearly interpolated
              'T': array,          # temperature
              'Tdlin': array,
              'V': array,          # voltage
              'dQdV': array,       # differential capacity
              't': array,          # time (minutes)
          }
      }
  }
  ```

**MATLAB Equivalents:**
- Struct arrays: `batch(i)` replaces dict key `b1c{i}`
- Fields accessed with dot notation: `batch(i).summary.QDischarge`

## Error Handling

**Strategy:** Minimal - research code assumes valid data inputs.

**Patterns:**
- Data cleaning is explicit: cells are removed if they don't meet criteria
  - Batch 1 cells with no EOL marking: manually deleted
  - Batch 3 cells with `QDischarge > 0.885`: identified via `find()` and removed
  - Noisy cells: hardcoded index lists removed (`nind = [3, 40:41]`)
- No try/catch blocks observed
- File I/O assumes files exist; no explicit error handling

**EOL Definition:**
- Cycle life = first cycle where `QDischarge < 0.88` Ah
- If never reached: `EOL = total_cycles + 1`

## Logging

**Framework:** `print()` (Python) / `disp()` (MATLAB)

**Patterns:**
- Minimal logging; notebooks rely on cell outputs
- Key checkpoints printed:
  - Cell counts: `numBat1`, `numBat2`, `numBat3`
  - Data shapes and keys after each transformation
- MATLAB plots display results inline

## Module Organization

**Utility Functions (MATLAB):**
- `EC_dQdV.m` - Computes dQ/dV (differential capacity) from (Q, V) vectors
  - Parameters: `Q` (capacity), `Ewe` (voltage), `Q_scale` (normalization), `step` (voltage step size, default 4 mV)
  - Returns: `[V, dQdV]` - voltage and differential capacity
  - Located: `./EC_dQdV.m`

- `EC_dVdQ.m` - Computes dV/dQ (analogous to dQ/dV)
  - Located: `./EC_dVdQ.m`

- `Figure4_mac.m` - Reproduction script for Figure 4 from paper
  - Located: `./Figure4_mac.m`

**Data Processing Steps:**
1. `BuildPkl_Batch{1,2,3}.ipynb` - Load HDF5 MAT files via `h5py`, extract per-cell data, serialize to pickle
2. `Load Data.ipynb` - Load pickles, merge batches, apply cleaning rules, compute train/test splits

## Constants and Thresholds

**EOL Threshold:** `0.88` Ah (discharge capacity)

**Voltage Step Size:** `0.004` V (4 mV) for dQ/dV computation

**Batch 1 ↔ Batch 2 Overlap Cycle Counts:**
- `add_len = [662, 981, 1060, 208, 482]` (MATLAB: 1-indexed)
- Cells: `b1c0`, `b1c1`, `b1c2`, `b1c3`, `b1c4`

**Batch 3 Cleaning Indices (0-indexed after removal of channel 46):**
- `rind` - cells where final `QDischarge > 0.885`
- `nind = [3, 40, 41]` - noisy cells removed

## Key Design Decisions

**Data Format:**
- MAT → PKL conversion centralizes data reading (stage 1)
- Pickle chosen for Python compatibility; MATLAB version uses struct arrays
- Nested dicts for flexibility in accessing cycle-level vs. summary data

**Batch Management:**
- Batch 1 and 2 overlap resolved by appending Batch 2 cycles to Batch 1 cells
- Batch 3 cleaned upfront (before any merges) due to missing channels and noisy cells
- Final combined batch allows optional removal of incomplete Batch 1 cells

**Train/Test Split:**
- Deterministic indices specified to match published paper
- Three groups: training (Batch 1+2 odd-indexed cells), test (Batch 1+2 even-indexed + cell 84), secondary test (Batch 3)

---

*Convention analysis: 2026-06-03*
