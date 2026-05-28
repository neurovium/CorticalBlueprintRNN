# 07 — Model variants

Notebook section: **Model Training**

The eleven variants are defined as a dictionary whose insertion order is the canonical order used throughout the paper, the saved CSVs, and the statistical pipeline. Python 3.7+ preserves dict order, so iterating `regularizers.items()` walks the variants deterministically.

```python
regularizers = {
    'W*D*C' : regu1,    # bio init + real coords + direct C
    'WD*C'  : regu2,    # random init + real coords + direct C
    'WDC'   : regu3,    # random init + grid + direct C
    'WD*'   : regu4,    # random init + real coords (no C)
    'WD'    : regu5,    # random init + grid (no C)
    'W'     : regu6,    # random init, plain L1 (Simple RNN baseline)
    'W*D*C*': regu7,    # bio init + real coords + EMD C
    'WD*C*' : regu8,    # random init + real coords + EMD C
    'W*DC*' : regu9,    # bio init + grid + EMD C
    'W!D*C' : regu10,   # permuted bio init + real coords + direct C
    'W!D*C*': regu11,   # permuted bio init + real coords + EMD C
}
```

## The naming convention

A variant name decomposes into three independent factor levels:

| segment | levels | meaning |
|---|---|---|
| `W` slot | `W` / `W*` / `W!` | random Orthogonal / biologically initialized / bio init with weights permuted |
| `D` slot | `D` / `D*` | grid coordinates / real MICrONS coordinates |
| `C` slot | (absent) / `C` / `C*` | no communicability regularization / direct communicability / EMD-on-communicability |

Star (`*`) = empirical; bang (`!`) = permutation control on `W*`.

The eleven variants are not the full Cartesian product of the three factor sets — only the combinations that test interesting hypotheses are included. For example, there is no `WD*C` paired with EMD because the `C*` arm exists explicitly to compare against direct `C`.

## Full variant table

(reproduces Table 1 of the paper, in dict / CSV row order)

| # | name | bio init | spatial | comm. | regularizer |
|---|---|---|---|---|---|
| 1 | **W\*D\*C**   | yes (`W*`)  | real (`D*`) | direct (`C`)  | $\lambda \, \lVert W^{*} \odot D^{*} \odot C \rVert$ |
| 2 | WD\*C     | no          | real (`D*`) | direct (`C`)  | $\lambda \, \lVert W \odot D^{*} \odot C \rVert$ |
| 3 | WDC       | no          | grid (`D`)  | direct (`C`)  | $\lambda \, \lVert W \odot D \odot C \rVert$ |
| 4 | WD\*      | no          | real (`D*`) | none          | $\lambda \, \lVert W \odot D^{*} \rVert$ |
| 5 | WD        | no          | grid (`D`)  | none          | $\lambda \, \lVert W \odot D \rVert$ |
| 6 | W (Simple) | no         | none        | none          | $\lambda \, \lVert W \rVert$ |
| 7 | **W\*D\*C\*** | yes (`W*`) | real (`D*`) | EMD (`C*`)   | $\lambda \, \lVert W^{*} \odot D^{*} \rVert + \lambda_{\text{EMD}} \cdot \text{EMD}(C_{\text{emp}}, C_{\text{art}})$ |
| 8 | WD\*C\*   | no          | real (`D*`) | EMD (`C*`)    | $\lambda \, \lVert W \odot D^{*} \rVert + \lambda_{\text{EMD}} \cdot \text{EMD}(C_{\text{emp}}, C_{\text{art}})$ |
| 9 | W\*DC\*   | yes (`W*`)  | grid (`D`)  | EMD (`C*`)    | $\lambda \, \lVert W^{*} \odot D \rVert + \lambda_{\text{EMD}} \cdot \text{EMD}(C_{\text{emp}}, C_{\text{art}})$ |
| 10 | W!D\*C   | yes (`W!`)  | real (`D*`) | direct (`C`)  | $\lambda \, \lVert W^{!} \odot D^{*} \odot C \rVert$ |
| 11 | W!D\*C\* | yes (`W!`)  | real (`D*`) | EMD (`C*`)    | $\lambda \, \lVert W^{!} \odot D^{*} \rVert + \lambda_{\text{EMD}} \cdot \text{EMD}(C_{\text{emp}}, C_{\text{art}})$ |

## How a variant is built and trained

For each variant the per-task training loop:

1. Looks at the variant name to decide the initializer:
   - contains `W*` → use `ConnectomeInitializer(W_bio)` (precomputed in section 6).
   - contains `W!` → use `ConnectomeInitializer(W_bio_permuted)` where the entries are scrambled fresh per simulation.
   - otherwise → use `random_network_initialization` (Orthogonal by default).
2. Picks the regularizer instance from the dict above (`regu1`…`regu11`), which already has the correct `D` (real or grid) and, if applicable, `C_emp` baked in.
3. Builds the model: `GaussianNoise(σ=noise_level)` → `SimpleRNN(units=number_of_nodes, activation=activation_function, recurrent_initializer=…, recurrent_regularizer=…)` → `Dense(num_classes, activation='softmax')`. If `sign_constraint = True`, a non-negativity constraint is attached to the recurrent kernel.
4. Compiles with Adam + categorical cross-entropy and fits for 10 epochs, with the `RNNWeightMatrixHistoryI` callback recording the recurrent kernel at the start of training and after each epoch.
5. Evaluates on the test set; computes the four graph-theoretic metrics on the trained recurrent matrix (see [08 — Outputs & metrics](08-Outputs-and-Metrics.md)).
6. Appends one row of results to the per-task accumulator.

After all variants × all simulations × all enabled tasks have run, the **Save** cell writes the CSVs.

## Why the CSV row order matters

Every `<metric>_raw.csv` written by the Save cell uses the same eleven variants in the same dict order as its row index. The downstream statistical pipeline (`StatisticalAnalysis/wiki/02-Input-Data-Schema.md`) hard-codes this order when it slices by factor (`W` slot, `D` slot, `C` slot). If you ever add, remove, or reorder variants, update the row-index list in both places at once.
