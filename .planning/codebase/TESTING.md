# Testing Patterns

**Analysis Date:** 2026-06-03

## Test Framework

**Status:** No formal testing framework detected.

- No test runner configured (pytest, unittest, vitest, etc.)
- No test configuration files: `pytest.ini`, `tox.ini`, `setup.cfg` not present
- No test file naming convention enforced (no `test_*.py` or `*_test.py` files)

**Assertion Library:** Not applicable

**Validation Strategy:**
- Manual validation via notebook cell outputs
- Visual inspection of plots (matplotlib) to verify data correctness
- Assertions embedded in data cleaning logic (implicit checks)

## Testing Approach

**Manual Verification in Notebooks:**

The codebase relies entirely on in-notebook validation:

1. **Data Loading Verification** (`BuildPkl_Batch*.ipynb`):
   - After loading MAT files via `h5py`, list available keys to confirm structure
   - Example:
     ```python
     list(f.keys())  # Display HDF5 structure
     batch['summary'].shape[0]  # Count cells
     ```
   - Plot sample cell data to verify integrity:
     ```python
     plt.plot(bat_dict['b1c43']['summary']['cycle'], 
              bat_dict['b1c43']['summary']['QD'])
     ```

2. **Data Merging Verification** (`Load Data.ipynb`):
   - Count cells after each transformation:
     ```python
     numBat1 = len(batch1.keys())
     numBat2 = len(batch2.keys())
     numBat3 = len(batch3.keys())
     ```
   - Plot all batteries' discharge capacity curves to inspect degradation:
     ```python
     for i in bat_dict.keys():
         plt.plot(bat_dict[i]['summary']['cycle'], 
                  bat_dict[i]['summary']['QD'])
     ```

3. **Batch Cleaning Validation**:
   - Manually verify removed cells match documented rules
   - No automated assertions; cleaning is done via explicit `del` statements with comments
   - Example from `Load Data.ipynb`:
     ```python
     # Manual deletion with inline comment
     del batch1['b1c8']  # Does not reach 80% capacity
     del batch1['b1c10']
     ```

## Data Integrity Checks (Implicit)

**EOL Computation** (`Load Data.ipynb`):
- All cells must have a `cycle_life` field populated
- Implicit check: cells that never drop below 0.88 Ah are assigned `EOL = total_cycles + 1`
- MATLAB version in `LoadData.m`:
  ```matlab
  if batch_combined(i).summary.QDischarge(end) < 0.88
      bat_label(i) = find(batch_combined(i).summary.QDischarge < 0.88, 1);
  else
      bat_label(i) = size(batch_combined(i).cycles, 2) + 1;
  end
  ```

**Batch Overlap Correction** (`Load Data.ipynb`):
- 5 cells continued from Batch 1 into Batch 2
- Cycles from Batch 2 appended to Batch 1 records
- Verification: manually check that appended cycle counts match `add_len`:
  ```python
  add_len = [662, 981, 1060, 208, 482]
  # After append, len(batch1[bk]['summary']['cycle']) should increase by add_len[i]
  ```

**Batch 3 Cleaning** (`Load Data.ipynb`):
- Cells where final `QDischarge > 0.885` Ah are removed (did not reach EOL)
- Noisy channels removed by hardcoded index lists
- Manual spot-checks via plotting

## Test Coverage

**Coverage:** Not applicable — no formal tests exist.

**What IS Tested (manually):**
- Data loading from HDF5 MAT files
- Dictionary serialization (pickle) / deserialization
- Batch merging and overlap resolution
- Train/test split indices match paper specifications
- Visualization of all 124 battery degradation curves

**What IS NOT Tested:**
- Unit tests for utility functions (`EC_dQdV.m`, `EC_dVdQ.m`) — functions are used inline in MATLAB scripts
- Automated validation of dQ/dV computation correctness
- Error handling for missing or corrupted data files
- Edge cases in batch cleaning logic

## Documentation and Reproducibility

**Batch Cleaning Rules** (documented in `CLAUDE.md`):
- Batch 1 ↔ Batch 2 overlap: 5 cells continue; their Batch 2 cycles appended to Batch 1
- Batch 3 cleaning: Remove channel 46 upfront; remove cells not reaching EOL; remove noisy cells at hardcoded indices
- These rules must be replicated manually in any downstream analysis

**Train/Test Split** (from published paper):
- Indices hardcoded to match paper exactly:
  ```python
  test_ind = np.hstack((np.arange(0,(numBat1+numBat2),2), 83))
  train_ind = np.arange(1,(numBat1+numBat2-1), 2)
  secondary_test_ind = np.arange(numBat-numBat3, numBat)
  ```

**Matlab Equivalent** (`LoadData.m`, lines 101-104):
```matlab
test_ind = [1:2:(numBat1+numBat2), 84];  % 1-indexed
train_ind = 1:(numBat1+numBat2);
train_ind(test_ind) = [];
secondary_test_ind = numBat-numBat3+1:numBat;
```

## Verification Artifacts

**Jupyter Notebooks as Test Reports:**
- Cell outputs serve as test results
- Plots of all 124 cells' capacity curves provide visual regression detection
- Printed counts (`numBat1 = 41`, `numBat2 = 43`, `numBat3 = 40`, `numBat = 124`) document expected dataset size

**MATLAB Scripts:**
- `Figure4_mac.m` reproduces a published figure; output PDF/PNG serves as validation artifact
- Inline plots of dQ/dV curves for specific cells confirm feature extraction correctness

## Known Limitations

1. **No Unit Tests:** Utility functions not isolated or tested in isolation
2. **No Error Handling:** Assumes valid data; crashes if files missing or malformed
3. **Manual Data Cleaning:** Batch cleaning logic is implicit in notebook cells; easy to miss or replicate incorrectly
4. **No Regression Tests:** No automated checks that future changes don't break data pipeline
5. **Hardcoded Indices:** Train/test split and cleaning rule indices embedded in notebooks; difficult to parameterize or audit

## Recommended Improvements (Future)

- Parameterize batch cleaning rules into a configuration file (YAML/JSON)
- Add `assert` statements for critical data integrity checks (cell counts, EOL thresholds)
- Create standalone Python module with testable functions (data loading, merging, cleaning)
- Implement pytest suite for batch cleaning logic
- Add pre-execution validation: verify data files exist and contain expected structure

---

*Testing analysis: 2026-06-03*
