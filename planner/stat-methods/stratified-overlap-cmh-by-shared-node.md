# Overlap of Two Edge Sets That Share Regulators: Stratify by Regulator With Cochran-Mantel-Haenszel and Drop Strata Without Both Margins

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: stat-methods -->
<!-- tags: cochran-mantel-haenszel, hypergeometric, stratified-odds-ratio, overlap, edge-sets, non-independence, simpson, statsmodels, universe -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
To ask whether two sets of directed edges (A-flagged and B-flagged) overlap more than expected, a hypergeometric test on the pooled set of edges treats edges as exchangeable. When edges share a source node, stratify by source node and report the Mantel-Haenszel common odds ratio with its confidence interval and test as the headline, with the pooled hypergeometric alongside.

## Knowledge
**Why pooling fails.** Sources with many tested targets tend to have many A edges and many B edges, so pooled overlap is inflated by source size even when, inside each source, A and B are unrelated (a Simpson-type confounding). Edges of one source are also not independent draws.

**Procedure.**
1. **Universe per stratum, read from the data.** For each source node, the universe is the set of targets actually tested in every analysis that defines A and B (intersect per-source tested sets). Do not assume every source was tested against every target.
2. For source `i`: `a` = |A_i and B_i|, `b` = |A_i| - a, `c` = |B_i| - a, `d` = |U_i| - |A_i or B_i|. Table `[[a, b], [c, d]]`.
3. **Drop strata with an empty margin** (no A edges or no B edges for that source). Such a table contributes zero to both the numerator `sum(a*d/n)` and the denominator `sum(b*c/n)` of the MH estimator, so dropping loses nothing and avoids degenerate tables.
4. Fit the stratified table (statsmodels `StratifiedTable`): pooled odds ratio, `oddsratio_pooled_confint()`, `test_null_odds()`.
5. **Report NaN, not an error, when `sum(b*c/n) == 0`** over the kept strata (the MH ratio is undefined).
6. Also report the pooled hypergeometric (`sf(k-1, N, K, n)` over the pooled universe) and the expected overlap `n*K/N`, labelled as the non-stratified reference.
7. Add a per-source hypergeometric table with BH across sources for readers who want to know which sources drive the overlap.

**What worked.** A single function returning both the stratified and the pooled result, with `n_strata` recorded.

**Limits acknowledged in review.** Stratifying by source removes dependence through the source but not through shared targets (a target hit by many sources). The reviewer's request to address this was declined as out of scope, with the cost recorded as "the confidence interval is slightly narrow". State this where the result is shown.

## When to Use
Overlap or enrichment between two edge-level annotations (differential in condition 1, differential in condition 2) in regulatory or interaction networks where each source node contributes many edges.

## Example
```python
tables = []
for src, U in universe.items():                 # per-source tested targets
    A, B = a_edges.get(src, set()), b_edges.get(src, set())
    if not A or not B:
        continue                                 # empty margin: zero contribution
    k = len(A & B)
    tables.append(np.array([[k, len(A) - k], [len(B) - k, len(U) - len(A | B)]], float))
if tables and sum(t[0, 1] * t[1, 0] / t.sum() for t in tables) > 0:
    st = StratifiedTable(tables)
    or_, (lo, hi), p = st.oddsratio_pooled, st.oddsratio_pooled_confint(), st.test_null_odds().pvalue
else:
    or_ = lo = hi = p = float("nan")
```

## Pitfalls
- With few informative strata the MH interval is unreliable; always show `n_strata`.
- A significant common OR below 1 means depletion; wording such as "more than expected" must be conditioned on the direction, not only on significance (a deferred review finding).
- Use the same universe definition for the pooled and the stratified test, or the two will disagree for trivial reasons.
