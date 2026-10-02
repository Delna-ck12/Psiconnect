# PsiConnect EEG Criticality Analysis

Channel-wise, band-resolved analysis of EEG criticality biomarkers (**DFA** and **fE/I**) computed from the [PsiConnect dataset](https://openneuro.org/datasets/ds006110/versions/1.2.1) (OpenNeuro `ds006110`), comparing resting-state EEG **before** and **after** a 19 mg oral dose of psilocybin.

This repository contains a single-subject (PC001) pipeline that downloads preprocessed FieldTrip-format EEG directly from OpenNeuro, computes DFA and fE/I for five canonical frequency bands plus broadband, and produces topographic maps, channel × band heatmaps, and frequency-resolved spectra.

---

## Background

Detrended Fluctuation Analysis (DFA) and the functional excitation/inhibition ratio (fE/I) are EEG-derived markers linked to long-range temporal correlations and excitation/inhibition balance in cortical dynamics, both of which are discussed in the brain-criticality literature as signatures of a system operating near a critical point. This pipeline compares these markers pre- and post-psilocybin to explore whether, and where, psilocybin shifts resting-state EEG dynamics relative to that regime.

**Note on scope:** this is a single-subject, descriptive/exploratory analysis (n = 1). Figures should be read as illustrative of the pipeline and of one subject's data, not as a statistically validated group effect. 

---

## Dataset

- **Source:** [OpenNeuro ds006110](https://openneuro.org/datasets/ds006110/versions/1.2.1) — *PsiConnect: A Multimodal Neuroimaging Study of Psilocybin-Induced Changes in Brain and Behaviour* (Monash University)
- **Files used:** `derivatives/EEG/cleaned_RELAX/FieldTrip_format/`
  - `sub-PC001_ses-01_task-rest_Clean-ft.mat` — baseline / **pre**-psilocybin resting state
  - `sub-PC001_ses-02_task-rest_Clean-ft.mat` — **post**-psilocybin resting state
- **Preprocessing:** EEG was cleaned upstream by the dataset authors using the [RELAX](https://github.com/NeilwBailey/RELAX) pipeline (filtering, ICA-based artefact removal, bad-channel interpolation) before being exported to FieldTrip format. This repository does not re-clean the data; it reads the cleaned derivative directly.
- **Access:** Files are downloaded programmatically via OpenNeuro's GraphQL API — no manual download or local dataset copy is required to run this pipeline.

---

## Pipeline Overview

| Step | Description |
|------|-------------|
| 1 | Install dependencies (`mne`, `crosci`) |
| 2 | Query the OpenNeuro GraphQL API for the dataset's file tree |
| 3 | Download the PC001 pre/post resting-state `.mat` files |
| 4 | Define the 5 canonical EEG bands + broadband |
| 5 | Load and parse FieldTrip `.mat` structures into NumPy arrays |
| 6 | Compute channel-wise DFA and fE/I per band (via `crosci`) |
| 7 | Validate channel consistency across sessions and consolidate results |
| 8 | Build an MNE `Info` object with a standard 10-20 montage |
| 9 | Plot topomap grids (Pre / Post / Difference) per band, per metric |
| 10 | Plot channel × band difference heatmaps |
| 11 | Compute fine-grained, frequency-resolved DFA/fE/I spectra |
| 12 | Plot the channel-averaged spectrum with canonical bands shaded |
| 13 | Plot the spectrum for individual representative channels |
| 14 | Export long-format CSV summary tables |

Each step is a separate, so the pipeline can be run top-to-bottom in Google Colab or adapted to Spyder/Jupyter.

---

## Metrics Computed

### DFA (Detrended Fluctuation Analysis)
Quantifies long-range temporal correlations in the amplitude envelope of band-limited EEG. The output is a scaling exponent **α**:
- α ≈ 0.5 → uncorrelated (white-noise-like)
- 0.5 < α < 1.0 → persistent power-law (scale-free) correlations
- α ≳ 1.0 → non-stationary / drift / possible artefact

### fE/I (functional Excitation/Inhibition ratio)
Derived from the coupling between a signal's amplitude and its own detrended fluctuation across sliding windows. Centered near 1:
- fE/I ≈ 1 → balanced (near-critical)
- fE/I < 1 → inhibition-dominated
- fE/I > 1 → excitation-dominated

fE/I is only meaningful where DFA indicates genuine long-range correlation (commonly α > 0.6); channels below this threshold are excluded/NaN'd by `crosci` internally.


### Frequency bands

| Band | Range (Hz) |
|---------|------------|
| Delta | 1–4 |
| Theta | 4–8 |
| Alpha | 8–13 |
| Beta | 13–30 |
| Gamma | 30–45 |
| Broadband | 1–45 |

> **Note:** Canonical delta/broadband lower bounds are conventionally 0.5 Hz; they are raised to 1 Hz here because `crosci`'s internal frequency-bin lookup table does not extend below ~1 Hz and raises an `IndexError` otherwise.

---

## Repository Structure

```
.
├── README.md
├── psiconnect_dfa_fei_pipeline.ipynb   # Main analysis notebook (Colab-ready)
├── requirements.txt                    # Python dependencies
└── outputs/                            # Generated on run — figures and CSVs
    ├── PC001_DFA_allbands_topogrid.png
    ├── PC001_fEI_allbands_topogrid.png
    ├── PC001_DFA_difference_row.png
    ├── PC001_fEI_difference_row.png
    ├── PC001_DFA_channel_band_heatmap.png
    ├── PC001_fEI_channel_band_heatmap.png
    ├── PC001_spectrum_pre_post_meanchannels.png
    ├── PC001_spectrum_pre_post_channels.png
    └── PC001_pre_post_allbands_dfa_fei.csv
```

---

## Requirements

```
mne
crosci
numpy
scipy
pandas
matplotlib
requests
```

Install with:

```bash
pip install -r requirements.txt
```

or, in a single Colab cell:

```bash
!pip install mne crosci -q
```

Tested in Google Colab (Python 3.13). No GPU required; runs on CPU.

---

## Usage

1. Open `psiconnect_dfa_fei_pipeline.ipynb` in Google Colab (or Jupyter/Spyder with cell support).
2. Run all cells top to bottom. No manual file download or Google Drive mount is required — the notebook fetches the two target `.mat` files directly from OpenNeuro.
3. Figures and the summary CSV are saved to `/content/` (Colab) or the working directory, and can be copied into `outputs/` before committing.

To analyze a different subject, change:

```python
SUBJECT = "PC001"
```

to any subject ID present in the dataset (see the dataset's `participants.tsv` on OpenNeuro for the full list).

---

## Output Description

- **Topomap grids** (`*_allbands_topogrid.png`): rows = frequency bands, columns = Pre / Post / Difference — the primary spatial summary figure per metric.
- **Difference rows** (`*_difference_row.png`): compact single-row version showing only the post-minus-pre topomap across all bands, useful for quick comparison or slides.
- **Channel × band heatmaps** (`*_channel_band_heatmap.png`): exact per-channel, per-band difference values, complementing the spatially-interpolated topomaps.
- **Frequency-resolved spectra** (`*_spectrum_pre_post_*.png`): DFA/fE/I computed across many fine frequency bins (not just the 5 canonical bands), with canonical band boundaries shaded for reference — shown both as a channel-average and for individual representative electrodes.
- **Summary CSV** (`*_dfa_fei.csv`): long-format table (`condition, band, channel, DFA, fEI`) suitable for further statistical analysis in Python, R, or Excel.

---


## Author

Delna Kuriyakose 
