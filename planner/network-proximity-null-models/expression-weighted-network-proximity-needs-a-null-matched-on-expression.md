# Expression-Weighted Network Proximity Needs a Null Matched on Expression Nested Inside Degree Bins

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: network-proximity-null-models -->
<!-- tags: network-proximity, weighted-graph, null-model, degree-matching, expression-matching, stratification, z-score, interactome -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
When edge weights come from node-level expression, a degree-only random-set null makes well-expressed node sets look closer to a target set than chance. The null must also be matched on expression, and the expression strata must be nested inside the degree bins, not crossed with them in a way that coarsens the degree resolution.

## Knowledge
**Pattern.** Network proximity of a node set S to a target set T is the mean, over S, of the closest shortest-path distance to T. The z-score compares it with sets of the same size drawn at random. With weighted edges (edge length = -ln(w) + tiny constant, w = a node-level expression percentile), distance is driven by expression: a set of highly expressed nodes has short edges everywhere. A null that matches only degree therefore produces a spurious "closer than random" z for any set that is more expressed than average.

Recipe for the null:
1. Bin all candidate nodes by degree (consecutive degrees pooled so every bin has at least about 100 nodes; a short last bin merges into the previous one).
2. Inside each degree bin, split nodes into about five near-equal expression strata (ties share a stratum; missing expression ranks lowest).
3. Replace each node of S by a random node from the same degree bin and the same expression stratum.
4. Also report the z under the degree-only null as a separate column, next to the headline z.

**What worked.** The nested design (expression strata inside degree bins) kept the degree resolution of the unweighted analysis and removed the expression artefact. Reporting the degree-only z alongside the headline z made the size of the artefact visible to readers.

**What required manual intervention.** The first implementation pooled degree bins within expression quintiles. Pooling made the degree bins coarser, which itself manufactured signal, so the "fix" for one confound introduced another. Only an independent review that re-ran the analysis at matched degree resolution caught it. The first test suite passed because it only checked that the null ran, not that its degree resolution matched the unweighted null.

**Evidence.**
- Under a degree-only null, the weighted z of one strongly expressing cell population moved from about +2.6 (unweighted) to about -4.8 (weighted); the node sets of interest had mean expression percentile around 0.70 and 0.56 versus 0.47 for other nodes.
- After matching on degree and expression with coarsened degree bins, the same population read about -2.1; at matched degree resolution (nested strata) it read about -0.6, and pooled z values for most populations went to about 0.
- A regression test (a null built from nested bins keeps the degree resolution; missing expression goes to the lowest stratum; one expression bin reduces exactly to the degree-only null) failed before the fix and passed after.

## When to Use
Any proximity, closeness or diffusion statistic computed on a graph whose edge weights are derived from a node attribute (expression, abundance, confidence of the node's measurement), compared against random sets.

## Example
```python
def matched_null_candidates(nodes, degree, attr, min_bin=100, n_strata=5):
    bins = degree_bins(degree, min_bin)            # pooled, >= min_bin nodes, short last bin merged
    pool = {}
    for b in bins:
        order = rank_with_ties_together(attr, b)   # missing attr -> lowest rank
        for stratum in split_equal(order, n_strata):
            for v in stratum:
                pool[v] = stratum                   # candidates share degree bin AND stratum
    return pool

def draw(S, pool, rng):
    return [rng.choice(pool[v]) for v in S]
```

## Pitfalls
- Do not build strata first and then pool degree inside them: that coarsens degree.
- Missing attribute values must be placed deliberately (lowest rank here) and tested; silently dropping them changes bin sizes.
- Ties in the attribute must stay together, otherwise the strata depend on arbitrary ordering.
- The difference between a weighted and an unweighted z mixes the graph change, the null change and the nodes that remain evaluable; label it as such and do not read it as an effect size.
- If the graph was assembled around the query set, the set is better embedded by construction; interpret proximity accordingly.
