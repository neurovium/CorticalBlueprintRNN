# 07 — Reproduction guide

End-to-end recipe for re-producing every paper number that comes from this folder, starting from a fresh checkout.

## Prerequisites

- Python ≥ 3.10 (Colab works directly).
- `pip install bctpy tensorflow scipy matplotlib numpy pandas seaborn tqdm networkx scikit-learn statsmodels`
- A GPU is not required (the field is small — 312 neurons by default — and 10 epochs × 20 sims × 11 variants × 3 tasks finishes in a few hours on CPU).

## Step 1 — Train the canonical run

Open [`Model/CorticalBlueprintRNN.ipynb`](../../Model/CorticalBlueprintRNN.ipynb) and set the parameter cell to the paper defaults:

```python
session, scan, field = 6, 6, 2
number_of_nodes      = 312
network_grid         = (12, 13, 2)
simulations          = 20
regu_strength        = 0.3      # λ
emd_strength         = 0.1      # λ_EMD
noise_level          = 0.05
sign_constraint      = False
use_only_sttc        = False
use_precision_matrix = False
TASK1 = TASK2 = TASK3 = True
```

Run all cells. The notebook writes:

```
6_6_2_sttc-corr/
├── config.txt, metrics.csv, metrics_SD.csv
└── Task{1,2,3}/<metric>_raw.csv
```

(see [02 — Input data schema](02-Input-Data-Schema.md) for the full layout).

## Step 2 — Make the output visible to the stat notebooks

The stat notebooks expect the output under `/content/data/simu/<base>/`. Three options:

- **Colab path (what the shipped notebooks do).** Zip `6_6_2_sttc-corr/` as `simu.zip`, drop it into Google Drive at `MyDrive/ISP/simu.zip`, and let the notebooks' first cells `drive.mount(...)` + `!unzip ... -d /content/data` do the rest.
- **Local path with minimal edits.** After unzipping locally to e.g. `~/runs/simu/6_6_2_sttc-corr/`, edit just the `folder_name` line at the top of each stat notebook:
  ```python
  folder_name = '/Users/you/runs/simu/6_6_2_sttc-corr'
  ```
- **Symlink.** `mkdir -p /content/data/simu && ln -s /your/path/6_6_2_sttc-corr /content/data/simu/` (POSIX), or the PowerShell `New-Item -ItemType SymbolicLink` equivalent (Windows).

## Step 3 — Run the statistical pipeline

In this order:

1. **Distribution diagnostics** — `6_6_2_sttc_corr_all_metric.ipynb`. No exported numbers; confirms the metric distributions are non-Gaussian and therefore that KW/MWU/Holm are appropriate. Skim the histograms before trusting the rest.
2. **Confidence intervals** — `6_6_2_sttc_corr_single_metric.ipynb`. Set `metric = 'Accuracy'` and `sign_constraint = False` to regenerate **Table 2**. Repeat with other `metric` values for the per-metric CIs cited elsewhere.
3. **Effect of W, D, C per task** — `The_Effect_of_w,_d_and_c_variants_Task_1.ipynb`, `_Task_2.ipynb`, `_Task_3.ipynb`. These produce the omnibus + pairwise Holm-corrected $p$-values that fill **Supplementary Tables S5 (W), S6 (D), S7 (C)** — one column per task.

## Step 4 — Robustness arms

### 4a. Positive-only recurrent weights (Table 3)

Rerun **Step 1** with:
```python
sign_constraint = True
```
→ produces `6_6_2_sttc-corr_sign+/`. Re-run `6_6_2_sttc_corr_single_metric.ipynb` with `sign_constraint = True` and `metric = 'Accuracy'`.

### 4b. Precision matrix as bio-init prior

Rerun **Step 1** with:
```python
use_precision_matrix = True   # the notebook variable; the stat notebooks read it as `use_precison_matrix`
```
→ produces `6_6_2_sttc-precision/`. Then in every stat notebook set `use_precison_matrix = True` (note the spelling — `precison`, one `i`) and re-run. The qualitative ranking in S5 should be preserved.

### 4c. Cross-field consistency

Pick a different `(session, scan, field)` from [`INF.md`](../../Model/INF.md). Update `number_of_nodes` to match the field's neuron count and `network_grid` to a factorization of that count. Rerun Step 1; the output folder name changes accordingly (e.g. `5_3_4_sttc-corr/`). Update the same `(session, scan, field)` triple at the top of each stat notebook and re-run. Compare against the canonical `6_6_2` results.

## Step 5 — Cross-check against the paper

| paper artifact | how to verify |
|---|---|
| Table 2 (CIs) | Numbers from Step 3.2 should match `0.917 (0.853, 0.981)`-style entries within bootstrap noise |
| Table 3 (positive-only CIs) | Numbers from Step 4a should match its entries |
| S5–S7 (W/D/C effects) | Numbers from Step 3.3 should match the corrected $p$-values cited in `main.tex` lines 111 & 241 |

If any of these diverge by more than a few significant digits, the most common culprits are: forgot to re-run the training step after toggling `sign_constraint` / `use_precision_matrix`; loaded the wrong `folder_name`; or used a different number of bootstrap resamples than the original run.

## Common pitfalls

- **Variable-name spelling.** Training notebook uses `use_precision_matrix`; stat notebooks use `use_precison_matrix` (one `i`). The folder-suffix string is spelled correctly (`-precision`) — only the in-notebook variable differs.
- **`number_of_nodes` must equal the field's neuron count.** Otherwise the connectome / functional matrices and the RNN's recurrent matrix won't line up. Cross-check against `INF.md`.
- **`network_grid` shape must factorize to `number_of_nodes`.** E.g. `12 × 13 × 2 = 312`. Mismatched shapes silently misalign the `D` (artificial-grid) control.
- **20 simulations is a hard requirement for the CIs to match the paper.** Fewer simulations widen the bootstrap intervals; more narrow them. Don't compare CIs from different `simulations` counts.
