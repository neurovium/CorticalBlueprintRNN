# 06 — Statistical methods

A single-page reference for every test the pipeline uses, what it assumes, and which `scipy` / `statsmodels` call implements it.

## 1. Shapiro–Wilk normality (implicit, via diagnostics)

Used informally to motivate the nonparametric choice. The diagnostics in [03 — Distribution diagnostics](03-Distribution-Diagnostics.md) show that the per-run distributions are not Gaussian; Shapiro–Wilk on those columns would reject normality for most cells. The pipeline therefore does not assume normality anywhere downstream.

## 2. Kruskal–Wallis $H$ test

One-way omnibus across $k \ge 2$ independent samples. Tests $H_0$: all groups come from the same distribution (against the alternative that at least one stochastically dominates the others).

$$H = \frac{12}{N(N+1)}\sum_{g=1}^{k}\frac{R_g^{2}}{n_g} - 3(N+1)$$

where $R_g$ is the sum of ranks in group $g$, $n_g$ its size, and $N = \sum n_g$. Reference: Kruskal & Wallis 1952.

```python
from scipy.stats import kruskal
H, p = kruskal(group1, group2, group3)
```

Used **per factor**, holding the other factors fixed (see [05 — Effect of Model Variants](05-Effect-of-Model-Variants.md)).

## 3. Mann–Whitney $U$ test (pairwise post-hoc)

Two-sample analog of KW. Computes the rank-based statistic

$$U = n_1 n_2 + \frac{n_1(n_1+1)}{2} - R_1$$

and converts to a $p$-value under $H_0$ of stochastic equality. Reference: Mann & Whitney 1947.

```python
from scipy.stats import mannwhitneyu
U, p = mannwhitneyu(group1, group2, alternative='two-sided')
```

Used only after KW returns $p < 0.05$, on every pair of factor levels within the current fixed context.

## 4. Holm step-down correction

Controls family-wise error rate over the set of pairwise tests within a given factor / fixed context.

```python
from statsmodels.stats.multitest import multipletests
reject, p_adj, _, _ = multipletests(raw_p_values, alpha=0.05, method='holm')
```

The corrected $p$-values are what enter S5/S6/S7 in the paper.

## 5. Rank-based factorial ANOVA

Nonparametric approximation to factorial ANOVA obtained by replacing the response values with their global ranks, then fitting a standard linear model on the ranked data.

For each metric, observations are ranked:

$$
r_i = \operatorname{rank}(x_i)
$$

and the model

$$
r \sim C(W) * C(D) * C(C)
$$

is fit using ordinary least squares, followed by a Type-II ANOVA decomposition.

```python
from statsmodels.formula.api import ols
import statsmodels.api as sm

model = ols(
    'ranked ~ C(w_variant) * C(d_variant) * C(c_variant)',
    data=d
).fit()

anova_table = sm.stats.anova_lm(model, typ=2)
```

Used as a global nonparametric test for main effects and interactions before the per-factor simple-effects analysis.


## 6. Bootstrap 95% confidence interval (percentile method)

For one `(variant, task, metric)` row of the raw CSV — i.e. 20 values — sample with replacement to produce many resampled means, then take the 2.5th and 97.5th percentiles.

```python
import numpy as np
rng = np.random.default_rng(0)
resampled = [rng.choice(x, size=len(x), replace=True).mean()
             for _ in range(10_000)]
lo, hi = np.percentile(resampled, [2.5, 97.5])
```

`6_6_2_confidence_intervals.ipynb` also reports the t-distribution CI
$\bar{x} \pm t_{0.975, n-1}\,\hat{\sigma}/\sqrt{n}$ for comparison; the paper cites the bootstrap CI in Tables 2 and 3.

## 7. Mutual information & Spearman correlation (diagnostics)

Used in `6_6_2_sttc_corr_metric_dist_check.ipynb` only, to map redundancy across the eight metrics:

```python
from sklearn.feature_selection import mutual_info_regression
mi = mutual_info_regression(features, target)

from scipy.stats import spearmanr
rho, p = spearmanr(metric_a, metric_b)
```

Neither feeds into the paper's reported $p$-values; both inform the methodological choice in §IV.E.

## Library versions

The pipeline is intentionally library-light:

```
scipy        (kruskal, mannwhitneyu, friedmanchisquare, spearmanr, stats.t)
statsmodels  (multipletests)
sklearn      (feature_selection.mutual_info_regression, preprocessing.MinMaxScaler/StandardScaler)
numpy        (resampling, percentile)
pandas       (CSV IO, slicing)
matplotlib + seaborn  (plots)
```

Any reasonably recent versions work; the notebooks were developed in Google Colab against the runtime defaults.
