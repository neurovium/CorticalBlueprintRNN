# 02 — Input data schema

This page is the data contract between the training notebook and the statistical pipeline. If you change either side, change this page too.

## Source of truth

Files in this folder are produced by [`CorticalBlueprintRNN.ipynb`](../../Model/CorticalBlueprintRNN.ipynb). For the preprocessed MICrONS *inputs* that the training notebook consumes (one folder per imaged field), see [`Model/DATA/info/`](../../Model/DATA/info/) and the field index in [`INF.md`](../../Model/INF.md).

## Top-level output folder

The training notebook writes one folder per run, named from the simulation parameters:

```
<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/
```

Suffix rules (matching the constants at the top of every notebook in this folder):

| condition                                         | suffix added |
|---------------------------------------------------|--------------|
| `use_only_sttc = True`                            | (no second component — base name ends `_sttc`) |
| `use_only_sttc = False, use_precison_matrix = False` | `-corr` |
| `use_only_sttc = False, use_precison_matrix = True`  | `-precision` |
| `sign_constraint = True`                          | `_sign+` (appended last) |

Examples:

| parameters | folder name |
|---|---|
| 6,6,2 — defaults | `6_6_2_sttc-corr` |
| 6,6,2 — `sign_constraint=True` | `6_6_2_sttc-corr_sign+` |
| 6,6,2 — `use_precison_matrix=True` | `6_6_2_sttc-precision` |
| 6,6,2 — `use_precison_matrix=True, sign_constraint=True` | `6_6_2_sttc-precision_sign+` |

> **Note** — the notebook variable is spelled `use_precison_matrix` (one `i`). The folder suffix is spelled correctly: `-precision`.

In Colab, the stat notebooks expect this folder at `/content/data/simu/<base>/` (i.e. unzipped from `simu.zip` mounted from Google Drive).

## Folder contents

```
6_6_2_sttc-corr/
├── config.txt          ← echo of all simulation parameters
├── metrics.csv         ← mean across the 20 simulations per (task, metric, variant)
├── metrics_SD.csv      ← standard deviation across simulations
├── Task1/
│   ├── Accuracy_raw.csv
│   ├── Assortativity_raw.csv
│   ├── Entropy_raw.csv
│   ├── Loss_raw.csv
│   ├── Modularity_raw.csv
│   ├── SmallWorldness_raw.csv
│   ├── Validation_Accuracy_raw.csv
│   └── Validation_Loss_raw.csv
├── Task2/              ← same 8 files
└── Task3/              ← same 8 files
```

## `<metric>_raw.csv` schema

Each per-task raw CSV is a `(11 × 20)` table:

- **Rows (11):** model variants in canonical order — `W*D*C, WD*C, WDC, WD*, WD, W, W*D*C*, WD*C*, W*DC*, W!D*C, W!D*C*`. This is the order matching Table 1 of the paper (and the in-notebook regularizer key order).
- **Columns (20):** one column per independent simulation run.

The eight metric names exactly as used in the notebooks (the list lives in `6_6_2_sttc_corr_all_metric.ipynb`):

```python
metrics = ['Accuracy', 'Assortativity', 'Entropy', 'Loss',
           'Modularity', 'SmallWorldness',
           'Validation_Accuracy', 'Validation_Loss']
```

## `metrics.csv` and `metrics_SD.csv`

These are the aggregated tables, written once per run:

- **Rows:** `(task, metric)` pairs (24 rows = 3 tasks × 8 metrics).
- **Columns:** the 11 model variants.
- **Cells:** mean (or SD) across the 20 simulations.

These are convenient for spot-checking; the statistical pipeline reads the per-run `*_raw.csv` files instead, because the bootstrap and KW tests need the full distribution.

## Cross-reference: training inputs

The training notebook in turn consumes per-field data under `Model/DATA/info/<sess>_<scan>_<field>/`. For the canonical field `6_6_2` the directory contains (verified by `ls`):

| file | content |
|---|---|
| `connectome_6_6_2.csv` | binary/weighted structural adjacency from EM |
| `sttc_matrix.npy` | pairwise STTC over the binned spike trains |
| `corr_matrix.npy` | pairwise Pearson correlation |
| `precision_matrix.npy` | inverse covariance (precision matrix) |
| `positions_6_6_2.npy` | 3-D soma coordinates (used to build `D*`) |
| `spikes_6_6_2.npy` | deconvolved spike trains used to derive STTC/corr/precision |
| `root_ids_6_6_2.npy` | proofread segmentation IDs |
| `unit_ids_6_6_2.npy` | functional ROI IDs (unique per scan) |
| `target_ids_6_6_2.npy` | postsynaptic targets for the connectome graph |

The list of all available `(session, scan, field)` combinations and their neuron counts is in [`INF.md`](../../Model/INF.md).
