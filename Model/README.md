[![Wiki](https://img.shields.io/badge/-Wiki-blue?style=flat-square&logo=github)](wiki/Home.md)
[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/neurovium/CorticalBlueprintRNN/blob/main/Model/CorticalBlueprintRNN.ipynb)
<!-- [![arXiv](https://img.shields.io/badge/arXiv-preprint-D12424?logo=arxiv)](https://arxiv.org/abs/X) -->

# Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks

This repository contains the implementation for the research paper *"Harnessing cortical geometry, wiring, and function as inductive biases for recurrent neural networks"*. The work explores how incorporating neuronal constraints from the MICrONS dataset can enhance the performance and realism of recurrent neural networks (RNNs) on decision-making tasks.

For the methodology, parameter reference, and step-by-step reproduction guide, see the [**wiki**](wiki/Home.md).

## Repository Structure

```
Model/
├── README.md                          # This file
├── INF.md                             # Per-field index (session, scan, field, neuron count, grid)
├── CorticalBlueprintRNN.ipynb        # Main experimental notebook
├── DATA/info/<sess>_<scan>_<field>/   # Preprocessed MICrONS arrays (one folder per field)
└── wiki/        # Detailed documentation (see below)
```

## Wiki contents

| page | what it covers |
|---|---|
| [Home](wiki/Home.md) | Landing page + table of contents |
| [01 — Overview](wiki/01-Overview.md) | Section-by-section walkthrough of the notebook |
| [02 — Simulation parameters](wiki/02-Simulation-Parameters.md) | Every knob, with shipped defaults vs paper config |
| [03 — Data inputs](wiki/03-Data-Inputs.md) | The `DATA/info/<sess>_<scan>_<field>/` contract |
| [04 — Bio weight initialization](wiki/04-Bio-Weight-Initialization.md) | Lognormal sampling, ⊙ Corr ⊙ STTC, spectral-radius rescale (Eq. 7) |
| [05 — Tasks](wiki/05-Tasks.md) | One-Choice Inference, Go/NoGo, Perceptual Decision-Making — timing & shapes |
| [06 — Regularizers & communicability](wiki/06-Regularizers-and-Communicability.md) | The four `Regularizer` classes; online communicability (Eq. 5); EMD form |
| [07 — Model variants](wiki/07-Model-Variants.md) | The eleven variants in canonical dict order |
| [08 — Outputs & metrics](wiki/08-Outputs-and-Metrics.md) | Output folder layout + eight metrics |
| [09 — Reproduction guide](wiki/09-Reproduction-Guide.md) | End-to-end recipe + robustness arms |

## Quick start

1. Open `CorticalBlueprintRNN.ipynb` in [Colab](https://colab.research.google.com/drive/1frTPKt8WgwJO8FUZD0UmOM5MPmvvOXbs?usp=sharing) or locally (`pip install bctpy tensorflow scipy matplotlib numpy pandas seaborn tqdm networkx`).
2. Set the parameters cell — for the paper config bump `simulations` from `1` to `20`. Full reference in [02 — Simulation parameters](wiki/02-Simulation-Parameters.md).
3. Run all cells. Outputs land in `<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/` — see [08 — Outputs & metrics](wiki/08-Outputs-and-Metrics.md).
4. Hand the output folder to the post-hoc statistical pipeline — see the [StatisticalAnalysis wiki](../StatisticalAnalysis/wiki/Home.md).

## Related

- [Top-level repo README](../README.md) — high-level methods recap + how this folder fits with `StatisticalAnalysis/`
- [StatisticalAnalysis wiki](../StatisticalAnalysis/wiki/Home.md) — KW / MWU / Holm / bootstrap pipeline that consumes the CSVs produced here
