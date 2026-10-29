# Edge Length Transform for Expression-Weighted Shortest Paths and Betweenness

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: numerical-stability -->
<!-- tags: igraph, dijkstra, betweenness, edge-length, log-transform, zero-weight, weighted-graph, python -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
To make shortest paths maximise the product of edge weights in (0, 1], use length = -ln(w) + epsilon with a tiny epsilon (1e-6). The epsilon keeps weight-1 edges usable in weighted betweenness, and all-ones weights reproduce hop counts up to rounding.

## Knowledge
**Pattern.** The sum of -ln(w) along a path is -ln of the product of weights, so the shortest path maximises the product. A weight of 1 gives length 0, and igraph's weighted betweenness rejects zero edge lengths. Adding a small constant epsilon (1e-6) avoids the rejection; with all weights equal to 1 the shortest paths equal hop counts (to rounding). A weight of 0 or a missing weight removes the edge rather than giving an infinite length. Keep the new parameters (graph, lengths, weights) keyword-only with no default in shared proximity functions, so existing validated code paths are not reinterpreted.

**What worked.** The epsilon was chosen at plan time and the sanity check (pooled mean distance equals the stored per-item mean) was added as a gate.

**What required manual intervention.** None beyond the plan-time ruling; the per-hop bias of epsilon was judged negligible.

**Evidence.** The pipeline ran Dijkstra distances and subset betweenness on all weighted graphs after the transform; weight-1 edges stayed usable.

## When to Use
Python graph code (igraph, networkx) combining weights in (0, 1] with shortest-path or betweenness routines.

## Example
```python
EPS = 1e-6
length = {e: -math.log(w[e]) + EPS for e in edges if w[e] > 0}   # w == 0 or missing: drop edge
g.es["length"] = [length[e] for e in g.es]
d = g.distances(source=S, target=T, weights="length")
```

## Pitfalls
- Epsilon adds a small per-hop bias that favours fewer hops; keep it tiny and document it.
- Do not use epsilon to rescue weights of 0; remove those edges.
