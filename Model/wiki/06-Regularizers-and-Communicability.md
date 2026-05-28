# 06 — Regularizers & communicability

Notebook section: **Regularization**

Four custom `tf.keras.regularizers.Regularizer` subclasses implement the four families of regularization used across the eleven variants. They differ only in their `__call__`:

| class | regularization term | used by variants |
|---|---|---|
| `W`        | $\lambda \, \lVert W \rVert$                                                                                            | the baseline `W` (Simple RNN) |
| `WD`       | $\lambda \, \lVert W \odot D \rVert$                                                                                    | `WD`, `WD*` |
| `WDC`      | $\lambda \, \lVert W \odot D \odot C \rVert$                                                                            | `WDC`, `WD*C`, `W*D*C`, `W!D*C` |
| `WDC_EMD`  | $\lambda \, \lVert W \odot D \rVert + \lambda_{\text{EMD}} \cdot \text{EMD}(C_{\text{emp}}, C_{\text{art}})$            | `W*D*C*`, `WD*C*`, `W*DC*`, `W!D*C*` |

Each class is constructed once per variant with the distance tensor `D` (real or grid) and (for `WDC_EMD`) the empirical communicability distribution `C_emp` baked in as `self.*` attributes.

## `W` — plain L1

```python
def __call__(self, abs_weight_matrix):
    return tf.math.reduce_sum(abs_weight_matrix)
```

(`abs_weight_matrix = tf.math.abs(W)` is computed once at the top of every regularizer.)

## `WD` — spatially weighted L1

```python
def __call__(self, abs_weight_matrix):
    return self.wd * tf.math.reduce_sum(
        tf.math.multiply(abs_weight_matrix, self.distance_tensor))
```

`self.distance_tensor` is the pairwise Euclidean distance matrix. For `D*` variants it is built from the min-max-normalized MICrONS coordinates; for `D` variants it is built from a regular three-dimensional grid (shape `network_grid`).

## `WDC` — spatial + online communicability

```python
def __call__(self, abs_weight_matrix):
    # Online communicability: C = exp(S^(-1/2) W S^(-1/2)), diagonal zeroed
    stepI   = tf.math.reduce_sum(abs_weight_matrix, axis=1)
    stepII  = tf.math.pow(stepI, -0.5)
    stepIII = tf.linalg.diag(stepII)
    stepIV  = tf.linalg.expm(stepIII @ abs_weight_matrix @ stepIII)
    comms_matrix = tf.linalg.set_diag(stepIV, tf.zeros(stepIV.shape[0:-1]))

    comms_weight_matrix = tf.math.multiply(abs_weight_matrix, comms_matrix)
    return self.wd * tf.math.reduce_sum(
        tf.math.multiply(comms_weight_matrix, self.distance_tensor))
```

This implements Eq. 5 of the paper, recomputed at every training step from the current weights — `C` evolves with `W` rather than being frozen at init. The penalty therefore acts on the product $W \odot D \odot C$, which encourages connections that are simultaneously short and *unnecessary* (already easy to reach via the rest of the graph) to shrink.

## `WDC_EMD` — spatial + EMD on communicability distributions

```python
def __call__(self, abs_weight_matrix):
    # spatial component (same as WD)
    spatial_loss = self.wd * tf.math.reduce_sum(
        tf.math.multiply(abs_weight_matrix, self.distance_tensor))

    # Earth Mover's Distance between sorted flattened communicability distributions
    #   - C_art: built online from W using the same expm formula above
    #   - C_emp: precomputed from the MICrONS connectome and stored in self.C_emp
    emd_loss = wasserstein_1(sort(flatten(C_art)), sort(flatten(self.C_emp)))

    return spatial_loss + self.emd_factor * emd_loss
```

The `C*` form replaces direct communicability modulation with a *distributional* match: the network is penalized when the distribution of its own communicability values diverges from the empirical distribution measured from the MICrONS structural connectome. This is the looser, more topological constraint — and the one that allows `W*D*C*` to reach high accuracy without developing strong modularity (see Results §III.B).

## How communicability is computed

The expression above is exactly the formulation in §IV.C of the paper:

$$C = e^{S^{-1/2} W S^{-1/2}}$$

with $S$ the diagonal matrix of node strengths (degree under absolute weights) and the diagonal of the result zeroed (no self-communicability).

Three points worth knowing:

1. **It is online, not frozen.** `WDC.__call__` runs at every backward pass, so the gradient sees the dependency of $C$ on $W$ — the regularizer is fully differentiable through `tf.linalg.expm`.
2. **It uses absolute weights.** Even when `sign_constraint = False`, the regularization quantity is built from $|W|$ — so the penalty acts on connection magnitude regardless of sign.
3. **The empirical $C_{\text{emp}}$ in the EMD variants is computed once.** It is built from the weighted MICrONS connectome at notebook initialization and stored as a constant tensor; it does not move during training.

## Choosing `λ` and `λ_EMD`

Both are set in the parameters cell (`regu_strength = 0.3`, `emd_strength = 0.1`) and held fixed across every variant, task, and field. The paper notes (§IV.D, Training) that these were picked in pilot experiments and that qualitative results are stable under moderate variation. If you change them, expect all four metric panels in Fig. 2 to shift.
