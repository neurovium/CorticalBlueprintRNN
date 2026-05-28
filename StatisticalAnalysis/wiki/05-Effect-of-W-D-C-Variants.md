# 05 — Effect of W, D, C variants

Notebooks: `The_Effect_of_w,_d_and_c_variants_Task_1.ipynb`, `_Task_2.ipynb`, `_Task_3.ipynb`
(plus the earlier combined `The_Effect_of_w_d_and_c_variants .ipynb`)

## Purpose

For each of the three cognitive tasks, decompose the eleven-variant comparison into three orthogonal factor-level tests so the paper can make claims like *"functional initialization has the largest effect; spatial embedding adds a robust secondary effect; communicability is more selective."*

These notebooks produce **Supplementary Tables S5–S7** — one supplementary table per factor:

| supplementary table | factor                                       |
|---|---|
| **S5** | Effect of $W$ — `W` vs `W*` vs `W!`, holding $D, C$ fixed |
| **S6** | Effect of $D$ — `D` vs `D*`, holding $W, C$ fixed |
| **S7** | Effect of $C$ — none vs `C` vs `C*`, holding $W, D$ fixed |

Each notebook is one task. The combined ` The_Effect_of_w_d_and_c_variants .ipynb` is an earlier version that runs the same analysis across all three tasks in one place; it is kept for cross-checking but is superseded by the task-specific notebooks.

## Inputs

```python
session, scan, field = 6, 6, 2
sign_constraint      = False
use_only_sttc        = False
use_precison_matrix  = False
```

Reads every metric CSV under `Task{N}/`:

```
/content/data/simu/{base}/Task{N}/<metric>_raw.csv
```

for `N ∈ {1, 2, 3}` (one task per notebook) and every metric in
`{Accuracy, Loss, Entropy, Modularity, Assortativity, SmallWorldness, Validation_Accuracy, Validation_Loss}`.

## Pipeline per notebook

For each metric:

1. **Slice by factor.** Given the 11-variant table, hold two factors fixed at one of their allowed combinations and gather the columns for the levels of the remaining factor.
   - Example: "Effect of $W$ at $(D = D^{*}, C = C)$" → take rows `W*D*C`, `WD*C`, `W!D*C` and treat them as three groups of 20 observations each.
2. **Kruskal–Wallis omnibus** (`scipy.stats.kruskal`) across the groups → $H$, $p$.
3. **Mann–Whitney U pairwise** (`scipy.stats.mannwhitneyu`) between every pair of levels → $U$, raw $p$.
4. **Holm step-down correction** (`statsmodels.stats.multitest.multipletests(..., method='holm')`) across the family of pairwise tests for that factor / fixed-context cell.
5. Optionally rank-standardize the inputs using `MinMaxScaler` / `StandardScaler` before re-running.

`itertools.combinations` is used to enumerate the pairs, and the corrected $p$-values are pasted into S5/S6/S7.

## What the paper says these tests show

From the Results section of `main.tex`:

- **Effect of $W$ (S5).** With $D^{*}$ and $C$ held fixed, both `W*` and `W!` significantly outperform `W` on every task, **but `W*` vs `W!` is not significant** — i.e. distribution structure, not the exact neuron-to-weight mapping, drives the gain. This is the strongest single factor.
- **Effect of $D$ (S6).** Replacing `D` with `D*` produces a significant performance gain across tasks, most clearly visible in the non-bio-init variants.
- **Effect of $C$ (S7).** Smaller and more task-specific. Direct `C` generally beats `C*` on Tasks 2 and 3 when paired with `D*`; on Task 1 the differences are small.

## Robustness arms inside these notebooks

The same notebooks also run two robustness checks referenced in §III.C of the paper:

- **Correlation vs. precision matrix** as the bio-init prior. Re-running with `use_precison_matrix = True` (and reading from `6_6_2_sttc-precision/`) preserves the qualitative ranking from S5 — supports the claim that both direct and indirect functional dependencies are informative.
- **Cross-field consistency.** Pointing `(session, scan, field)` at other entries from [`INF.md`](../../Model/INF.md) preserves the variant ranking despite varying neuron counts — supports the claim that the cortical priors generalize across sampled fields.

For step-by-step replication of all of these, see [07 — Reproduction guide](07-Reproduction-Guide.md).
