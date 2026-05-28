# 04 — Bio weight initialization

Notebook section: **Bio Weight Init.**

## Goal

Build a single initial recurrent weight matrix `W_bio` of shape `(N, N)` that carries the cortical functional structure — heavy-tailed magnitudes, modulated by inter-neuron co-activity and temporal tiling — so the `W*` variants start training from a biologically informed point in weight space, and the `W!` variants start from a permutation of the same matrix (same marginal, scrambled mapping).

This implements Eq. 7 of the paper:

$$W_{\text{bio}} = W_{\text{lognormal}} \odot \text{Corr} \odot \text{STTC}$$

## What the notebook actually does

The construction happens in straight numpy / TensorFlow inside the *Bio Weight Init.* cell (no custom Keras `Initializer` subclass is involved at this stage — the matrix is built numerically, then later handed to the `SimpleRNN` layer through a thin `ConnectomeInitializer` wrapper).

The numeric pipeline is, in order:

```python
# 1. Sample the base log-normal recurrent weights
sigma = 0.5
mu    = -0.5
lognormal_weights = np.random.lognormal(mu, sigma, size=corr.shape)   # shape (N, N)

# 2. Modulate by the functional matrices
if use_only_sttc:
    bio_weights = lognormal_weights * normalized_sttc
else:
    if use_precison_matrix:
        bio_weights = lognormal_weights * normalized_perc * normalized_sttc
    else:
        bio_weights = lognormal_weights * normalized_corr * normalized_sttc

# 3. Rescale to a fixed mean
desired_mean   = 0.1
current_mean   = np.mean(bio_weights)
scaling_factor = desired_mean / current_mean if current_mean > 0 else 1.0
bio_weights   *= scaling_factor

# 4. Apply a hard minimum on non-zero entries
min_threshold = 0.01
bio_weights   = np.maximum(bio_weights, min_threshold * (bio_weights != 0))

# 5. Rescale to spectral radius 0.95 (for stable recurrent dynamics)
eigenvalues     = np.linalg.eigvals(bio_weights)
spectral_radius = np.max(np.abs(eigenvalues))
if spectral_radius > 0:
    bio_weights *= 0.95 / spectral_radius

# 6. Cast to a TF tensor for use as the SimpleRNN initial recurrent kernel
W_bio = tf.convert_to_tensor(bio_weights, dtype=tf.float32)
```

`normalized_corr`, `normalized_sttc`, `normalized_perc` are min-max scalings of the three functional matrices loaded in section 5 (see [03 — Data inputs](03-Data-Inputs.md)).

## Numerical knobs (constants in the notebook)

| symbol | value | rationale |
|---|---|---|
| log-normal $\mu$ | `-0.5` | reflects heavy-tailed cortical synaptic-weight distributions (Song et al. 2005; Loewenstein et al. 2011) |
| log-normal $\sigma$ | `0.5` | same |
| target mean | `0.1` | gives the matrix a consistent scale across fields with different neuron counts |
| floor / threshold | `0.01` | suppresses near-zero entries on the already-nonzero support so they don't vanish under multiplication |
| spectral radius | `0.95` | recurrent dynamics stable but expressive (Pascanu et al. 2013) |

## Variants of the construction

The same numeric pipeline produces three different matrices depending on the parameters cell:

| `use_only_sttc` | `use_precison_matrix` | resulting `W_bio` |
|---|---|---|
| `False` | `False` *(default)* | `lognormal ⊙ Corr ⊙ STTC` — the main-text bio init |
| `False` | `True` | `lognormal ⊙ Precision ⊙ STTC` — the precision-matrix robustness arm (paper §III.C) |
| `True` | (ignored) | `lognormal ⊙ STTC` — STTC-only arm |

The folder-name suffix on the output (`-corr` / `-precision`) records which version of the construction was used.

## The `W!` permutation

`W!` (variants `W!D*C` and `W!D*C*`) uses the same `W_bio` matrix, but with its entries permuted to destroy the neuron-to-weight mapping while preserving the marginal distribution. The permutation is done once per simulation inside the variant loop (so each `W!` run gets a fresh scramble, but every `W*` run sees the same `W_bio`).

The control answers a specific question: *does performance depend on the precise pairing of weights to neurons, or just on the empirical distribution of weight magnitudes?* The paper's answer (see Results §III.A and Table 2) is that `W*` and `W!` are statistically indistinguishable on task accuracy — distribution structure does the work.

## How the matrix gets into the `SimpleRNN` layer

The training loop wraps `W_bio` in a `ConnectomeInitializer` (a thin `tf.keras.initializers.Initializer` subclass that just returns the precomputed tensor when called). That initializer is passed as `recurrent_initializer=` when constructing the `SimpleRNN`. The non-bio variants (`W`, `WD`, `WDC`, `WD*`, `WD*C`) use `'Orthogonal'` instead — controlled by the `random_network_initialization` parameter.
