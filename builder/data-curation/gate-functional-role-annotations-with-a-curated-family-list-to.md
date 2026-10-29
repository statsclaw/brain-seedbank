# Gate Functional Role Annotations With a Curated Family List to Avoid Over-Labelling

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: data-curation -->
<!-- tags: annotation, kinase, receptor, omnipath, hgnc, consensus-score, gating, false-positives -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
Deriving protein roles directly from an interaction resource labels many proteins wrongly. Intersect with a curated gene-family list and require multi-resource consensus for composite annotations.

## Knowledge
**Pattern.**
- Kinase or phosphatase: take the (de)phosphorylating-enzyme annotation from an enzyme-substrate resource and intersect it with the HGNC kinase/phosphatase gene groups, excluding regulatory, anchoring and inhibitor groups. Enzyme-substrate annotation alone flagged 1,385 network proteins as kinases (including obvious non-kinases such as structural and metabolic proteins); the gated rule gave 315 kinases and 73 phosphatases.
- Intercell roles (receptor, ligand, adhesion, transporter): use the composite annotation rows with a consensus score of at least 2 (a configurable threshold). Single-resource annotations doubled the receptor count (4,265 versus 1,820).
- Define role sets with the resource's parent vocabulary (ligand = ligand or cell-surface ligand; adhesion = adhesion, cell adhesion, matrix adhesion; transporter = transporter or ion channel).
- Verify downloaded sources: checksums against the provider's metadata file when present; an unmapped tissue name raises instead of silently dropping; a corrupt sidecar triggers a refetch and an empty body is rejected.

**What worked.** The gate is configuration, not code, and tests use cached files only (downloads disabled in the test config).

**What required manual intervention.** The implausible kinase counts were spotted by looking at real data labels; the unit tests passed with toy fixtures.

**Evidence** is the counts above (1,385 versus 315; 4,265 versus 1,820) plus the red-then-green tests for the corrupt sidecar and empty body.

## When to Use
Building role or family annotation layers over protein or gene networks from aggregated public resources.

## Example
```python
kinase = set(enzyme_substrate_enzymes) & set(hgnc_kinase_groups) - set(hgnc_regulatory_groups)
receptor = {g for g, row in intercell.items() if row.role == "receptor" and row.consensus >= cfg.min_consensus}
```

## Pitfalls
- Symbol drift between resources means some proteins stay unlabeled; report coverage.
- Gating trades recall for precision; state which side was chosen.
