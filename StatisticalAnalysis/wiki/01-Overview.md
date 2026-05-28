# 01 — Overview

## Why a nonparametric pipeline?

Each run of `CorticalBlueprintRNN.ipynb` produces 20 independent simulations per variant per task. The per-metric distributions of these 20 values are typically:

- **bimodal or multimodal** for accuracy (runs that found a good basin vs. runs that collapsed near chance),
- **heavy-tailed** for loss, modularity and small-worldness, and
- **bounded** (e.g. assortativity ∈ [−1, 1], accuracy ∈ [0, 1]).

A Gaussian assumption is violated for most metric × variant cells, so the paper uses a fully nonparametric pipeline:

1. **Kruskal–Wallis** omnibus across the levels of a factor ($W$, $D$, or $C$) holding the other two factors fixed.
2. If KW is significant, **Mann–Whitney U** pairwise post-hoc between every pair of levels.
3. **Holm step-down** correction across the family of pairwise tests within a factor.
4. **Bootstrap** (and t-distribution) 95% confidence intervals on the per-variant per-metric distributions for the reported numbers in Table 2.

The diagnostics that justify this choice are produced by `6_6_2_sttc_corr_all_metric.ipynb` (see [03 — Distribution diagnostics](03-Distribution-Diagnostics.md)).

## Notebooks at a glance

| notebook | input | what it computes | paper artifact |
|---|---|---|---|
| `6_6_2_sttc_corr_all_metric.ipynb` | all 8 `*_raw.csv` files for one variant (default `W*D*C`) across all 3 tasks | per-metric distribution plots, mutual-information ranking between metrics, Spearman correlation heatmap, KW omnibus, MWU pairwise | motivates nonparametric choice |
| `6_6_2_sttc_corr_single_metric.ipynb` | one `<metric>_raw.csv` per task across all 11 variants | 95% CIs per variant per task (t-distribution + bootstrap) | parenthesized CIs in Table 2 (& Table 3) |
| `6_6_2_sttc_corr.ipynb` | one `<metric>_raw.csv` per task | boxplots + KW + MWU on one metric across variants | scratchpad; superseded by Task notebooks |
| `The_Effect_of_w,_d_and_c_variants_Task_1.ipynb` | all 8 `*_raw.csv` files for Task 1 | KW + MWU + Holm per factor ($W$, $D$, $C$); robustness arms | Supplementary Tables S5/S6/S7, Task 1 |
| `The_Effect_of_w,_d_and_c_variants_Task_2.ipynb` | Task 2 equivalent | same | S5/S6/S7, Task 2 |
| `The_Effect_of_w,_d_and_c_variants_Task_3.ipynb` | Task 3 equivalent | same | S5/S6/S7, Task 3 |
| `The_Effect_of_w_d_and_c_variants .ipynb` | combined Task 1–3 | early version, mostly subsumed | reference only |

## Factor definitions

The "Effect of …" notebooks slice the 11 variants by three orthogonal factors:

| factor | levels | meaning |
|---|---|---|
| $W$ | `W`, `W*`, `W!` | standard / bio / bio-permuted weight init |
| $D$ | `D`, `D*` | grid coordinates / real MICrONS coordinates |
| $C$ | none, `C`, `C*` | no communicability / direct $C$ / EMD-based $C^{*}$ |

"Effect of $W$ for fixed $D, C$" means: hold $D$ and $C$ at one of their allowed combinations, vary $W$ across its levels, and run KW + MWU + Holm. The paper's hierarchy (functional init > spatial > communicability) drops out of these tests.

## Where each notebook expects its input

All notebooks share the same path constants at the top:

```python
session, scan, field = 6, 6, 2
sign_constraint      = False
use_only_sttc        = False
use_precison_matrix  = False        # note: spelled `precison` in the code

base = f'{session}_{scan}_{field}_sttc'
if not use_only_sttc:
    base += '-precision' if use_precison_matrix else '-corr'
if sign_constraint:
    base += '_sign+'

folder_name = f'/content/data/simu/{base}'
```

`folder_name` is the directory the training notebook wrote out. The "Load Dataset" cells immediately afterwards `drive.mount('/content/drive')` and `!unzip /content/drive/MyDrive/ISP/simu.zip -d /content/data`. To run locally, replace those two cells with a direct path to your training output (or pre-unzip to `/content/data/simu/`).

See [07 — Reproduction guide](07-Reproduction-Guide.md) for the full end-to-end recipe.
