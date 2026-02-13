# Astrocyte Calcium Signaling Analysis Pipeline

A Python-based post-processing pipeline for analyzing astrocyte calcium event data exported from AQuA2. This pipeline normalizes calcium event features, generates publication-ready visualizations, and performs statistical comparisons across experimental groups.

## Overview

This project processes confocal calcium imaging data from astrocyte recordings. Raw calcium events are first detected and quantified using **AQuA2** (Activity Quantification and Analysis), and the resulting feature exports are then analyzed through this pipeline for normalization, visualization, and statistical testing.

### Pipeline Workflow

```
Confocal Calcium Imaging
        │
        ▼
   AQuA2 Processing
   (Event Detection & Feature Extraction)
        │
        ▼
  ┌─────────────────────────────────┐
  │  analysis.ipynb                 │
  │  ├─ Load AQuA2 CSV exports      │
  │  ├─ Filter events by frame      │
  │  ├─ Normalize to baseline       │
  │  ├─ Timepoint plots             │
  │  ├─ Statistical analysis        │
  │  └─ Export normalized CSVs      │
  └─────────────┬───────────────────┘
                │
                ▼
  ┌─────────────────────────────────┐
  │  analysis_time.ipynb            │
  │  ├─ Load normalized CSVs        │
  │  ├─ Bin events by frame groups  │
  │  └─ Time-series visualization   │
  └─────────────────────────────────┘
```

## Experimental Groups

| Group | Description | Drug Condition |
|-------|-------------|----------------|
| **WT** | Wild Type | Psilocybin |
| **AV** | Antagonist Volinanserin | Psilocybin |
| **IP** | IP3R2 cKO | Psilocybin |
| **CE** | CalEx | Psilocybin |

Each group is recorded across three conditions: **Baseline → Drug → Washout**

## Features Analyzed

The pipeline analyzes 16 features extracted by AQuA2:

**Intensity & Magnitude**
- Max ΔF, Max ΔF/F
- AUC metrics (dat, dF, dF/F)

**Morphological**
- Area, Perimeter, Circularity

**Temporal Dynamics**
- Event overlay duration
- Duration at 50% and 10% thresholds
- Rising duration (10%–90%)
- Decaying duration (90%–10%)

**Network**
- Number of co-localized events
- Co-localized events with similar size
- Maximum simultaneous events

### Normalization

All features are normalized using **fold-change** relative to the baseline condition median for the same slice:

```
normalized_value = value / baseline_median
```

Interpretation: 2.0 = doubled relative to baseline, 0.5 = halved.

## Statistical Methods

The pipeline uses a non-parametric statistical framework:

```
Kruskal-Wallis (omnibus test across groups)
        │
        ▼  (if significant after FDR correction)
Dunn's test (pairwise comparisons vs WT control)
        │
        ▼
FDR correction (Benjamini-Hochberg)
```

- **Kruskal-Wallis test** — non-parametric omnibus comparison across all groups
- **Dunn's post-hoc test** — pairwise comparisons against the WT control group, run only for features significant after FDR correction
- **FDR correction** — Benjamini-Hochberg method applied at both the omnibus level (across features) and the post-hoc level (across pairwise comparisons)

Plots display **median ± IQR/2** (half interquartile range), consistent with the non-parametric approach.

## Data Structure

The pipeline expects AQuA2 output files organized in the following directory structure. Place your data in the `Output__/` directory at the project root.

```
Output__/
├── WT/
│   ├── data1/
│   │   ├── slice1_baseline_AQuA2_Ch1.csv
│   │   ├── slice1_psi_AQuA2_Ch1.csv
│   │   ├── slice1_washout_AQuA2_Ch1.csv
│   │   ├── slice2_baseline_AQuA2_Ch1.csv
│   │   └── ...
│   └── data2/
│       └── ...
├── Antagonist- Volinanserin/
│   ├── data1/
│   │   ├── slice1_baseline_AQuA2_Ch1.csv
│   │   ├── slice1_psi+antag_AQuA2_Ch1.csv
│   │   ├── slice1_washout_AQuA2_Ch1.csv
│   │   └── ...
│   └── data2/
│       └── ...
├── IP3R2 cKO/
│   ├── data1/
│   └── data2/
└── CalEx/
    ├── data1/
    └── data2/
```

### File Naming Convention

Each CSV file follows the pattern:

```
slice<N>_<condition>_AQuA2_<channel>.csv
```

Where:
- `<N>` — slice number (integer)
- `<condition>` — one of: `baseline`, `psi`, or `washout`
- `<channel>` — AQuA2 channel identifier. Configurable via `CHANNEL_SUFFIX` in the notebook; set to `''` if your filenames don't include a channel suffix.

### AQuA2 CSV Format

AQuA2 exports data in **transposed format** (features as rows, events as columns). The pipeline automatically transposes this to standard format (events as rows, features as columns) during loading.

### Generated Files

After running `analysis.ipynb`, normalized CSVs are generated for use by `analysis_time.ipynb`:

```
Output__/<group>/<GROUP>_baseline_normalized.csv
Output__/<group>/<GROUP>_drug_normalized.csv
Output__/<group>/<GROUP>_washout_normalized.csv
```

## Usage

### Step 1: Run the main analysis

Open and run `analysis.ipynb`. This will:
1. Load and preprocess all AQuA2 CSV exports
2. Filter events to frames 20–100 (configurable)
3. Normalize all features relative to baseline
4. Generate timepoint plots (Baseline → Drug → Washout) for each group
5. Run statistical tests across groups
6. Export normalized CSVs for time-series analysis

### Step 2: Run time-series analysis

Open and run `analysis_time.ipynb`. This will:
1. Load the normalized CSVs generated in Step 1
2. Bin events into frame groups (default: 10 frames per bin)
3. Generate multi-panel time-series plots for each group

### Configuration

Key parameters can be adjusted in each notebook:

**analysis.ipynb**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `MIN_FRAME` | 20 | Start frame (excludes early artifacts) |
| `MAX_FRAME` | 100 | End frame (excludes late artifacts) |
| `CHANNEL_SUFFIX` | `'Ch1'` | Channel identifier in filenames (set `''` to omit) |
| `ERROR_TYPE` | `'iqr'` | Error bars: `'iqr'` (half IQR) or `'mad'` |

**analysis_time.ipynb**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `GROUP_SIZE` | 10 | Frames per time bin |
| `MIN_FRAME` | 20 | Start frame |
| `MAX_FRAME` | 100 | End frame |

### Adapting for Your Data

To use this pipeline with your own experimental groups:

1. **Organize your AQuA2 exports** into the directory structure shown above
2. **Edit `DATA_CONFIG`** in `analysis.ipynb` to define your groups:
   ```python
   DATA_CONFIG = {
       'YourGroup': {
           'path': 'folder_name_in_Output__',
           'drug_suffix': 'your_drug_condition',
           'slices': {
               'data1': [1, 2, 3],  # slice numbers in data1/
               'data2': [1, 2],     # slice numbers in data2/
           }
       },
   }
   ```
3. **Update the data loading section** in `analysis_time.ipynb` to match your groups

