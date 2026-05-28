# 04 — Confidence intervals

Notebook: `6_6_2_sttc_corr_single_metric.ipynb`

## Purpose

Produce the per-variant per-task 95% confidence intervals reported parenthetically in **Table 2** of the paper (e.g. `0.917 (0.853, 0.981)`) and in the positive-only counterpart **Table 3**.

The notebook is parameterized by one metric at a time (default `metric = 'Modularity'`) and runs across all three tasks and all eleven variants.

## Inputs

```python
session, scan, field = 6, 6, 2
sign_constraint      = True       # ← note: set to True in the shipped notebook
use_only_sttc        = False
use_precison_matrix  = False
metric               = 'Modularity'     # set to 'Accuracy' to reproduce Table 2 / Table 3
```

Reads three files (one per task):

```python
task1_file_dir = f'{folder_name}/Task1/{metric}_raw.csv'
task2_file_dir = f'{folder_name}/Task2/{metric}_raw.csv'
task3_file_dir = f'{folder_name}/Task3/{metric}_raw.csv'
```

with `folder_name = f'/content/data/simu/{base}'` (see [02 — Input data schema](02-Input-Data-Schema.md) for how `base` is constructed).

> To reproduce the **Table 2 accuracies**, set `sign_constraint = False` and `metric = 'Accuracy'`; the folder becomes `6_6_2_sttc-corr/`.
> To reproduce the **Table 3 (positive-only) accuracies**, keep `sign_constraint = True` and `metric = 'Accuracy'`; the folder becomes `6_6_2_sttc-corr_sign+/`.

## What the notebook computes

For each `(variant, task)` cell — i.e. the 20-element row of the raw CSV — it computes:

- **Point estimate:** the mean across the 20 simulations.
- **t-distribution 95% CI:** $\bar{x} \pm t_{0.975, n-1}\, \hat{\sigma}/\sqrt{n}$ with $n = 20$.
- **Percentile bootstrap 95% CI:** resample 20 values with replacement many times, take the 2.5th and 97.5th percentiles of the resulting means.

The paper reports the bootstrap CIs (more robust given the non-Gaussian per-run distributions established in [03 — Distribution diagnostics](03-Distribution-Diagnostics.md)). Both intervals are typically close on accuracy, but can diverge on heavy-tailed metrics (loss, modularity).

## Outputs

- A table per task laying out, for each of the 11 variants:
  `mean | t-CI lower | t-CI upper | bootstrap lower | bootstrap upper`.
- Distribution-overlay plots per variant.

These tables are what get pasted into the paper. The notebook does not write a CSV — copy the numbers manually, or add a `df.to_csv(...)` if you want a machine-readable artifact.

## Mapping to paper tables

| paper artifact                          | notebook settings                                                                                         |
|----------------------------------------|----------------------------------------------------------------------------------------------------------|
| Table 2 (CIs on accuracy)              | `sign_constraint=False`, `metric='Accuracy'` → folder `6_6_2_sttc-corr/`                                  |
| Table 3 (CIs under positive recurrence) | `sign_constraint=True`, `metric='Accuracy'` → folder `6_6_2_sttc-corr_sign+/`                             |
| Per-metric CIs cited in the Results / Supplement | iterate over `metric ∈ {Loss, Entropy, Modularity, SmallWorldness, Assortativity, Validation_*}` |
