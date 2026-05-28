# 01 — Overview

`CorticalBlueprintRNN.ipynb` is one long notebook organized as a linear pipeline. This page maps its section structure to what each part does, so you know where to jump when editing or debugging.

## Section structure

Each item below corresponds to a markdown header in the notebook:

| # | section | role |
|---|---|---|
| 1 | *(title cell)* | Title + Colab badge |
| 2 | *(install cell)* | `pip install` of `bctpy`, `tensorflow`, `scipy`, `matplotlib`, `numpy`, `pandas`, `seaborn`, `tqdm`, `networkx` |
| 3 | *(data download cell)* | `curl` `INF.md` + `DATA.zip` from the GitHub repo and unzip in place |
| 4 | *(simulation parameters cell)* | The single source-of-truth for all knobs (see [02 — Simulation parameters](02-Simulation-Parameters.md)) |
| 5 | **Loading** | Load `connectome_*.csv`, `sttc_matrix.npy`, `corr_matrix.npy`, `precision_matrix.npy`, `positions_*.npy` for the selected field |
| 6 | **Bio Weight Init.** | Build `W_bio` from a lognormal sample modulated by the functional matrices (see [04 — Bio weight initialization](04-Bio-Weight-Initialization.md)) |
| 7 | **Tasks** | Define three task-generator classes: `mazeGeneratorI`, `GoNogoGeneratorI`, `PerceptualDecisionMakingGeneratorI` (see [05 — Tasks](05-Tasks.md)) |
| 8 | **Generate datasets for training** | Instantiate the generators and build `(X_train, y_train, X_val, y_val, X_test, y_test)` tuples |
|   | ↳ *TASK 1 / 1-step Inference* | One-Choice Inference dataset |
|   | ↳ *TASK 2 / GoNogo* | Go/NoGo dataset |
|   | ↳ *TASK 3 / PerceptualDecisionMaking* | PDM dataset |
| 9 | **Regularization** | Define the four custom regularizer classes: `W`, `WD`, `WDC`, `WDC_EMD` (see [06 — Regularizers & communicability](06-Regularizers-and-Communicability.md)) |
| 10 | **Model Defenition** | Define the `RNNWeightMatrixHistoryI` callback and the model-building helper |
| 11 | **Model Training** | Per-task training loops over the eleven variants |
|   | ↳ *TASK1* | Train all variants on One-Choice Inference |
|   | ↳ *TASK2* | Train all variants on Go/NoGo |
|   | ↳ *TASK3* | Train all variants on PDM |
| 12 | **Save** | Write `config.txt`, `metrics.csv`, `metrics_SD.csv`, and `Task{1,2,3}/<metric>_raw.csv` to the run folder |

> The notebook keeps the original (slightly misspelled) headers — `Model Defenition`. They are preserved verbatim above so a find-in-notebook for the section name will work.

## Execution model

Top to bottom, no skipped cells. The training cells are the slow ones: with `simulations = 20` × 11 variants × 3 tasks × 10 epochs, runtime is several hours on CPU or under an hour on a modest GPU. The shipped default is `simulations = 1` for fast iteration — bump it to `20` before kicking off the paper run.

## How configuration flows through

```
Simulation Parameters cell
        │
        ├── session_scan_field → which folder under DATA/info/ to load
        ├── number_of_nodes, network_grid → wired into the RNN layer + the D control
        ├── sign_constraint → name suffix (_sign+) + non-negative weight constraint
        ├── use_only_sttc / use_precison_matrix → name suffix (-corr / -precision)
        ├── regu_strength, emd_strength → λ and λ_EMD in regularizers
        ├── noise_level → Gaussian-noise input layer σ
        ├── activation_function → SimpleRNN activation
        └── TASK1/TASK2/TASK3 → which task training loops fire

Save cell uses the same parameters to construct the output folder name.
```

The naming convention is:

```
<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/
```

with `-corr` vs `-precision` chosen by `use_precison_matrix` (and dropped entirely if `use_only_sttc = True`), and `_sign+` appended iff `sign_constraint = True`.

## Where to look when something breaks

| symptom | section to inspect first |
|---|---|
| "file not found" loading data | sections 3 (data download) and 5 (Loading) |
| shape mismatch in RNN layer | parameters cell — `number_of_nodes` must equal the field's neuron count and factorize `network_grid` |
| `W_bio` has NaNs or zeros | section 6 (Bio Weight Init.) — check the spectral-radius rescale guard |
| variant trains to chance | parameters cell — likely `simulations = 1` and a bad seed; bump and retry |
| empty `*_raw.csv` files | section 12 (Save) — make sure all three TASKx toggles match the training loops that actually ran |
