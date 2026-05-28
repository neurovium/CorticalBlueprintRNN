# 08 — Outputs & metrics

Notebook sections: **Model Training** (metrics computed per simulation) + **Save** (everything written to disk)

## Output folder layout

After the Save cell runs, you get one folder per `(session, scan, field, sign_constraint, init-source)` combination:

```
<session>_<scan>_<field>_sttc[-corr|-precision][_sign+]/
├── config.txt          ← echo of every parameter in the simulation-parameters cell
├── metrics.csv         ← mean across simulations, indexed by (task, metric); 11 variant columns
├── metrics_SD.csv      ← standard deviation across simulations (same shape)
├── Task1/
│   ├── Accuracy_raw.csv
│   ├── Assortativity_raw.csv
│   ├── Entropy_raw.csv
│   ├── Loss_raw.csv
│   ├── Modularity_raw.csv
│   ├── SmallWorldness_raw.csv
│   ├── Validation_Accuracy_raw.csv
│   └── Validation_Loss_raw.csv
├── Task2/              ← same 8 files
└── Task3/              ← same 8 files
```

If a `TASKn` toggle is `False` in the parameters cell, the corresponding subfolder will be empty (or absent). The downstream stat notebooks will silently skip any task whose CSV they can't find.

## `<metric>_raw.csv` schema

Each per-task raw CSV is a `(simulations × 11)` table:

- **Rows:** one per simulation run (indexed by `simulation`).
- **Columns:** the eleven variant keys in canonical dict order — `W*D*C, WD*C, WDC, WD*, WD, W, W*D*C*, WD*C*, W*DC*, W!D*C, W!D*C*`.

> **Important orientation note.** In *this* folder the orientation is `(simulations × variants)`. The downstream stat notebooks transpose where needed; they hard-code the variant column order, so don't reorder columns in the writer without updating both sides.

## `metrics.csv` and `metrics_SD.csv`

Aggregate views, one row per `(task, metric)` cell:

- **Rows:** the 24 combinations of (Task1, Task2, Task3) × (8 metrics).
- **Columns:** the eleven variants, in the same canonical order.
- **Cells:** mean (or SD) across the `simulations` runs.

These are convenient for spot-checking and for the in-paper performance tables, but the statistical pipeline reads `*_raw.csv` instead because it needs the full per-run distribution for KW / MWU / bootstrap.

## The eight metrics

The same eight metrics are computed for every variant on every task on every simulation. They are listed verbatim in `6_6_2_sttc_corr_all_metric.ipynb`:

```python
metrics = ['Accuracy', 'Assortativity', 'Entropy', 'Loss',
           'Modularity', 'SmallWorldness',
           'Validation_Accuracy', 'Validation_Loss']
```

### Training metrics

| metric | how it's computed |
|---|---|
| `Accuracy` | proportion of correctly classified trials on the test set |
| `Loss` | mean categorical cross-entropy on the test set |
| `Validation_Accuracy` | accuracy on the held-out validation set at the end of training |
| `Validation_Loss` | val cross-entropy at the end of training |

### Graph-theoretic metrics

These are all computed on the **trained recurrent weight matrix** after each simulation, on an undirected binary graph derived as follows:

1. Take $|W|$ (absolute values).
2. Threshold at the top 10% strongest entries — `thresh = np.quantile(W_abs, q=0.9)`.
3. Zero the diagonal (no self-loops).
4. Treat as undirected (symmetric) for the network-X computations.

| metric | code path | interpretation |
|---|---|---|
| `Modularity` ($Q$) | `nx.community.greedy_modularity_communities(G)` → `nx.community.modularity(G, communities)` (Clauset–Newman–Moore) | how well the graph partitions into densely connected communities |
| `SmallWorldness` ($\sigma$) | Humphries 2008 $\sigma = (C / C_{\text{rand}}) / (L / L_{\text{rand}})$ — clustering via `bct.clustering_coef_bu`, path-length via `bct.efficiency_bin` (and analog), nulls from 1,000 randomized binary graphs | coexistence of high local clustering and short global paths |
| `Assortativity` ($r$) | `nx.degree_assortativity_coefficient(G)` (Newman 2003) | $r > 0$: hubs prefer hubs; $r < 0$: hub-and-spoke |
| `Entropy` ($H$) | Gaussian KDE on the (continuous) `\|W\|` values evaluated on a 100-point grid, then `scipy.stats.entropy(probs, base=2)` | low = structured / specialized; high = uniform / random-like |

The `RNNWeightMatrixHistoryI` callback also stores the recurrent kernel at training start and after every epoch (`model.history.history['RNN_Weight_Matrix']`). The Save cell does not write these snapshots by default — they live in the in-memory `history` object — but they're handy if you want to study the trajectory of the topology during learning.

## `config.txt`

A plain text dump of every variable from the simulation-parameters cell, written once at the top of Save. Use it as the audit trail when a folder's contents look surprising.

## Downstream consumption

Every notebook in `StatisticalAnalysis/` reads only the `*_raw.csv` files (never `metrics.csv` / `metrics_SD.csv`). The expected path is `/content/data/simu/<folder_name>/Task{1,2,3}/<metric>_raw.csv`, with `folder_name` reconstructed from the same parameters as the writer here. See [StatisticalAnalysis 02 — Input data schema](../../StatisticalAnalysis/wiki/02-Input-Data-Schema.md) for the read-side contract.
