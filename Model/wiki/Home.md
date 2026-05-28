# CorticalBlueprintRNN — Wiki

This wiki documents the training pipeline `CorticalBlueprintRNN.ipynb` — the notebook that builds and trains the eleven biologically grounded RNN variants on three cognitive tasks and writes out the per-run CSVs that the [StatisticalAnalysis wiki](../../StatisticalAnalysis/wiki/Home.md) consumes.

## What the notebook does, in one paragraph

For each `(session, scan, field)` of the MICrONS dataset specified at the top, the notebook loads the preprocessed connectome, soma coordinates, and three functional matrices (STTC, Pearson correlation, precision); builds a biologically informed initial recurrent-weight matrix `W_bio = W_lognormal ⊙ Corr ⊙ STTC` rescaled to a fixed mean and spectral radius; generates trial data for the three tasks (One-Choice Inference, Go/NoGo, Perceptual Decision-Making); instantiates eleven RNN variants that selectively combine biological initialization, real-vs-grid spatial embedding, and direct-vs-EMD communicability regularization; trains each variant for a configurable number of independent simulation runs; and writes per-task, per-metric `*_raw.csv` files plus aggregate `metrics.csv` / `metrics_SD.csv` summaries to a folder whose name encodes the configuration.

## Contents

| page | what it covers |
|---|---|
| [01-Overview](01-Overview.md) | Notebook structure; section-by-section walkthrough |
| [02-Simulation-Parameters](02-Simulation-Parameters.md) | Every knob in the parameters cell, with shipped defaults vs paper config |
| [03-Data-Inputs](03-Data-Inputs.md) | The `DATA/info/<sess>_<scan>_<field>/` contract and where each file comes from |
| [04-Bio-Weight-Initialization](04-Bio-Weight-Initialization.md) | Lognormal sampling, ⊙ Corr ⊙ STTC, mean & spectral-radius rescale (Eq. 7) |
| [05-Tasks](05-Tasks.md) | Three task generators — timing, input/output shapes, semantics |
| [06-Regularizers-and-Communicability](06-Regularizers-and-Communicability.md) | The four custom regularizer classes; online communicability (Eq. 5); EMD form |
| [07-Model-Variants](07-Model-Variants.md) | The eleven variants — initialization × embedding × communicability matrix |
| [08-Outputs-and-Metrics](08-Outputs-and-Metrics.md) | Output folder layout; the eight metrics and how they are computed |
| [09-Reproduction-Guide](09-Reproduction-Guide.md) | End-to-end recipe to regenerate the paper's training outputs |

## Quick context

- **Entry point:** `CorticalBlueprintRNN.ipynb` at the repo root.
- **Inputs:** preprocessed MICrONS arrays under `DATA/info/<sess>_<scan>_<field>/` (downloaded automatically from the GitHub-hosted `DATA.zip`).
- **Outputs:** `<sess>_<scan>_<field>_sttc[-corr|-precision][_sign+]/` containing `config.txt`, `metrics.csv`, `metrics_SD.csv`, and `Task{1,2,3}/<metric>_raw.csv`.
- **Downstream:** those CSVs are read by every notebook in [`StatisticalAnalysis/`](../../StatisticalAnalysis/) — see [its wiki](../../StatisticalAnalysis/wiki/Home.md) for the post-hoc tests, confidence intervals, and Supplementary Tables S5–S7.

## Related

- [Top-level repo README](../../README.md) — high-level methods recap + repo layout
- [`INF.md`](../INF.md) — table of all available `(session, scan, field)` combinations with neuron counts
- [`README.md`](../README.md) — short repo readme + Colab badge
