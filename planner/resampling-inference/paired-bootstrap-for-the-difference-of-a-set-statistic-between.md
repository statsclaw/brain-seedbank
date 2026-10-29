# Paired Bootstrap for the Difference of a Set Statistic Between Two Conditions With Heavily Overlapping Sets

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: resampling-inference -->
<!-- tags: bootstrap, paired-resampling, confidence-interval, difference, overlapping-sets, union, two-conditions -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
When two conditions evaluate a statistic on largely the same items, a confidence interval for the difference must resample the items jointly from their union. Independent resampling per condition ignores the shared items, inflates the interval, and can change the conclusion.

## Knowledge
**Pattern.** Let condition A and B each have a set of items with per-item values (here: a closest-distance per item). The statistic is the pooled mean converted to a z-score against a fixed null. For the difference z_A - z_B:
1. Take the union U of the two item sets.
2. Draw B bootstrap samples of items from U with replacement (seed derived from a global seed and the pair identity, so the mirror pair (B,A) reproduces the same draws).
3. For each draw, compute both pooled means from the same sampled items (an item absent from a condition contributes nothing there), convert each to a z with that condition's fixed null mean and sd, and take z*_A - z*_B.
4. Report the 2.5 / 97.5 percentiles of z*_A - z*_B.

**What worked.** Deterministic seeding by pair, a mirror test (swapping A and B negates the interval endpoints), and a coverage test on synthetic data with a known true difference.

**What required manual intervention.** The first version resampled each condition independently. Final review showed this ignores that the conditions share most items (two compared conditions shared 337 of 357 items). The user chose the paired scheme. Note the interval still treats the null as fixed, so it covers item sampling only, not null uncertainty; the method text states this.

**Evidence.** For one pair the independent bootstrap gave a difference of -3.07 with 95% CI [-6.30, 0.25], which includes zero. The paired bootstrap on the same data gave [-5.80, -0.30], which excludes zero. Tests that went red then green: deterministic mirror and narrower interval; coverage of the known true difference.

## When to Use
Comparing a set-level statistic (mean distance, mean score, enrichment fraction) between two conditions, tissues or time points where the item sets overlap by more than a small fraction.

## Example
```python
def paired_bootstrap_diff(items_a, items_b, val_a, val_b, null_a, null_b, B, seed):
    U = sorted(set(items_a) | set(items_b))
    rng = make_rng(seed)
    out = []
    for _ in range(B):
        draw = [rng.choice(U) for _ in U]
        ma = mean([val_a[v] for v in draw if v in val_a])
        mb = mean([val_b[v] for v in draw if v in val_b])
        out.append((ma - null_a.mean) / null_a.sd - (mb - null_b.mean) / null_b.sd)
    return percentile(out, 2.5), percentile(out, 97.5)
```

## Pitfalls
- Draws where a condition receives no items need a defined behaviour (skip or NaN), tested explicitly.
- Fixed-null intervals are too narrow when null uncertainty matters; say so.
- Independent resampling is only appropriate when the sets are genuinely disjoint samples.
