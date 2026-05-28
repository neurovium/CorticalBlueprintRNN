# 02 — Simulation parameters

Every knob the notebook exposes lives in a single cell near the top, labeled *Simulation Parameters*. This page is a reference for what each knob does, the shipped default, and the value the paper used.

## The parameters cell, verbatim

```python
simulations                   = 1
session_scan_field            = '6_6_2'
number_of_nodes               = 312
network_grid                  = (12, 13, 2)
sign_constraint               = False
use_only_sttc                 = False
use_precison_matrix           = False     # (sic — misspelled in code; same in stat notebooks)
TASK1, TASK2, TASK3           = True, True, True
noise_level                   = 0.05
regu_strength                 = 0.3
emd_strength                  = 0.1
activation_function           = 'relu'
random_network_initialization = 'Orthogonal'
```

## Knob-by-knob reference

| parameter | shipped default | paper config | meaning |
|---|---|---|---|
| `simulations` | `1` | `20` | independent training runs per variant. The paper's CIs assume 20 — bump it before any reproduction run |
| `session_scan_field` | `'6_6_2'` | `'6_6_2'` | which MICrONS field to load. Picks the subfolder `DATA/info/<this>/` |
| `number_of_nodes` | `312` | `312` | size of the recurrent layer. **Must equal the field's neuron count** (see [`INF.md`](../INF.md)) |
| `network_grid` | `(12, 13, 2)` | `(12, 13, 2)` | grid shape used to build the `D` control. Product must equal `number_of_nodes` |
| `sign_constraint` | `False` | `False` for Table 2; `True` for Table 3 | force recurrent weights non-negative (the positive-only / excitatory-only arm) |
| `use_only_sttc` | `False` | `False` | if `True`, bio init uses **only** STTC; correlation and precision are skipped. Drops the `-corr` / `-precision` suffix from the output folder |
| `use_precison_matrix` | `False` | `False` for main; `True` for the precision robustness arm | swap the Pearson correlation for the precision matrix when building `W_bio`. **Note the misspelling** (one `i`) — kept identical in the stat notebooks for cross-compatibility |
| `TASK1, TASK2, TASK3` | all `True` | all `True` | toggle the per-task training loops independently |
| `noise_level` | `0.05` | `0.05` | σ for the leading `tf.keras.layers.GaussianNoise` input layer |
| `regu_strength` | `0.3` | `0.3` | $\lambda$ — strength of every spatial / communicability regularizer |
| `emd_strength` | `0.1` | `0.1` | $\lambda_{\text{EMD}}$ — strength of the EMD term in the `C*` variants |
| `activation_function` | `'relu'` | `'relu'` | passed straight to the `SimpleRNN` layer |
| `random_network_initialization` | `'Orthogonal'` | `'Orthogonal'` | initializer used by the non-`W*` / non-`W!` variants |

## Output folder name

Built from the parameters above:

```
<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/
```

Suffix rules:

| condition                                            | suffix added       |
|------------------------------------------------------|--------------------|
| `use_only_sttc = True`                               | (none — base ends `_sttc`) |
| `use_only_sttc = False, use_precison_matrix = False` | `-corr`            |
| `use_only_sttc = False, use_precison_matrix = True`  | `-precision`       |
| `sign_constraint = True`                             | `_sign+` (appended last) |

The same string is what the stat notebooks expect as their `folder_name` constant — see [StatisticalAnalysis 02 — Input data schema](../../StatisticalAnalysis/wiki/02-Input-Data-Schema.md).

## Invariants you should not break

1. **`product(network_grid) == number_of_nodes == size of the loaded matrices`.** A mismatch will either crash on the first matmul or silently misalign the `D` control. Cross-check `number_of_nodes` against [`INF.md`](../INF.md) for the chosen field.
2. **Variable name spelling is load-bearing.** `use_precison_matrix` (one `i`) is misspelled in both this notebook and the four stat notebooks; renaming one without the others will silently divert the stat pipeline to the wrong folder.
3. **`simulations = 20` is what the paper's CIs assume.** Running fewer simulations widens the bootstrap intervals and re-running the stat notebooks against a `simulations < 20` output will not reproduce Table 2.

## Picking a different field

To run on a different `(session, scan, field)`:

1. Pick a row from [`INF.md`](../INF.md) — note its `Neurons` and `Grid` columns.
2. In this cell:
   - set `session_scan_field` to the underscore-joined string (e.g. `'5_3_4'`),
   - set `number_of_nodes` to the `Neurons` value,
   - set `network_grid` to a factorization (the `Grid` column gives one; any factorization with the right product works).
3. Run the notebook. The output folder name will reflect the new prefix automatically.
