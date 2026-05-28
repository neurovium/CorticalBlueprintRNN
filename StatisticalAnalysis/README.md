[![Wiki](https://img.shields.io/badge/-Wiki-blue?style=flat-square&logo=github)](wiki/Home.md)
<!-- [![arXiv](https://img.shields.io/badge/arXiv-preprint-D12424?logo=arxiv)](https://arxiv.org/abs/X) -->

# Statistical analysis pipeline — Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks

This folder contains the post-hoc statistical pipeline for the research paper *"Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks"*. It consumes the `*_raw.csv` outputs of [`../Model/CorticalBlueprintRNN.ipynb`](../Model/CorticalBlueprintRNN.ipynb) and produces the numbers reported in the paper's main text (Table 2) and Supplementary Results (Tables S5–S7).

The pipeline is nonparametric end-to-end (Kruskal–Wallis omnibus → Mann–Whitney U pairwise → Holm correction, plus bootstrap CIs), because the per-run metric distributions across 20 simulations are heavy-tailed and not Gaussian. For the methodology, schema reference, and step-by-step reproduction guide, see the [**wiki**](wiki/Home.md).

## Repository Structure

```
StatisticalAnalysis/
├── README.md                                            # This file
├── 6_6_2_sttc_corr_all_metric.ipynb                     # Distribution diagnostics (justifies KW/MWU)
├── 6_6_2_sttc_corr_single_metric.ipynb                  # 95% confidence intervals (Table 2)
├── 6_6_2_sttc_corr.ipynb                                # Per-metric comparison scratchpad
├── The_Effect_of_w,_d_and_c_variants_Task_1.ipynb       # KW + MWU + Holm on Task 1 (S5–S7)
├── The_Effect_of_w,_d_and_c_variants_Task_2.ipynb       # same for Task 2
├── The_Effect_of_w,_d_and_c_variants_Task_3.ipynb       # same for Task 3
├── The_Effect_of_w_d_and_c_variants .ipynb              # Earlier combined version (reference only)
└── wiki/                                                # Detailed documentation (see below)
```

## Wiki contents

| page | what it covers |
|---|---|
| [Home](wiki/Home.md) | Landing page + table of contents |
| [01 — Overview](wiki/01-Overview.md) | What each notebook does and which paper table/figure it backs |
| [02 — Input data schema](wiki/02-Input-Data-Schema.md) | Folder layout, CSV schema, the 11-variant row order |
| [03 — Distribution diagnostics](wiki/03-Distribution-Diagnostics.md) | `6_6_2_sttc_corr_all_metric.ipynb` — why nonparametric |
| [04 — Confidence intervals](wiki/04-Confidence-Intervals.md) | `6_6_2_sttc_corr_single_metric.ipynb` — Table 2 CIs |
| [05 — Effect of W, D, C variants](wiki/05-Effect-of-W-D-C-Variants.md) | `Task_{1,2,3}` notebooks — Tables S5–S7 |
| [06 — Statistical methods](wiki/06-Statistical-Methods.md) | Reference for KW, MWU, Holm, bootstrap |
| [07 — Reproduction guide](wiki/07-Reproduction-Guide.md) | End-to-end recipe (train → stat) |

## Notebook ↔ paper artifact map

| notebook | role | paper artifact |
|---|---|---|
| `6_6_2_sttc_corr_all_metric.ipynb` | distribution diagnostics across all 8 metrics for the canonical `W*D*C` model | motivation for using nonparametric tests |
| `6_6_2_sttc_corr_single_metric.ipynb` | bootstrap / t-distribution 95% CIs per metric per variant per task | parenthesized CIs in Table 2 (and Table 3) |
| `6_6_2_sttc_corr.ipynb` | per-metric model comparison scratchpad | exploratory; superseded by the Task_{1,2,3} notebooks |
| `The_Effect_of_w,_d_and_c_variants_Task_1.ipynb` | KW + MWU + Holm on Task 1 across W, D, C variants | Supplementary Tables S5–S7 (Task 1 column) |
| `The_Effect_of_w,_d_and_c_variants_Task_2.ipynb` | same for Task 2 | S5–S7 (Task 2 column) |
| `The_Effect_of_w,_d_and_c_variants_Task_3.ipynb` | same for Task 3 | S5–S7 (Task 3 column) |
| `The_Effect_of_w_d_and_c_variants .ipynb` | earlier combined version of the three task notebooks | reference only |

## Input contract

Every notebook in this folder reads CSV files emitted by the training notebook. The contract is:

- Training output folder: `<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/`
- Per task: `Task{1,2,3}/<metric>_raw.csv` where `<metric>` ∈ `{Accuracy, Assortativity, Entropy, Loss, Modularity, SmallWorldness, Validation_Accuracy, Validation_Loss}`
- Each `_raw.csv`: rows = the 11 model variants (canonical order: `W*D*C, WD*C, WDC, WD*, WD, W, W*D*C*, WD*C*, W*DC*, W!D*C, W!D*C*`), columns = the 20 simulation runs.

Full schema in [02 — Input data schema](wiki/02-Input-Data-Schema.md).

## Quick start

1. Run the training pipeline first — see [`../Model/README.md`](../Model/README.md) — to produce a `<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/` output folder.
2. Install the stat dependencies (`pip install scipy numpy pandas matplotlib seaborn statsmodels scikit-learn`).
3. Point the `folder_name` constant at the top of each notebook to your training-output folder (Colab default: `/content/data/simu/<base>/`).
4. Run, in this order:
   1. `6_6_2_sttc_corr_all_metric.ipynb` — verifies the metric distributions are non-Gaussian.
   2. `6_6_2_sttc_corr_single_metric.ipynb` — produces the 95% CIs in Table 2.
   3. `The_Effect_of_w,_d_and_c_variants_Task_{1,2,3}.ipynb` — KW + MWU + Holm per factor (`W`, `D`, `C`), producing Supplementary Tables S5–S7.

Full step-by-step recipe, including the robustness arms (precision matrix, sign constraint, cross-field), is in [07 — Reproduction guide](wiki/07-Reproduction-Guide.md).

## Related

- [Top-level repo README](../README.md) — high-level methods recap + how this folder fits with `Model/`
- [Model README](../Model/README.md) — training notebook that produces the `*_raw.csv` inputs this pipeline consumes
- [Model wiki](../Model/wiki/Home.md) — parameter reference and reproduction guide for the upstream pipeline
