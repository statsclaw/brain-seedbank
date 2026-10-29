# Differential Analysis Between Networks Induced From One Reference Interactome Is Gene-Set Membership, Not Topology

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: differential-networks -->
<!-- tags: induced-subgraph, differential-network, rewiring, interactome, gene-set, hypergeometric, partner-availability -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
If every context-specific network (tissue, cell type) is the same reference interactome induced on that context's gene set, then any two networks have identical edges among their shared nodes. "Rewiring" analysis then reduces to which nodes are present, and edge-level quantitative differences can only come from added weights.

## Knowledge
**Pattern.** For two contexts A and B built as induced subgraphs of one reference graph:
- Edges among nodes present in both are identical (the design analysis found 40,573 edges among 4,265 shared nodes, none present in only one).
- Topological differences are exactly gene-set membership differences. Do not present them as rewiring evidence.
- A meaningful per-node question is "partner availability": for node v in both contexts, how many of its partners in A are outside B's gene set. Test with a hypergeometric: x = partners of v in A outside B's set, n = partners of v in A, N = nodes of A minus 1, K = nodes of A outside B's set; p = P(X >= x). Evaluate when n >= 3, BH within each direction (A to B, B to A). Assumption: partners are random draws (degree ignored).
- Quantitative difference needs an extra signal: expression-weighted edges, w = min of the two endpoint percentiles. See the entry on rank-within-bin scores for how to report per-node change.
- Draw overlay classes (only in X, in all, in some) against each context's full graph, never against the displayed subsample.

**What worked.** Stating the limitation up front in the methods and UI, and building the node-level tests on membership.

**What required manual intervention.** Two defects surfaced at review: the reported counts for the hypergeometric used the graph node count rather than the gene-set size (conservative, deferred, flagged); and the pair analysis had a mirror (B,A) bug that did not swap the two z-scores and the sizes. Moving the pair analysis into one shared function used by the endpoint and by help-text facts fixed the mirror bug.

**Evidence.** Direct count of differing edges among shared nodes was zero in the analysed pair; a test that the mirror pair swaps z_a/z_b and n went red then green.

## When to Use
Any comparison of context-specific networks derived by subsetting a single reference network, before claiming "differential wiring".

## Example
```python
shared = nodes_A & nodes_B
assert edges(A, among=shared) == edges(B, among=shared)   # true by construction here
# So test availability, not topology:
p = hypergeom_sf(x - 1, N, K, n)   # P(X >= x)
```

## Pitfalls
- If contexts have their own measured interactions, this limitation does not apply.
- Random-partner assumption ignores degree; high-degree nodes will look different for trivial reasons.
- Keep the hypergeometric inputs defined on the same universe (gene set vs graph nodes).
