# Size-Matched Specificity Test for One Set Against a Comparator Set

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: permutation-tests -->
<!-- tags: size-matched, specificity, degree-matched, permutation, percentile, two-sided-p, comparator-set, subsampling -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
To ask whether a small set S behaves differently from a larger comparator set C, draw subsets of C with the size of S, matched on degree, and read where the observed statistic falls among the draws. A high percentile means S is farther than comparable members of C.

## Knowledge
**Pattern.**
1. Statistic d: for example the mean closest distance of a set to a target set.
2. Draw n = 1000 subsets of C, each of size |S|; each member of S is replaced by a member of C from the same degree bin (bins of at least 20 comparator members; use the nearest bin when the member's degree is absent from the pool).
3. percentile = #{d_i <= d} / n.
4. p_low = (1 + #{d_i <= d}) / (n + 1), p_high = (1 + #{d_i >= d}) / (n + 1); two-sided p = min(1, 2 * min(p_low, p_high)).
5. Run the disjoint variant as well (S-only against C-only) so shared members do not sit on both sides.
6. If fewer than 3 members of S reach the target, return NaN for all statistics and no p-value.

**What worked.** Exposing the candidate-selection step (member to its degree bin pool) as its own testable function; using a common seed across comparisons; reporting both the full and the disjoint variants.

**What required manual intervention.** Final review found that when no member could reach the target the test still printed a "significant" result; the NaN rule and its test were added afterwards.

**Evidence.** In the analysis a smaller set had percentile 0.175 (p = 0.35) against the larger set, and 0.37 (p = 0.75) for the disjoint variant, so the smaller set was not distinguishable from comparably sized subsets despite looking "less close" than the larger set at face value (z +1.04 against +3.39 against a size-unmatched null).

## When to Use
Comparing two sets of different size on any distance or score statistic where raw z values are not comparable because they depend on set size.

## Example
```python
def size_matched(S, C, stat, degree, n=1000, rng=None):
    pools = {v: candidates_same_bin(v, C, degree, min_bin=20) for v in S}
    draws = [stat([rng.choice(pools[v]) for v in S]) for _ in range(n)]
    d = stat(S)
    lo = sum(x <= d for x in draws); hi = sum(x >= d for x in draws)
    return lo / n, min(1.0, 2 * min((1 + lo) / (n + 1), (1 + hi) / (n + 1)))
```

## Pitfalls
- Comparing z-scores of sets of different size is not a specificity test.
- Without the disjoint variant, shared members make the sets look more similar than they are.
- No multiple-testing correction is applied across contexts by convention here; state it.
