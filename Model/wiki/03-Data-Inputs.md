# 03 — Data inputs

## Where the inputs come from

The notebook expeccts all inputs to be **present locally next to the notebook**.
You should have the following structure:

```
INF.md
DATA/
  info/
    <session>_<scan>_<field>/
      ...
```

### Required files

* `INF.md` — index of all available fields (session, scan, field metadata)
* `DATA/info/<session>_<scan>_<field>/` — preprocessed arrays per field

Each field folder contains the five preprocessed files required by the notebook pipeline (described below).

### Setup

If you are running locally or in Colab, simply ensure that:

* The `DATA/` directory is placed next to the notebook
* `INF.md` is in the same directory as the notebook

Then run the notebook normally. The loading cell will read directly from disk.

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
