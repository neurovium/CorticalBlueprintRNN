# 03 — Distribution diagnostics

Notebook: `6_6_2_sttc_corr_all_metric.ipynb`

## Purpose

Establish — for the canonical model and field — that the per-run distributions of the eight tracked metrics are *not* well approximated by Gaussians. This is the empirical justification for using Kruskal–Wallis / Mann–Whitney U / Holm throughout the rest of the pipeline rather than ANOVA / t-tests.

The notebook is run once for the chosen reference model (default `model = 'W*D*C'` at the top) and once per task. Its outputs are diagnostic plots and tables that motivate the test choice in `01-Overview.md`; they are not numbered tables in the paper, but they back the methodological choice stated in §IV.E ("StatisticalAnalysis").

## Inputs

```python
session, scan, field = 6, 6, 2
sign_constraint      = False
use_only_sttc        = False
use_precison_matrix  = False
model                = 'W*D*C'
```

Reads all 8 `Task{1,2,3}/<metric>_raw.csv` files under
`/content/data/simu/<base>/` (`base = '6_6_2_sttc-corr'` for these defaults).

The metric list used internally:

```python
metrics = ['Accuracy', 'Assortativity', 'Entropy', 'Loss',
           'Modularity', 'SmallWorldness',
           'Validation_Accuracy', 'Validation_Loss']
```

## What the notebook does

1. **Per-metric distribution plots.** A helper `draw_olumn_disturbutions(df)` (sic — name preserved verbatim) renders a histogram for each variant's column. Visually confirms multi-modality / heavy-tailedness for most metrics.
2. **Non-linear dependence between metrics.** `captures_non_linear_dependencies(df, metric)` regresses every other metric on the chosen target using `sklearn.feature_selection.mutual_info_regression`, ranking which metrics carry redundant information about the target.
3. **Spearman rank correlations.** Pairwise Spearman ρ across the 8 metrics, rendered as a heatmap. Confirms that ordinal relationships, not linear ones, dominate — another reason to prefer rank-based tests.
4. **Kruskal–Wallis omnibus** across variant columns for each metric — a sanity check that variants differ in distribution on every metric.
5. **Mann–Whitney U pairwise comparisons** to scope the strength and direction of the per-variant differences.

## Outputs

In-notebook only:

- One histogram per metric per variant (24 panels per task).
- A mutual-information ranking table per metric.
- One Spearman heatmap per task.
- Inline KW H-statistics and MWU U-statistics.

There are no CSV exports from this notebook; its purpose is methodological justification.

## Methodological role in the paper

This notebook is what backs the statement in `main.tex` §IV.E that the pipeline uses **"a rank-based factorial ANOVA … followed by Kruskal–Wallis tests, then pairwise Mann–Whitney post hoc comparisons with Holm correction."** Without these diagnostics, the choice to forgo parametric tests would be unmotivated.

## Reading the output

If you re-run this notebook on a new training output and any of the eight metrics looks roughly Gaussian (single mode, light tails, symmetric), the nonparametric assumption is unnecessarily conservative for that metric — flag it before pushing the results downstream. In practice every metric we have inspected on this dataset fails the visual normality check, which is why the rest of the pipeline is uniformly nonparametric.
