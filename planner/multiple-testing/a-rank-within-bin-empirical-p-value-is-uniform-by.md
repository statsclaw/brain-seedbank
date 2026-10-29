# A Rank-Within-Bin Empirical p-Value Is Uniform by Construction, So FDR Control Can Never Reject

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: multiple-testing -->
<!-- tags: empirical-p-value, rank, uniform, benjamini-hochberg, fdr, outlier-score, standardised-score, binning -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
If a per-item score is standardised or ranked within strata of the same data, an empirical p-value taken from its rank within the stratum is uniformly distributed regardless of any real effect. Benjamini-Hochberg on those p-values is structurally null. Present the score as a descriptive outlier ranking, not as a test.

## Knowledge
**Pattern.** Suppose each item has a score D (for example the mean change of a quantity over its shared edges) and items are grouped into bins of similar size. Compute z = (D - bin mean) / bin sd, and an empirical p from the rank of D inside its bin. By construction the ranks inside a bin are a permutation of 1..n, so the p-values are uniform whatever the truth is. The reference distribution is the data themselves, so there is nothing for FDR control to detect. Rank-based p-values also have a finite resolution of 1 / n_bin.

**What worked.** Relabelling the quantity as a "descriptive outlier score", ranking by |z|, and stating in the methods that no multiple-testing procedure is applied. A test named for the property ("score is uniform by construction") pinned the new behaviour: the report returns z with no p column.

**What required manual intervention.** The first diagnosis was wrong. The initial ruling said the FDR step "rarely reaches 0.05" because bins of 100-200 items give a minimum p near 0.005-0.01, and kept the method. The final review showed the issue was structural, not a resolution limit, so the earlier ruling was explicitly superseded and the p/FDR columns were removed (user chose z as an outlier score). Lesson: when a test almost never rejects, check whether it can reject at all before explaining it as low power or coarse resolution.

**Evidence.** Across about 2,400 items, zero were significant at FDR 0.05 in every run; the fix replaced the p/FDR output with an outlier score and added a test that the report contains z without p.

## When to Use
Any "which items changed unusually" screen where the comparison distribution is made of the same items (within-bin standardisation, within-bin rank, quantile-normalised scores).

## Example
```python
def outlier_scores(D, bin_id):
    z = {}
    for b in set(bin_id):
        v = [D[i] for i in range(len(D)) if bin_id[i] == b]
        m, s = mean(v), sd(v)
        for i in range(len(D)):
            if bin_id[i] == b:
                z[i] = None if len(v) < 2 or s == 0 else (D[i] - m) / s   # bins of one or sd 0: no z
    return z   # rank by abs(z); do NOT convert to p and run BH
```

## Pitfalls
- A real test needs a reference distribution independent of the items being tested (permutation of labels, an external null).
- If you must keep a p-value, calibrate it against an independent null; do not feed a bin-rank p into BH.
- Bins of one item or with zero spread give no z; keep them as missing, not as zero.
