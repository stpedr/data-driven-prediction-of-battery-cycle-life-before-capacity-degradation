# Technology Stack

**Analysis Date:** 2026-06-03

## Languages

**Primary:**
- Python 3.7.10 - Data processing, analysis, and model development
- MATLAB - Legacy analysis scripts and visualization (optional)

**Secondary:**
- MATLAB - Data loading and figure generation utilities

## Runtime

**Environment:**
- Python 3.7.10 (recommended)
- Modern Python versions also supported (tested with 3.11.15)

**Package Manager:**
- pip with requirements.txt

## Frameworks

**Scientific Computing:**
- NumPy 1.22.0 - Numerical operations on arrays
- SciPy 1.6.1 - Scientific computing utilities (e.g., `scipy.io` for MATLAB file operations)
- Pandas 1.2.3 - Data manipulation and analysis
- Matplotlib 3.3.4 - Data visualization and plotting

**Data Processing:**
- h5py 2.8.0 - HDF5 file format reading (battery cycle data from .mat files)

**Development:**
- Jupyter - Interactive notebook environment for analysis and data exploration

## Key Dependencies

**Critical:**
- `h5py` 2.8.0 - Loads HDF5-formatted battery test data from MATLAB `.mat` files (essential for `BuildPkl_Batch*.ipynb`)
- `numpy` 1.22.0 - Core numerical operations for battery cycle data processing
- `pandas` 1.2.3 - Data aggregation and manipulation across multiple battery batches
- `matplotlib` 3.3.4 - Plotting cycle life curves and diagnostic visualizations
- `scipy` 1.6.1 - MATLAB file I/O (`scipy.io`)

**Development:**
- `jupyter` - Required to execute `.ipynb` notebooks

## Configuration

**Environment:**
- No `.env` files or environment-based configuration detected
- All paths are file-based (local filesystem)

**Data Format:**
- Input: MATLAB `.mat` files (HDF5-encoded) containing battery test data
- Processing: Python pickle format (`.pkl`) for serialized dictionary structures
- Output: Pickle files with nested dictionaries organized by battery cell

## Platform Requirements

**Development:**
- Python 3.7.10+ installed with pip
- Jupyter installed for notebook execution
- MATLAB installed (optional, for legacy `.m` scripts)
- Dependencies listed in `requirements.txt`

**Production:**
- Python 3.7.10+ runtime
- All dependencies from `requirements.txt` installed
- Local filesystem access to `.pkl` data files
- No network connectivity required

---

*Stack analysis: 2026-06-03*
