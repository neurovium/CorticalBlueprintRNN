# StatisticalAnalysis — Wiki

This wiki documents the post-hoc statistical pipeline that consumes the outputs of [`CorticalBlueprintRNN.ipynb`](../../Model/CorticalBlueprintRNN.ipynb) and produces the numbers reported in the paper's main text (Table 2) and Supplementary Results (Tables S5–S7).

The pipeline is nonparametric end-to-end (Kruskal–Wallis omnibus → Mann–Whitney U pairwise → Holm correction, plus bootstrap CIs), because the per-run metric distributions across 20 simulations are heavy-tailed and not Gaussian. The justification for this choice lives in [`03-Distribution-Diagnostics.md`](03-Distribution-Diagnostics.md).

## Contents

| page | what it covers |
|---|---|
| [01-Overview](01-Overview.md) | What each notebook does and which paper table/figure it backs |
| [02-Input-Data-Schema](02-Input-Data-Schema.md) | Folder layout, CSV schema, the 11-variant row order |
| [03-Distribution-Diagnostics](03-Distribution-Diagnostics.md) | `6_6_2_sttc_corr_all_metric.ipynb` — why nonparametric |
| [04-Confidence-Intervals](04-Confidence-Intervals.md) | `6_6_2_sttc_corr_single_metric.ipynb` — Table 2 CIs |
| [05-Effect-of-W-D-C-Variants](05-Effect-of-W-D-C-Variants.md) | `Task_{1,2,3}` notebooks — Tables S5–S7 |
| [06-Statistical-Methods](06-Statistical-Methods.md) | Reference for KW, MWU, Holm, bootstrap |
| [07-Reproduction-Guide](07-Reproduction-Guide.md) | Full step-by-step recipe (train → stat) |

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
- Each `_raw.csv`: rows = the 11 model variants, columns = the 20 simulation runs.

Full schema in [`02-Input-Data-Schema.md`](02-Input-Data-Schema.md).
