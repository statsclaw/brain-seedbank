# Direction-Agreement Test for Edges That Share a Node: Cluster-Robust Ratio Estimator Instead of the Binomial

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: stat-methods -->
<!-- tags: cluster-robust, ratio-estimator, linearised-variance, binomial, proportion, clustering, direction-agreement, p-value-underflow -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
A test of whether the share of shared items changing in the same direction differs from 0.5 is wrong with a plain binomial when the items cluster (edges sharing a regulator). Treat clusters as the sampling unit with a ratio estimator and linearised variance.

## Knowledge
**Setup.** For cluster `i` (a node), `n_i` = number of shared edges, `s_i` = number with the same direction. The overall proportion is the ratio `r = sum(s_i) / sum(n_i)`; clusters differ in size so this is not a mean of per-cluster proportions.

**Variance (linearisation, `k` clusters):**

`Var(r) = k/(k-1) * sum_i (s_i - r*n_i)^2 / (sum_i n_i)^2`

Then `se = sqrt(Var(r))`, the 95 percent interval is `r +/- 1.96*se`, and the p-value is `2 * norm.sf(|r - 0.5| / se)`. Require `k >= 2`; return NaN otherwise. If `se == 0`, return p = 0 when `r != 0.5` and 1 when `r == 0.5`.

**What changed in practice.** The first version used `binomtest(same, n, 0.5)`, which treats every edge as independent, so a few large regulators with coordinated edges made it overconfident. The review fix made the cluster-robust interval and p the headline, kept the binomial in the output only as a labelled reference, and added a test that the direction agreement is cluster-robust by node, observed red before and green after; a test of the generated facts was also red then green.

**Per-cluster table.** Report each cluster's own hypergeometric overlap, BH across clusters, so readers can see which clusters carry the agreement.

## When to Use
Proportion or agreement statistics computed over edges, pairs, probes or reads that belong to a smaller number of units (regulator, gene, donor, batch), with the unit count known and at least a few dozen units.

## Example
```python
g = d.groupby("cluster").same.agg(["sum", "size"])
k = len(g)
s, n = g["sum"].to_numpy(float), g["size"].to_numpy(float)
r = s.sum() / n.sum()
se = np.sqrt(k / (k - 1) * ((s - r * n) ** 2).sum() / n.sum() ** 2)
lo, hi = r - 1.96 * se, r + 1.96 * se
p = 2 * norm.sf(abs(r - 0.5) / se) if se > 0 else (0.0 if r != 0.5 else 1.0)
```

## Pitfalls
- The normal approximation is poor with very few clusters or one dominant cluster; a t reference with `k - 1` degrees of freedom is a more cautious option (not used in the source analysis).
- A normal-approximation p can underflow to exactly 0 for large effects. Never print "p = 0" (the review re-graded this from minor to important because it displays a false statement): format `p == 0` as "< 1e-300" in one shared formatter.
- This handles dependence through the cluster only; edges that share a target remain correlated.
