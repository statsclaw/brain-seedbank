# Enrichment Validity: Match the Background to the Universe Where the Query Can Occur, and Flag Circular Query/Target Pairs

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: enrichment-analysis -->
<!-- tags: enrichment, background, universe, circularity, selection-bias, hypergeometric, benjamini-hochberg, sensitivity-analysis -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
Enrichment of a query set against a feature is only meaningful when the background is the set of items that could have been in the query, and when the query was not selected on the property being tested. Provide a sensitivity background and flag circular combinations rather than silently ranking them.

## Knowledge
**Pattern.**
- **Background.** Test module (cluster) enrichment against the nodes of the network that defines the modules, not against all known genes. An all-genes background makes every in-network group look enriched, because modules exist only in the network. This was caught at final review (test: module background is the module universe). Run a sensitivity analysis on a second background (for example all approved protein-coding genes) and mark rows whose significance changes at FDR 0.05.
- **Circularity flags.** Mark a (query, target) pair as circular when the query set was selected on the target's property: interactors chosen for expression in cell type X tested for enrichment in X; sets that largely overlap X (the design analysis found 74% to 84% overlap between gene sets of related cell types); nodes of a network that was built from the target's genes. Show flagged rows but exclude them from the headline ranking, computed separately for each background.
- **Unmatched items.** Report items outside the universe separately from items with unrecognised identifiers (separate status column and counts), and state the denominator of any coverage gate. If fewer than three query items can reach the target, report no p-value (NaN) rather than a spurious extreme one.
- BH within each query set and background.

**What worked.** Flag-and-show instead of drop: readers see why a row is excluded.

**What required manual intervention.** The module background and the lumping of out-of-universe with unrecognised items were both found only by independent final review; the reviewer-derived tests were red before the fix.

**Evidence.** After correcting the module background, partition-module counts changed; the unmapped-item split and NaN-for-unreachable tests moved from red to green.

## When to Use
Over-representation analysis of gene or protein sets (modules, pathways, cell-type panels) where the query came from a data-driven selection.

## Example
```python
rows = enrich(query, feature_sets, background=module_universe)       # primary
rows_s = enrich(query, feature_sets, background=all_protein_coding)  # sensitivity
rows["differs_by_background"] = (rows.fdr < .05) != (rows_s.fdr < .05)
rows["circular"] = rows.apply(lambda r: selected_on(query, r.feature), axis=1)
headline = rows[~rows.circular].sort_values("fdr")
```

## Pitfalls
- Overlap percentages between gene sets are context-specific; compute them, do not assume.
- Excluding flagged rows from the headline also removes true effects; keep them visible.
- Do not change the denominator of a coverage gate without noting it, because a data update can trip the gate.
