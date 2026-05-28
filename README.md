# CorticalBlueprintRNN

Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks.

This repository accompanies the paper *"Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks"* (Shakiba, Rokni, Mohammadi, Dehghani). It contains the manuscript, the training pipeline that builds and trains eleven biologically grounded RNN variants on three cognitive decision-making tasks using data from the [MICrONS](https://www.microns-explorer.org/cortical-mm3) program, and the post-hoc statistical analysis used in the paper's main text and Supplementary Results.

**Central finding.** Recurrent networks initialized from cortical functional connectivity and embedded in measured cortical coordinates learn faster and generalize better than baseline RNNs. Among the priors, function-derived weight initialization contributes the most, real spatial embedding adds a robust secondary benefit, and communicability-aware regularization refines the topology of the learned solution.

---

## Repository layout

```
CorticalBlueprintRNN/
├── README.md                           ← you are here
├── CorticalBlueprintRNN.pdf            ← the manuscript 
│
├── Model/         ← model & training pipeline
│   ├── README.md                       ← repo readme + Colab badge + wiki index
│   ├── INF.md                          ← table of available MICrONS session/scan/field combinations
│   ├── CorticalBlueprintRNN.ipynb     ← main training notebook (11 variants × 3 tasks × 20 sims)
│   ├── DATA/info/<sess>_<scan>_<field>/  ← preprocessed MICrONS inputs (one folder per field)
│   └── wiki/     ← detailed documentation for the training notebook
│       ├── Home.md, _Sidebar.md
│       ├── 01-Overview.md
│       ├── 02-Simulation-Parameters.md
│       ├── 03-Data-Inputs.md
│       ├── 04-Bio-Weight-Initialization.md
│       ├── 05-Tasks.md
│       ├── 06-Regularizers-and-Communicability.md
│       ├── 07-Model-Variants.md
│       ├── 08-Outputs-and-Metrics.md
│       └── 09-Reproduction-Guide.md
│
└── StatisticalAnalysis/               ← post-hoc nonparametric stats
    ├── 6_6_2_sttc_corr.ipynb           ← per-metric comparison scratchpad
    ├── 6_6_2_sttc_corr_all_metric.ipynb       ← distribution diagnostics (justifies KW/MWU)
    ├── 6_6_2_sttc_corr_single_metric.ipynb    ← 95% confidence intervals (Table 2)
    ├── The_Effect_of_w,_d_and_c_variants_Task_{1,2,3}.ipynb  ← KW + MWU + Holm per task (S5–S7)
    ├── The_Effect_of_w_d_and_c_variants .ipynb               ← earlier combined version
    └── wiki/                           ← detailed documentation for the statistical pipeline
        ├── Home.md, _Sidebar.md
        ├── 01-Overview.md
        ├── 02-Input-Data-Schema.md
        ├── 03-Distribution-Diagnostics.md
        ├── 04-Confidence-Intervals.md
        ├── 05-Effect-of-W-D-C-Variants.md
        ├── 06-Statistical-Methods.md
        └── 07-Reproduction-Guide.md
```

The two wikis are kept side-by-side rather than merged: [`Model/wiki/`](Model/wiki/Home.md) documents the training notebook, and [`StatisticalAnalysis/wiki/`](StatisticalAnalysis/wiki/Home.md) documents the statistical pipeline that consumes its outputs.

### Wiki quick links

**Training pipeline** ([Home](Model/wiki/Home.md))
[Overview](Model/wiki/01-Overview.md) ·
[Parameters](Model/wiki/02-Simulation-Parameters.md) ·
[Data inputs](Model/wiki/03-Data-Inputs.md) ·
[Bio weight init](Model/wiki/04-Bio-Weight-Initialization.md) ·
[Tasks](Model/wiki/05-Tasks.md) ·
[Regularizers](Model/wiki/06-Regularizers-and-Communicability.md) ·
[Variants](Model/wiki/07-Model-Variants.md) ·
[Outputs & metrics](Model/wiki/08-Outputs-and-Metrics.md) ·
[Reproduction](Model/wiki/09-Reproduction-Guide.md)

**Statistical analysis** ([Home](StatisticalAnalysis/wiki/Home.md))
[Overview](StatisticalAnalysis/wiki/01-Overview.md) ·
[Input schema](StatisticalAnalysis/wiki/02-Input-Data-Schema.md) ·
[Distribution diagnostics](StatisticalAnalysis/wiki/03-Distribution-Diagnostics.md) ·
[Confidence intervals](StatisticalAnalysis/wiki/04-Confidence-Intervals.md) ·
[Effect of W, D, C](StatisticalAnalysis/wiki/05-Effect-of-W-D-C-Variants.md) ·
[Statistical methods](StatisticalAnalysis/wiki/06-Statistical-Methods.md) ·
[Reproduction](StatisticalAnalysis/wiki/07-Reproduction-Guide.md)

---

## End-to-end pipeline

```
                MICrONS public release
   (calcium imaging + EM-reconstructed connectome)
                       │
                       ▼
  preprocessing  ──  CaveClient (coregistration_manual_v4)
                     microns_phase3_nda  (deconvolved spikes)
                       │
                       ▼
       DATA/info/<sess>_<scan>_<field>/
         connectome_<...>.csv          ← structural adjacency
         positions_<...>.npy           ← 3D soma coordinates
         sttc_matrix.npy               ← spike-time tiling
         corr_matrix.npy               ← Pearson correlation
         precision_matrix.npy          ← inverse covariance
         spikes_<...>.npy
         root_ids / unit_ids / target_ids .npy
                       │
                       ▼
       CorticalBlueprintRNN.ipynb
         (11 model variants × 3 tasks × 20 simulation runs)
                       │
                       ▼
       <sess>_<scan>_<field>_sttc[-corr|-precision][_sign+]/
         config.txt
         metrics.csv    metrics_SD.csv
         Task{1,2,3}/{Accuracy,Loss,Validation_Accuracy,Validation_Loss,
                      Modularity,SmallWorldness,Assortativity,Entropy}_raw.csv
                       │
                       ▼
       StatisticalAnalysis/*.ipynb
         (KW omnibus, Mann–Whitney U pairwise, Holm correction,
          bootstrap CIs, distribution diagnostics)
                       │
                       ▼
       Paper figures + Tables 2, 3, S5–S7
```

---

## Methods at a glance

The paragraphs below summarize §IV of the paper (Methods, `main.tex` lines 272–574). Refer to the manuscript for full citations.

### Dataset

[MICrONS](https://www.microns-explorer.org/cortical-mm3) is a multimodal release covering mouse visual cortex (V1, LM, RL, AL) in the same animal. It pairs detailed EM reconstructions of >200,000 cells and 523 million synapses with two-photon calcium imaging from ~75,000 neurons; ~12,000 excitatory neurons are functionally coregistered. Structural data are accessed through `CaveClient` (table `coregistration_manual_v4`); deconvolved functional traces are accessed through the [microns_phase3_nda](https://github.com/datajoint/microns_phase3_nda) repo. The unit of analysis is one `(session, scan, field)` combination — a calcium-imaging session, an acquisition within it, and a specific imaged cortical region. The full list of available combinations is in [`Model/INF.md`](Model/INF.md); the canonical field used in the paper's main tables is `(6, 6, 2)` with 312 neurons.

### Functional measures

Three complementary pairwise functional measures are derived from the binned spike trains $b_i$:

- **Pearson correlation**
  $$C[i, j] = \frac{\langle b_i - \mu_i,\, b_j - \mu_j \rangle}{\sqrt{\langle b_i - \mu_i,\, b_i - \mu_i \rangle \cdot \langle b_j - \mu_j,\, b_j - \mu_j \rangle}}$$
- **Spike Time Tiling Coefficient** (Cutts & Eglen 2014) — less sensitive to firing-rate confounds and to silent periods than Pearson correlation:
  $$\text{STTC} = \tfrac{1}{2}\!\left(\frac{P_A - T_B}{1 - P_A T_B} + \frac{P_B - T_A}{1 - P_B T_A}\right)$$
- **Precision matrix** — the inverse covariance, $\Theta = \Sigma^{-1}$, which isolates direct conditional dependencies after partialling out every other neuron. Used as an alternative initializer to test whether direct dependencies alone are sufficient.

### Bio weight initialization

Recurrent weights for `W*` and `W!` variants are sampled from a log-normal distribution ($\mu = -0.5$, $\sigma = 0.5$, chosen to reflect biological heavy-tailed synaptic distributions) and modulated element-wise by the functional matrices:

$$
W_{\text{bio}} = W_{\text{lognormal}} \odot \text{Corr} \odot \text{STTC}
$$

<p align="right"><em>(Eq. 7)</em></p>

After construction, $W_{\text{bio}}$ is rescaled to mean 0.1, thresholded at 0.01 (to suppress near-zero connections), and rescaled to a spectral radius of 0.95 for stable recurrent dynamics. `W!` is a permutation of `W*` that preserves the marginal distribution while breaking the neuron-to-weight mapping. Non-bio variants use `Orthogonal` initialization. Replacing `Corr` with the precision matrix yields the precision-based init used in the robustness arm.

### Spatial embedding

Real neuronal coordinates (`D*`) are min–max-normalized per axis and converted to a pairwise Euclidean distance matrix. The control (`D`) replaces them with a regular three-dimensional grid that matches the number of neurons. Distances are dimensionless and directly comparable across fields.

### Communicability

Communicability is computed online during training from the absolute recurrent weight matrix:

$$C = e^{S^{-1/2} W S^{-1/2}} \tag{Eq.~5}$$

where $S$ is the diagonal matrix of node strengths and diagonal entries are zeroed. For the EMD arm (`C*`), the empirical communicability $\mathbf{C}_{\text{emp}}$ is computed once from the MICrONS connectome with the same formula, and the model is regularized to match this empirical distribution via the Wasserstein distance between sorted quantiles.

### Regularization

Three forms appear across the variants:

| form | applies to |
|---|---|
| $\lambda\,\lVert W\rVert$                                                              | baseline $W$ only |
| $\lambda\,\lVert W \odot D \rVert$                                                     | `WD`, `WD*` |
| $\lambda\,\lVert W \odot D \odot C \rVert$                                             | direct-$C$ variants (`*DC`, `*D*C`) |
| $\lambda\,\lVert W \odot D \rVert + \lambda_{\text{EMD}}\cdot\text{EMD}(C_{\text{emp}},C_{\text{art}})$ | EMD-$C^{*}$ variants |

with $\lambda = 0.3$, $\lambda_{\text{EMD}} = 0.1$ held fixed across all tasks, fields and variants.

### The eleven model variants

Asterisks denote empirical constraints; `!` denotes a structure-disrupting permutation; `D` vs `D*` distinguishes grid vs. real coordinates; `C` vs `C*` distinguishes direct communicability regularization vs. an Earth-Mover's-Distance penalty against empirical communicability.

| # | Model       | Bio init  | Spatial   | Communicability  | Regularizer                                                                                              |
|---|-------------|-----------|-----------|------------------|----------------------------------------------------------------------------------------------------------|
| a | **W\*D\*C**     | yes (W\*) | real (D\*) | direct (C)      | $\lambda\,\lVert W^{*}\odot D^{*}\odot C\rVert$                                                          |
| b | WD\*C       | no        | real (D\*) | direct (C)       | $\lambda\,\lVert W\odot D^{*}\odot C\rVert$                                                              |
| c | WDC         | no        | grid (D)  | direct (C)       | $\lambda\,\lVert W\odot D\odot C\rVert$                                                                  |
| d | WD\*        | no        | real (D\*) | none             | $\lambda\,\lVert W\odot D^{*}\rVert$                                                                     |
| e | WD          | no        | grid (D)  | none             | $\lambda\,\lVert W\odot D\rVert$                                                                         |
| f | W (Simple)  | no        | none      | none             | $\lambda\,\lVert W\rVert$                                                                                |
| g | **W\*D\*C\***   | yes (W\*) | real (D\*) | EMD (C\*)        | $\lambda\,\lVert W^{*}\odot D^{*}\rVert + \lambda_{\text{EMD}}\cdot\text{EMD}(C_{\text{emp}},C_{\text{art}})$ |
| h | WD\*C\*     | no        | real (D\*) | EMD (C\*)        | $\lambda\,\lVert W\odot D^{*}\rVert + \lambda_{\text{EMD}}\cdot\text{EMD}(C_{\text{emp}},C_{\text{art}})$    |
| i | W\*DC\*     | yes (W\*) | grid (D)  | EMD (C\*)        | $\lambda\,\lVert W^{*}\odot D\rVert + \lambda_{\text{EMD}}\cdot\text{EMD}(C_{\text{emp}},C_{\text{art}})$    |
| j | W!D\*C      | yes (W!)  | real (D\*) | direct (C)       | $\lambda\,\lVert W^{!}\odot D^{*}\odot C\rVert$                                                          |
| k | W!D\*C\*    | yes (W!)  | real (D\*) | EMD (C\*)        | $\lambda\,\lVert W^{!}\odot D^{*}\rVert + \lambda_{\text{EMD}}\cdot\text{EMD}(C_{\text{emp}},C_{\text{art}})$ |

### Tasks

All three tasks come pre-implemented in the training notebook as one-hot integer-input sequences; each batch is 128 trials, with 5,120 training, 2,560 validation and 2,560 test trials per task.

- **One-Choice Inference** (Achterberg et al. 2023). Stimulus A (20 steps) → delay (10) → stimulus B (20). 8-channel input (4 goal + 4 candidate directions). 4-way output. Tests one-step inference / working memory.
- **Perceptual Decision-Making** (Britten et al. 1992). 1-step fixation → 30-step noisy stimulus at coherence $\in\{0, 6.4, 12.8, 25.6, 51.2\}\%$ → 10-step delay. 3-channel input (fixation + 2 evidence). 2-way output. Tests evidence integration.
- **Go/NoGo** (Zhang et al. 2019). 5-step fixation → 20-step stimulus → 10-step delay → 5-step decision. 3-channel input. 2-way output. Tests short-term memory + output gating.

### Architecture & training

A `Gaussian noise (σ=0.05)` layer → `SimpleRNN(units=N, activation='relu')` with custom initializer / regularizer per variant → `Dense(softmax)`. `N` equals the number of neurons in the selected field. Adam optimizer, categorical cross-entropy, 10 epochs, 20 independent simulation runs per variant.

### Metrics

Every simulation logs eight metrics per task (the same set produced as `*_raw.csv` files):

- **Accuracy / Loss / Validation_Accuracy / Validation_Loss** — task performance.
- **Entropy** — Shannon entropy over a Gaussian-KDE estimate of the recurrent-weight distribution on a 100-point grid (low = structured, high = random-like).
- **Modularity** $Q$ — Clauset–Newman–Moore greedy community detection.
- **Assortativity** $r$ — Newman degree-degree correlation; $r>0$ = hub-clustered, $r<0$ = hub-and-spoke.
- **Small-worldness** $\sigma$ — Humphries normalization against 1,000 random binary nulls.

All graph metrics are computed on undirected binary graphs obtained by taking $|W|$, thresholding to the top 10% of strongest entries, and zeroing self-loops.

### Statistical analysis

Because metrics across the 20 simulation runs are heavy-tailed and not Gaussian, the paper uses a nonparametric pipeline (justified by the diagnostics in `StatisticalAnalysis/6_6_2_sttc_corr_all_metric.ipynb`):

1. Rank-based three-way ANOVA over factors $W$, $D$, $C$.
2. Kruskal–Wallis omnibus per factor.
3. Mann–Whitney U pairwise post-hoc, with Holm step-down correction.
4. Bootstrap and t-distribution 95% confidence intervals for Table 2.

These are implemented in [`StatisticalAnalysis/wiki/`](StatisticalAnalysis/wiki/Home.md) — see in particular [`06-Statistical-Methods.md`](StatisticalAnalysis/wiki/06-Statistical-Methods.md) and [`07-Reproduction-Guide.md`](StatisticalAnalysis/wiki/07-Reproduction-Guide.md).

---

## Quickstart (developers)

### 1. Clone & install

```bash
git clone <this-repo-url>
cd CorticalBlueprintRNN
pip install bctpy tensorflow scipy matplotlib numpy pandas seaborn tqdm networkx scikit-learn statsmodels
```

Python ≥ 3.10 is recommended. The training notebook also runs unmodified in Google Colab — open `Model/CorticalBlueprintRNN.ipynb` directly from the [Colab badge](https://colab.research.google.com/drive/1frTPKt8WgwJO8FUZD0UmOM5MPmvvOXbs?usp=sharing) in `Model/README.md`.

### 2. Train

Open `Model/CorticalBlueprintRNN.ipynb`. The "Simulation Parameters" cell exposes the knobs you need; defaults match the paper:

| parameter | shipped default | paper config | meaning |
|---|---|---|---|
| `session_scan_field` | `'6_6_2'` | `'6_6_2'` | which MICrONS field to load from `DATA/info/<sess>_<scan>_<field>/` |
| `number_of_nodes` | `312` | `312` | must equal the field's neuron count (see `INF.md`) |
| `network_grid` | `(12, 13, 2)` | `(12, 13, 2)` | grid shape for the `D` (artificial-coordinates) control |
| `simulations` | `1` | `20` | runs per variant — bump to 20 to match the paper |
| `epochs` (in training loop) | `10` | `10` | per simulation |
| `regu_strength` | `0.3` | `0.3` | $\lambda$ |
| `emd_strength` | `0.1` | `0.1` | $\lambda_{\text{EMD}}$ |
| `sign_constraint` | `False` | `False` (Table 2) / `True` (Table 3) | if `True`, force positive recurrent weights |
| `use_precison_matrix` | `False` | `False` (main) / `True` (precision robustness arm) | if `True`, use $\Theta$ instead of correlation. **Note the misspelling** (`precison`, one `i`) — it appears identically in the stat notebooks |
| `use_only_sttc` | `False` | `False` | if `True`, bio init uses only STTC (skips correlation/precision) |
| `TASK1, TASK2, TASK3` | all `True` | all `True` | toggle individual tasks |

Outputs land in a folder named `<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/`. For the defaults above this is `6_6_2_sttc-corr/`. The folder contains:

```
<run>/
├── config.txt                       ← echoed parameters
├── metrics.csv                      ← mean over the 20 runs, per variant per task
├── metrics_SD.csv                   ← standard deviation
└── Task{1,2,3}/
    ├── Accuracy_raw.csv
    ├── Loss_raw.csv
    ├── Validation_Accuracy_raw.csv
    ├── Validation_Loss_raw.csv
    ├── Modularity_raw.csv
    ├── SmallWorldness_raw.csv
    ├── Assortativity_raw.csv
    └── Entropy_raw.csv
```

In each `*_raw.csv`, rows index the 11 variants in canonical order (`W*D*C, WD*C, WDC, WD*, WD, W, W*D*C*, WD*C*, W*DC*, W!D*C, W!D*C*`) and columns index the 20 simulation runs.

### 3. Run the statistical analysis

The stat notebooks expect the training-output folder under `/content/data/simu/<base>/` (Colab convention). Either:

- run them in Colab with `simu.zip` mounted under `/content/drive/MyDrive/ISP/`, or
- update the `folder_name` constant at the top of each notebook to point at your local output directory.

Then run, in this order:

1. `6_6_2_sttc_corr_all_metric.ipynb` — verifies the metric distributions are non-Gaussian (justifies the nonparametric tests).
2. `6_6_2_sttc_corr_single_metric.ipynb` — produces the 95% CIs that appear parenthesized in Table 2 of the paper.
3. `The_Effect_of_w,_d_and_c_variants_Task_1.ipynb`, `_Task_2.ipynb`, `_Task_3.ipynb` — Kruskal–Wallis omnibus + Mann–Whitney U post-hoc with Holm correction per factor ($W$, $D$, $C$), producing Supplementary Tables S5–S7.

Full step-by-step recipe, including the robustness arms (precision matrix, sign constraint, cross-field), is in [`StatisticalAnalysis/wiki/07-Reproduction-Guide.md`](StatisticalAnalysis/wiki/07-Reproduction-Guide.md).

---

## Reproducing the paper's numbers

The accuracies in Table 2 (`main.tex` lines 184–203) are produced with:

- `session = 6`, `scan = 6`, `field = 2` (312 nodes; grid control `(12, 13, 2)`)
- `simulations = 20`, 10 training epochs each (the shipped notebook defaults `simulations` to `1` for fast iteration — bump it to `20` before running)
- $\lambda = 0.3$, $\lambda_{\text{EMD}} = 0.1$
- `sign_constraint = False` for Table 2; `True` for Table 3 (positive-only recurrence)
- correlation-based bio init (`use_precison_matrix = False` — note the misspelling) for the main tables; rerun with `True` for the precision-matrix robustness arm

Other `(session, scan, field)` combinations from `Model/INF.md` reproduce the cross-field robustness results.

---

## Data & code availability

- MICrONS portal: <https://www.microns-explorer.org/cortical-mm3>
- Calcium imaging (DANDI): <https://dandiarchive.org/dandiset/000402>
- EM segmentation + morphology (BossDB): <https://bossdb.org/project/microns-minnie>
- Preprocessed weight-init inputs (this repo): `Model/DATA/info/<sess>_<scan>_<field>/`

---

## Citation

```bibtex
@article{shakiba2025cortical,
  title   = {Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks},
  author  = {Shakiba, Mo and Rokni, Rana and Mohammadi, Mohammad and Dehghani, Nima},
  journal = {[in preparation]},
  year    = {2025}
}
```

## Acknowledgments

N.D. is supported by NIH grant R24MH117295. M.S., R.R. and M.M. thank [Neuromatch Academy](https://neuromatch.io/) for support.
