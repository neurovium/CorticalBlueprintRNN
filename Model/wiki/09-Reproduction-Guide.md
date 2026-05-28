# 09 — Reproduction guide

End-to-end recipe for running the training pipeline that produces the inputs to the statistical analysis. The downstream half of the recipe (turning these CSVs into Table 2, Table 3, S5–S7) lives at [StatisticalAnalysis 07 — Reproduction guide](../../StatisticalAnalysis/wiki/07-Reproduction-Guide.md).

## Prerequisites

- Python ≥ 3.10. (Colab works directly — open the notebook through the Colab badge in [`README.md`](../README.md).)
- `pip install bctpy tensorflow scipy matplotlib numpy pandas seaborn tqdm networkx`
- Disk: under 100 MB for a single field's output folder.
- Compute: the canonical run (`simulations = 20`, 11 variants, 3 tasks, 10 epochs, 312-neuron field) finishes in well under an hour on a single modern GPU; multiple hours on CPU.

## Step 1 — Run the data download cell (once)

The data-download cell at the top of the notebook fetches `DATA.zip` and `INF.md` from the public GitHub mirror and unzips `DATA/info/<sess>_<scan>_<field>/` folders into the working directory. If you already have `DATA/` populated (as is the case in this repo checkout), this cell is a no-op.

## Step 2 — Set the parameters cell to the paper config

Override the shipped defaults to match the paper:

```python
simulations         = 20                # the shipped default is 1 (fast iteration only)
session_scan_field  = '6_6_2'
number_of_nodes     = 312
network_grid        = (12, 13, 2)
sign_constraint     = False             # set True for the Table 3 positive-only arm
use_only_sttc       = False
use_precison_matrix = False             # set True for the precision-matrix robustness arm
TASK1 = TASK2 = TASK3 = True
noise_level         = 0.05
regu_strength       = 0.3
emd_strength        = 0.1
activation_function = 'relu'
random_network_initialization = 'Orthogonal'
```

Cross-check `number_of_nodes` and `network_grid` against [`INF.md`](../INF.md) for the chosen field.

## Step 3 — Run all cells

Top to bottom. The training cells dominate runtime; the rest is essentially instantaneous.

When the Save cell finishes you'll have:

```
6_6_2_sttc-corr/
├── config.txt
├── metrics.csv      metrics_SD.csv
└── Task{1,2,3}/{Accuracy,Loss,Validation_Accuracy,Validation_Loss,
                 Modularity,SmallWorldness,Assortativity,Entropy}_raw.csv
```

These are the eleven-variant CSVs the stat notebooks read.

## Step 4 — Hand the output to the statistical pipeline

The stat notebooks expect to find the folder at `/content/data/simu/<base>/`. Options (all of these work):

- **Colab mirror**: zip `6_6_2_sttc-corr/` as `simu.zip`, drop it in `MyDrive/ISP/`, let the stat notebooks' download cells handle the rest.
- **Local edit**: in each stat notebook, override `folder_name = '<absolute-path-to-your-output>'`.
- **Symlink**: `ln -s /your/path/6_6_2_sttc-corr /content/data/simu/6_6_2_sttc-corr` (or the PowerShell `New-Item -ItemType SymbolicLink` equivalent on Windows).

Then follow [StatisticalAnalysis 07 — Reproduction guide](../../StatisticalAnalysis/wiki/07-Reproduction-Guide.md) to regenerate Tables 2, 3, S5, S6, S7.

## Robustness arms

Each arm is **rerun Step 2 with the indicated parameter change, then Step 3, then Step 4**:

| paper arm | parameter change | output folder |
|---|---|---|
| Table 3 — positive-only recurrence | `sign_constraint = True` | `6_6_2_sttc-corr_sign+/` |
| §III.C — precision matrix as bio init | `use_precison_matrix = True` (sic — one `i`) | `6_6_2_sttc-precision/` |
| §III.C — STTC-only bio init | `use_only_sttc = True` | `6_6_2_sttc/` (no `-corr`/`-precision` suffix) |
| §III.C — cross-field consistency | `session_scan_field`, `number_of_nodes`, `network_grid` to another row of [`INF.md`](../INF.md) | e.g. `5_3_4_sttc-corr/` |

## Quick sanity checks before trusting a run

1. **`config.txt` matches what you intended.** Open it and verify the parameters are what you set in Step 2.
2. **`metrics.csv` has 24 rows × 11 columns.** 24 = 3 tasks × 8 metrics; missing rows mean a `TASKn` was off.
3. **Top-row accuracies on `metrics.csv` are sane.** For the canonical config, `W*D*C` accuracy on Task 1 should land near 0.92 (cf. Table 2). If everything's at chance, the bio init likely failed to load — re-check Step 1.
4. **Folder name matches the suffix rules.** See [02 — Simulation parameters](02-Simulation-Parameters.md) for the table. A mismatch usually means a stale parameter that didn't actually take effect.
