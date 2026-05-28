# 03 — Data inputs

This page is the contract between the preprocessed MICrONS arrays and the training notebook. If you change either side, change this page too.

## Where the inputs come from

The notebook's data-download cell pulls the preprocessed archive directly from the public mirror:

```bash
curl -sS -L -o INF.md   https://github.com/moneuron/neuro-constrained-RNN/raw/refs/heads/main/INF.md
curl -sS -L -o DATA.zip https://github.com/moneuron/neuro-constrained-RNN/raw/refs/heads/main/DATA.zip
unzip -q DATA.zip && rm -f DATA.zip
```

This produces:

- [`INF.md`](../INF.md) — the per-field index (session, scan, field, neuron count, grid shape).
- `DATA/info/<sess>_<scan>_<field>/` — one folder per imaged field, each containing the five files described below.

If you ship preprocessed data alongside the repo (as is the case here), the `DATA/` folder is already populated and the download cell becomes a no-op; the `Loading` cell just reads from disk.

## Per-field folder layout

```
DATA/info/<sess>_<scan>_<field>/
├── connectome_<sess>_<scan>_<field>.csv   ← structural adjacency (EM-derived)
├── positions_<sess>_<scan>_<field>.npy    ← (N, 3) soma coordinates
├── sttc_matrix.npy                        ← (N, N) Spike Time Tiling Coefficient
├── corr_matrix.npy                        ← (N, N) Pearson correlation
├── precision_matrix.npy                   ← (N, N) inverse covariance (precision)
├── spikes_<sess>_<scan>_<field>.npy       ← (N, T) deconvolved spike trains (raw source)
├── root_ids_<sess>_<scan>_<field>.npy     ← proofread EM segmentation IDs
├── unit_ids_<sess>_<scan>_<field>.npy     ← functional ROI IDs (unique per scan)
└── target_ids_<sess>_<scan>_<field>.npy   ← postsynaptic targets for the connectome graph
```

`N` = neuron count for the field (see [`INF.md`](../INF.md)). The training notebook reads the first five files; the remaining three are the raw identifiers that link MICrONS structural and functional records — useful for re-deriving the functional matrices, not used at training time.

## Where each file comes from

| file | source | how it was derived |
|---|---|---|
| `connectome_*.csv` | CaveClient query against `coregistration_manual_v4` plus pre/postsynaptic adjacency lookups | weighted/binary structural adjacency among the field's neurons |
| `positions_*.npy` | CaveClient `position` field, voxel-space (4×4×40 nm) | min-max normalized per axis at training time to produce dimensionless `D*` |
| `sttc_matrix.npy` | computed from `spikes_*.npy` via the Cutts & Eglen 2014 formulation | pairwise temporal-tiling coefficient (less sensitive to rate confounds than Pearson r) |
| `corr_matrix.npy` | Pearson correlation on binned spike trains | the standard pairwise co-activity statistic |
| `precision_matrix.npy` | inverse of the covariance of binned spike trains | conditional dependencies after partialling out all other neurons |
| `spikes_*.npy` | deconvolved calcium traces from [microns_phase3_nda](https://github.com/datajoint/microns_phase3_nda) | the raw functional data |
| `root_ids` / `unit_ids` / `target_ids` | CaveClient | structural↔functional joins for re-deriving any matrix above |

For the methods background, see §IV.A–B of the paper.

## What the notebook does on load

The `Loading` section (notebook section 5) parses the field name and reads each array:

```python
session, scan, field = map(int, session_scan_field.split('_'))

connectome = pd.read_csv(f'DATA/info/{session_scan_field}/connectome_{session_scan_field}.csv')
positions  = np.load(f'DATA/info/{session_scan_field}/positions_{session_scan_field}.npy')
sttc       = np.load(f'DATA/info/{session_scan_field}/sttc_matrix.npy')
corr       = np.load(f'DATA/info/{session_scan_field}/corr_matrix.npy')
precision  = np.load(f'DATA/info/{session_scan_field}/precision_matrix.npy')
```

Each `(N, N)` functional matrix is then min-max normalized to `[0, 1]` per matrix and used by the bio-weight construction in section 6.

## Picking a field

The same `DATA/info/` folder contains many fields. Switching is just a parameters-cell change (see [02 — Simulation parameters](02-Simulation-Parameters.md)). Cross-check that:

- `number_of_nodes` in the parameters cell equals `N` for the chosen field (see [`INF.md`](../INF.md));
- `network_grid` factorizes to that `N`.

A mismatch on either of these will either crash the first matmul (visible) or silently misalign the `D` control (worse).
