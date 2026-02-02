# MD → Kinetic Ensemble → SESCA CD Prediction Pipeline

A reproducible computational workflow to predict the circular dichroism (CD) spectrum of a peptide from molecular dynamics (MD) simulations, build a **state-resolved kinetic ensemble** (VAMPnet + MSM), and compare the **probability-weighted theoretical spectrum** to experimental CD data via a **concentration-free rescaling** (scale + baseline offset).

---

## Abstract

This repository provides a complete computational pipeline to predict a peptide CD spectrum from MD simulations. The approach combines:
1. **Multi-replica MD sampling**
2. **Kinetic modeling** (VAMPnet + Markov State Model, MSM) to obtain a state-resolved conformational ensemble and **stationary probabilities**
3. **Extraction of representative frames per state**
4. **CD spectrum prediction** for each frame using **SESCA**
5. **Probability-weighted ensemble spectrum** construction
6. **Quantitative comparison** to experimental CD by rescaling the theoretical spectrum (**scale + baseline offset**) without assuming concentration

The workflow is designed to be reproducible, extensible, and suitable for **model selection across SESCA basis sets**.

---

## Features

- End-to-end **MD → MSM → SESCA → ensemble CD**
- Multi-replica trajectory handling and consistent preprocessing
- State discretization via **VAMPnet** and MSM estimation
- Representative structure selection per state (cluster centers / medoids / stratified sampling)
- SESCA per-frame CD prediction + ensemble averaging
- Fit to experimental CD with **scale + baseline offset** (concentration-independent)
- Built-in evaluation hooks: basis set benchmarking, cross-validation, uncertainty estimates
---

## Requirements

### Core software
- Python >= 3.10
- MD analysis: `mdtraj` and/or `MDAnalysis`
- Kinetic modeling: `deeptime` / `pyemma` *(depending on your implementation)*
- Deep learning (VAMPnet): `pytorch` (or `tensorflow`, if used)
- Scientific stack: `numpy`, `scipy`, `pandas`, `matplotlib`
- **SESCA** installed and accessible from CLI (or via local script wrapper)

### Optional
- `gmx` / GROMACS tools for preprocessing
- `h5py` for model checkpoints
- `jax` if you use accelerated fitting / training

---

## Installation

### 1) Create environment

**Conda (recommended):**
```bash
conda env create -f env/environment.yml
conda activate cd-md-pipeline
```

**Pip:**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r env/requirements.txt
```

### 2) Install / configure SESCA

Ensure SESCA is callable, e.g.:
```bash
sesca --help
```

If SESCA is not on your PATH, set:
```bash
export SESCA_PATH=/path/to/sesca
export PATH="$SESCA_PATH:$PATH"
```

---

## Input data format

### MD data
Provide trajectories and topology for each replica, e.g.:
- `data/md/replica_01.xtc`, `data/md/replica_01.pdb`
- `data/md/replica_02.xtc`, ...

### Experimental CD data
Place experimental spectrum as a 2-column file:
- wavelength (nm), signal (e.g., mdeg or mean residue ellipticity)

Example:
```
# nm    CD
190     -8.12
191     -8.05
...
260      0.10
```

---

## Workflow overview

### Step 1 — Preprocess MD
Typical operations:
- unwrap / center / align
- select atoms (e.g., backbone)
- featurize (dihedrals, distances, contacts)
- time slicing / stride

```bash
python scripts/01_preprocess_md.py --config configs/md_preprocess.yaml
```

Outputs:
- processed trajectories/features in `results/` (and cached arrays)

---

### Step 2 — Train VAMPnet (state decomposition)
```bash
python scripts/02_train_vampnet.py --config configs/vampnet.yaml
```

Outputs:
- trained model checkpoints
- soft assignments per frame
- implied timescales diagnostics (if implemented)

---

### Step 3 — Build MSM
Estimate transition matrix and stationary distribution.

```bash
python scripts/03_build_msm.py --config configs/msm.yaml
```

Outputs:
- discrete trajectories / transition matrix
- implied timescales / Chapman–Kolmogorov tests (optional)
- stationary probabilities: `pi(state)`

---

### Step 4 — Select representative frames per state
Common strategies:
- medoid/centroid in feature space
- top-probability frames
- stratified sampling within state

```bash
python scripts/04_select_frames.py --config configs/msm.yaml
```

Outputs:
- state-wise frame lists
- extracted PDBs in `results/frames/`

---

### Step 5 — Run SESCA per frame
```bash
python scripts/05_run_sesca.py --config configs/sesca.yaml
```

Outputs:
- per-frame predicted CD spectra
- organized by state and SESCA basis set

---

### Step 6 — Build probability-weighted ensemble CD
For each wavelength λ:

\[
CD_{\mathrm{ens}}(\lambda) = \sum_{s} \pi_s \, \langle CD(\lambda)\rangle_{s}
\]

(where the state mean may be over one or multiple representative frames)

```bash
python scripts/06_build_ensemble_cd.py --config configs/sesca.yaml
```

Outputs:
- ensemble spectrum(s) per basis set in `results/ensemble_cd/`

---

### Step 7 — Fit to experiment (scale + baseline offset)
Compare theoretical and experimental spectra after rescaling:

\[
CD_{\mathrm{fit}}(\lambda) = a \, CD_{\mathrm{ens}}(\lambda) + b
\]

This avoids assuming peptide concentration explicitly.

```bash
python scripts/07_fit_to_experiment.py --config configs/fit.yaml
```

Outputs:
- best-fit parameters `(a, b)`
- RMSE / χ² / R² (depending on config)
- overlay plots and residuals in `results/comparison/`

---

## Model selection across SESCA basis sets

If multiple SESCA basis sets are evaluated, the pipeline can rank them using:
- fit error metrics (RMSE, χ²)
- wavelength-window robustness (e.g., 190–260 nm vs 200–260 nm)
- cross-validation on frames/states (optional)

Configure basis sets in `configs/sesca.yaml`.

---

## Reproducibility

- All stages are driven by versioned YAML configs in `configs/`
- Random seeds are set in training and sampling steps (see configs)
- Outputs are stored deterministically under `results/`
- Recommended: record software versions in `results/metadata.json`

---

## Troubleshooting

- **SESCA not found**: ensure `sesca` is in PATH or set `SESCA_PATH`
- **Mismatch in wavelength grids**: enable interpolation of theoretical → experimental grid in `fit.yaml`
- **Poor fit at low wavelengths (≤195 nm)**: consider experimental noise/cutoff and adjust fitting window
- **Overfitting VAMPnet**: enable early stopping / dropout / CV in `vampnet.yaml`

---

## Citation

If you use this pipeline in academic work, please cite:
- SESCA (CD prediction)
- VAMPnet / variational approach for Markov processes
- MSM methodology

Add your preferred BibTeX entries in `CITATION.bib` (optional).

---

## License



---

## Acknowledgements

- SESCA developers
- Open-source MD analysis and kinetic modeling communities
