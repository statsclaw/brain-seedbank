# Injective Symbol-to-Project-ID Canonicalisation With Collision Logging When Merging Gene-Symbol Data Into an ID-Keyed Graph

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: python-patterns -->
<!-- tags: gene-symbols, identifier-mapping, injective, collision, hgnc, alias, qc-log, join-key, graph-integration -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
When a third-party table keyed by gene symbol is joined to a network keyed by project identifiers, map each symbol to a canonical name with a rule that cannot send two different input symbols to the same output name. Symbols that cannot be mapped keep their own symbol, and every mapping failure is logged by category.

## Knowledge
**The problem.** Symbols map to the project universe by exact match, then by the approved-symbol via previous symbol or unique alias. Several provider symbols can map to one project id (an old symbol and its replacement are both present). A naive `symbol -> id` dictionary then merges two distinct provider rows into one node, which silently changes counts, creates duplicated (regulator, target) pairs and corrupts per-regulator universes.

**Rule (injective by construction).**
1. Map every distinct symbol; each gets an id or none, plus a status (exact, approved symbol, previous symbol, alias, unmapped, ambiguous).
2. For each project id, choose one owner symbol among the symbols that map to it: the symbol equal to the id if present, else the first in sorted order (deterministic).
3. The owner's canonical name is the id. Every other symbol keeps its own symbol and gets status `collision`; unmapped symbols keep their own symbol too.
4. Use one canonical key for all downstream joins (API, tables, universes); apply the same map to every table and also to the per-regulator tested-target sets, so the stratified tests still see consistent universes.
5. Write one QC log entry per status category with its count and the first few hundred symbols, including a `collision` category. Symbols that stay unmapped remain in the analysis layer but are not joined to the network.

**What worked.** Reusing the mapping code that an earlier data source already used (extracted into one shared helper) so all conditions follow the same rules; the collision category in the QC log makes any merge visible.

**Known weakness (deferred in review).** Ambiguous aliases (one alias, several approved symbols) were logged under "unrecognised", which hides how many are ambiguous. Give ambiguity its own status.

## When to Use
Any pipeline that integrates symbol-keyed omics tables (differential results, regulons, pathway sets) into an identifier-keyed graph or database.

## Example
```python
syms = sorted(symbols)
ids, status = map_symbols(syms, universe, symbol_map, approved)   # shared helper
owner = {}
for s, i in zip(syms, ids):
    if i is not None and (i not in owner or s == i):
        owner[i] = s
names, st = {}, {}
for s, i, k in zip(syms, ids, status):
    if i is not None and owner[i] == s:
        names[s], st[s] = i, k
    else:
        names[s], st[s] = s, ("collision" if i is not None else k)
```

## Pitfalls
- The id and a collided alias may both look valid downstream; show the status in the QC log, not just in code.
- Sorting for the owner choice is arbitrary but must be deterministic across runs.
- A renamed alias shows under its approved symbol in the interface, which users may not expect; document it.
