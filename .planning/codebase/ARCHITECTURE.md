<!-- refreshed: 2026-06-03 -->
# Architecture

**Analysis Date:** 2026-06-03

## System Overview

This is a **data processing and analysis pipeline** for lithium-ion battery cycling experiments. The system converts raw battery telemetry from MATLAB struct format to Python pickle format, applies batch-specific data cleaning rules, and prepares merged datasets for predictive modeling. The pipeline follows a three-stage batch processing architecture with explicit merge and validation steps.

```text
┌─────────────────────────────────────────────────────────────────┐
│           Raw MATLAB Data (HDF5-formatted .mat files)            │
│  `./Data/*.mat` (not tracked in git)                             │
└──────────────────────┬──────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Batch 1     │ │  Batch 2     │ │  Batch 3     │
│  46 cells    │ │  48 cells    │ │  45 cells    │
│  Notebook:   │ │  Notebook:   │ │  Notebook:   │
│ BuildPkl_1.  │ │ BuildPkl_2.  │ │ BuildPkl_3.  │
│ ipynb        │ │ ipynb        │ │ ipynb        │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       │ Extract cells  │ Extract cells  │ Extract cells
       │ via h5py       │ via h5py       │ via h5py
       │                │                │
       ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│           Serialized Pickle Dictionaries                         │
│  `batch1.pkl` | `batch2.pkl` | `batch3.pkl`                     │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│      Data Merging & Cleaning Layer                              │
│      `Load Data.ipynb` / `LoadData.m`                            │
│                                                                  │
│  • Detect B1↔B2 overlap (5 cells)                               │
│  • Merge overlapped cycles into Batch 1 records                │
│  • Remove noisy/incomplete cells per batch rules               │
│  • Validate 80% capacity threshold (EOL definition)            │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│      Merged Dataset                                              │
│      `bat_dict` (Python dict or struct array in MATLAB)         │
│                                                                  │
│      Final cell count: 109 (after validation cleanup)          │
│      Structure: {cell_id: {cycle_life, charge_policy,           │
│                            summary, cycles}}                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│      Modeling & Visualization (Licensed code — not in repo)     │
│      `Figure4_mac.m` — Reproduces paper Figure 4                │
│      Predictive model training — Contact braatz@mit.edu        │
└─────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| **Data Ingestion (Batch 1)** | Extract 46 cell records from Batch 1 HDF5, convert arrays, serialize to pickle | `BuildPkl_Batch1.ipynb` |
| **Data Ingestion (Batch 2)** | Extract 48 cell records from Batch 2 HDF5, convert arrays, serialize to pickle | `BuildPkl_Batch2.ipynb` |
| **Data Ingestion (Batch 3)** | Extract 45 cell records from Batch 3 HDF5, convert arrays, serialize to pickle | `BuildPkl_Batch3.ipynb` |
| **Batch Merging & Cleaning** | Detect batch overlaps, merge cycles, apply batch-specific removal rules, validate EOL threshold | `Load Data.ipynb` / `LoadData.m` |
| **Feature Extraction (Utilities)** | Compute dQ/dV (differential capacity) and dV/dQ from raw Q-V curves via configurable step size | `EC_dQdV.m`, `EC_dVdQ.m` |
| **Publication Figure** | Reproduce Nature Energy paper Figure 4 using merged dataset + external diagnostic data | `Figure4_mac.m` |

## Pattern Overview

**Overall:** Pipeline-based ETL (Extract, Transform, Load) with explicit batch reconciliation.

**Key Characteristics:**
- **Immutable source data**: Raw `.mat` files never modified; only pickles and MATLAB structs are working copies.
- **Deterministic cleaning**: Batch-specific cell removal rules are hardcoded and documented in `LoadData.m` / `CLAUDE.md` for reproducibility.
- **Three-stage ingestion**: Each batch processed independently, then merged with overlap detection to avoid double-counting cells that spanned experimental runs.
- **Dual-language support**: Python notebooks for pickle generation; MATLAB for analysis and licensed modeling code. Both consume identical data structures.
- **Early validation**: EOL (80% capacity threshold) checked after merge; invalid cells removed before downstream analysis.

## Layers

**Stage 1: Data Extraction (Notebooks)**
- Purpose: Convert HDF5-formatted MATLAB struct arrays into Python-serializable nested dictionaries; one notebook per experimental batch.
- Location: `BuildPkl_Batch1.ipynb`, `BuildPkl_Batch2.ipynb`, `BuildPkl_Batch3.ipynb`
- Contains: h5py file reading, array flattening, dict nesting, pickle serialization
- Depends on: Raw `.mat` files in `./Data/` (external, not tracked)
- Used by: Merge layer (reads pickles)

**Stage 2: Data Merge & Cleaning (Python/MATLAB)**
- Purpose: Merge three batches, detect/resolve overlaps (5 cells cycling from B1→B2), remove invalid/noisy cells per batch-specific rules, define train/test split.
- Location: `Load Data.ipynb` (Python) or `LoadData.m` (MATLAB)
- Contains: Batch overlap detection (hardcoded cell indices), cycle appending, removal of incomplete/noisy cells, EOL validation, train/test index generation
- Depends on: Pickles from Stage 1
- Used by: Modeling code (licensed, external)

**Utility Layer (MATLAB)**
- Purpose: Electrochemistry calculations (dQ/dV, dV/dQ) used during feature extraction and figure generation.
- Location: `EC_dQdV.m`, `EC_dVdQ.m`
- Contains: Configurable voltage/capacity step stepping, numerical differentiation, NaN removal
- Depends on: Charge/discharge capacity and voltage arrays
- Used by: `Figure4_mac.m`, modeling code

**Analysis Layer (MATLAB)**
- Purpose: Reproduce paper results and visualize early-cycle signatures predicting cycle life.
- Location: `Figure4_mac.m`
- Contains: Loading merged batch data, diagnostic cycling data, dQ/dV computation, multi-panel figure assembly
- Depends on: Merged batch struct, external diagnostic data files
- Used by: Publication validation

## Data Flow

### Primary Flow: Data Preparation (Stages 1 & 2)

1. **User obtains raw data** — Download `.mat` files from https://data.matr.io/1/, place in `./Data/` (not in git)
2. **Extract Batch 1** — `BuildPkl_Batch1.ipynb` opens HDF5, iterates cells 0–45, extracts summary + cycle arrays, saves `batch1.pkl` (`[file:cell-5]`)
3. **Extract Batch 2** — `BuildPkl_Batch2.ipynb` extracts cells 0–47, saves `batch2.pkl` (same pattern)
4. **Extract Batch 3** — `BuildPkl_Batch3.ipynb` extracts cells 0–44, saves `batch3.pkl` (same pattern)
5. **Merge in Python** — `Load Data.ipynb` loads all three pickles in order (`[file:cell-1,3,8]`)
6. **Detect B1↔B2 overlap** — Hardcoded: 5 cells (`b2c7`, `b2c8`, `b2c9`, `b2c15`, `b2c16`) continued from B1 (`b1c0`–`b1c4`). Append their B2 cycles to B1 records and update `cycle_life` (`[file:cell-4,5]`)
7. **Remove from B2** — Delete the 5 overlap cells from Batch 2 (`[file:cell-6]`)
8. **Clean Batch 3** — Remove by index: channel 46 (noisy), cells with final QDischarge > 0.885 Ah (incomplete), noisy indices 3, 40, 41 (`[file:cell-8]`)
9. **Merge all** — Combine B1, B2, B3 dicts into `bat_dict`; final count = ~109 cells after optional removal of 5 incomplete B1 cells (`[file:cell-11]`)
10. **Generate EOL labels** — For each cell, find first cycle where QDischarge < 0.88 Ah; store as ground-truth labels (`LoadData.m:71–81`)
11. **Split train/test** — Use paper's indices: test = every other cell from B1∪B2 + cell 84; train = remainder; secondary_test = all of B3 (`LoadData.m:91–93`)

**Data structure at each step:**
- After extraction: `bat_dict[cell_id] = {'cycle_life': int, 'charge_policy': str, 'summary': {...}, 'cycles': {'0': {...}, '1': {...}, ...}}`
- After merge: Same structure, but cell keys merged globally (e.g., `b1c0`, `b2c0`, `b3c0` coexist); B1 cells have extended `summary` and `cycles` arrays
- After cleaning: Only valid cells remain; `bat_dict` ready for modeling

### Secondary Flow: Figure Generation & Validation

1. **Load merged batch struct** (`Figure4_mac.m:14`) — Read the MATLAB version of merged data
2. **Load diagnostic cycling data** — External `.mat` files with additional controlled experiments (4C/6C/8C charge rates, initial/final data)
3. **Compute dQ/dV** — Call `EC_dQdV.m` on early cycles (cycle 10) and late cycles (cycle 100 / cycle at EOL)
4. **Overlay dQdV curves** — Plot early-cycle signature, late-cycle signature, and end-of-life dQdV to show degradation
5. **Export figure** — Save as `.png` and `.fig` to `./Figures/`

**State Management:**
- **No persistent state**: Each notebook/script is stateless; all data flows through pickle/struct files
- **Deterministic**: Same input files always produce same outputs (cell removal rules are hardcoded, not learned)
- **Validation point**: EOL threshold (0.88 Ah) enforced after merge; cells not reaching it are flagged or removed

## Key Abstractions

**Cell Record:**
- Purpose: Encapsulate all telemetry for a single lithium-ion cell across its full cycling life.
- Examples: `bat_dict['b1c0']`, `bat_dict['b2c15']`, `batch_combined(3)` in MATLAB
- Pattern: Nested dict in Python (cycle number keys as strings); struct array in MATLAB. Both store `cycle_life` (int), `charge_policy` (str), `summary` (per-cycle aggregates), `cycles` (per-cycle raw samples).

**Batch:**
- Purpose: Group cells from a single experimental run (e.g., Batch 1 = 46 cells tested in May 2017)
- Examples: `batch1`, `batch2`, `batch3` (pickles); `batch`, `batch_combined` (MATLAB structs)
- Pattern: Dict of cell records keyed by batch-cell IDs (`b1c0`–`b1c45`, `b2c0`–`b2c47`, `b3c0`–`b3c44`). After merge, one global namespace.

**Summary Statistics:**
- Purpose: Aggregate measurements across cycles (one value per cycle).
- Examples: `summary['QD']` = discharge capacity per cycle; `summary['Tavg']` = mean temperature per cycle
- Pattern: Numpy arrays (Python) or column vectors (MATLAB), indexed by cycle number.

**Cycle Data:**
- Purpose: Time-series measurements within a single charge/discharge event.
- Examples: `cycles['10']['V']` = voltage samples during cycle 10; `cycles['10']['dQdV']` = differential capacity curve
- Pattern: Nested dict/struct with keys: `I` (current), `V` (voltage), `Qc`/`Qd` (charge/discharge capacity), `T`/`Tdlin` (temperature), `dQdV` (computed feature), `t` (time).

**Overlap Cell:**
- Purpose: Cells that continued cycling from Batch 1 into Batch 2 experimental run.
- Examples: `b1c0`–`b1c4` (indices 0–4 in Batch 1) correspond to `b2c7`–`b2c16` (hardcoded mapping in `Load Data.ipynb:cell-4`)
- Pattern: Detected during merge; their Batch 2 cycles appended to Batch 1 records; Batch 2 records deleted to avoid duplication.

## Entry Points

**Stage 1 — Data Extraction:**
- **Location:** `BuildPkl_Batch1.ipynb` (cell 0)
- **Triggers:** User runs notebook manually (via Jupyter or papermill)
- **Responsibilities:** Opens `./Data/2017-05-12_batchdata_updated_struct_errorcorrect.mat`, iterates cells, serializes to `batch1.pkl`

**Stage 2 — Data Merge & Cleaning (Python):**
- **Location:** `Load Data.ipynb` (cell 0, imports)
- **Triggers:** User runs notebook after Stage 1 pickles exist
- **Responsibilities:** Loads pickles, applies merge + cleaning rules, generates `bat_dict`, computes EOL labels, defines train/test split

**Stage 2 — Data Merge & Cleaning (MATLAB):**
- **Location:** `LoadData.m` (line 1, `clear; close all; clc`)
- **Triggers:** User runs script in MATLAB after raw `.mat` files placed in `./Data/`
- **Responsibilities:** Loads three `.mat` files directly, applies identical merge + cleaning rules, outputs `batch_combined` struct + train/test indices

**Analysis & Validation:**
- **Location:** `Figure4_mac.m` (line 1, copyright notice)
- **Triggers:** User runs after data preparation (requires merged batch struct + diagnostic data files)
- **Responsibilities:** Reproduces Nature Energy paper Figure 4, validates early-cycle dQ/dV signatures

## Architectural Constraints

- **Single-threaded event loop:** Both Python (Jupyter kernel) and MATLAB operate sequentially. No concurrency; notebooks run cell-by-cell in order.
- **Global state (notebooks):** Cell variables persist across cells in Jupyter execution context. Running cells out of order breaks dependencies. `Load Data.ipynb` requires `batch1`, `batch2`, `batch3` already loaded from earlier cells.
- **File-based IPC:** Batches communicate via pickle/MAT files. No shared memory or network protocols.
- **Hardcoded indices:** Batch overlap detection and cell removal rules are baked into notebooks/scripts (e.g., `batch2_idx = [8,9,10,16,17]` in `LoadData.m:19`). Changes require manual edit + re-run.
- **Immutable external data:** Raw `.mat` files in `./Data/` are never modified. Extraction is append-only (new pickles written, originals left intact).
- **No circular dependencies:** Data flows one direction: Stage 1 → Stage 2 → Modeling. No feedback loops.
- **Voltage binning:** dQ/dV computation uses configurable voltage step (default 4 mV in `EC_dQdV.m`). Step size affects feature granularity; must be consistent across analyses.

## Anti-Patterns

### Hardcoded Batch Indices

**What happens:** Cell removal rules for Batch 3 use literal indices (e.g., `batch3([3, 40, 41]) = []`). If cells are reordered or external data changes, indices become invalid.

**Why it's wrong:** Indices are fragile and undocumented. A future user reviewing `LoadData.m:39–42` must cross-reference with external documentation (`CLAUDE.md`) to understand why these specific indices are removed. If the source `.mat` file changes slightly (e.g., cells reordered), indices silently point to wrong cells.

**Do this instead:** Use cell identifiers (barcodes/serial numbers) or metadata-based filtering. For example, instead of `batch3(nind) = []`, match cells by `batch3(i).barcode` and conditional logic. This makes removal rules explicit and traceable.

### Floating-Point EOL Threshold

**What happens:** `LoadData.m:44` uses `if batch_combined(i).summary.QDischarge(end) < 0.88` as the EOL boundary. If a cell drops from 0.880001 to 0.879999 due to measurement noise, EOL label can shift by multiple cycles.

**Why it's wrong:** 0.88 Ah is an arbitrary threshold without uncertainty margins. No discussion of measurement precision or why 0.88 specifically. Cells near the boundary are sensitive to rounding and sorting order.

**Do this instead:** Validate measurement precision (state uncertainty as ±X mAh), and either (a) use a conservative margin (e.g., 0.87 for 0.88 target), or (b) fit a smooth degradation curve and define EOL as intersection with threshold with confidence interval. Document assumptions.

### Index Mapping Without Validation

**What happens:** `Load Data.ipynb:cell-4` hardcodes overlap mapping (`batch1_keys = ['b1c0', 'b1c1', ...]` paired with `batch2_keys = ['b2c7', 'b2c8', ...]` and `add_len = [662, 981, ...]`). If someone edits the lists but forgets `add_len`, or if lists fall out of sync, cycle counts silently diverge.

**Why it's wrong:** No assertion that lengths match, no logging of appended cycles, no post-merge validation that B1 cycle counts increased by expected amounts. A bug here produces subtly corrupted data.

**Do this instead:** Wrap in a function with pre/post-conditions:
```python
def merge_overlap_cells(batch1, batch2, overlap_pairs, cycle_counts):
    assert len(overlap_pairs) == len(cycle_counts), "Mismatch in overlap mapping"
    for (b1_key, b2_key), expected_cycles in zip(overlap_pairs, cycle_counts):
        assert len(batch2[b2_key]['cycles']) == expected_cycles, f"Unexpected cycles for {b2_key}"
        # ... merge logic ...
        actual_appended = len(batch1[b1_key]['cycles']) - old_len
        assert actual_appended == expected_cycles, f"Merge failed for {b1_key}"
```

### No Error Handling for Missing Files

**What happens:** `BuildPkl_Batch1.ipynb:cell-1` assumes `./Data/2017-05-12_...mat` exists. If the file is missing or path is wrong, `h5py.File()` raises `FileNotFoundError`, halting the notebook without clear guidance.

**Why it's wrong:** Users downloading data from external sources may place files in different directories. Error message doesn't suggest corrective action (e.g., "Expected ./Data/2017-05-12_...; download from https://data.matr.io/1/").

**Do this instead:** Add a helper at the top of each notebook:
```python
import os
expected_file = './Data/2017-05-12_batchdata_updated_struct_errorcorrect.mat'
if not os.path.exists(expected_file):
    raise FileNotFoundError(f"Missing {expected_file}. Download from https://data.matr.io/1/ and place in ./Data/")
```

## Error Handling

**Strategy:** Fail-fast with manual intervention. Notebooks halt if files missing or data malformed; no silent fallbacks.

**Patterns:**
- Missing `.mat` file: `h5py.File()` raises exception; user must download and retry.
- Unexpected HDF5 structure: Array access (e.g., `f[batch['cycle_life'][i,0]].value`) raises `KeyError` if keys don't exist; user must inspect raw `.mat` file.
- Batch overlap detection: Hardcoded cell indices (`batch2_idx = [8,9,10,16,17]`) assume these specific cells exist in correct positions; no validation if indices are out of range or refer to different cells than intended.
- EOL threshold: `LoadData.m:44` silently skips validation if a cell never drops below 0.88 Ah; cycle_life set to total cycles + 1 (no error, just assumption).

## Cross-Cutting Concerns

**Logging:** Minimal. Notebooks use `print()` (Python) or `disp()` (MATLAB) for user feedback. No structured logging framework.

**Validation:** Post-hoc only. After merge, user visually inspects `bat_dict` keys or plots (e.g., `plt.plot(bat_dict['b1c43']['summary']['cycle'], ...)` in `BuildPkl_Batch1.ipynb:cell-7`) to confirm data integrity.

**Batch consistency:** Relies on external documentation (`CLAUDE.md`) to specify hardcoded rules. No in-code metadata (e.g., struct field with batch ID, date range, cell count) to validate consistency.

---

*Architecture analysis: 2026-06-03*
