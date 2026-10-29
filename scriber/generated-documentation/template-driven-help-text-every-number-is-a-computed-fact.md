# Template-Driven Help Text: Every Number Is a Computed Fact and Wording Is Conditional on Significance

<!-- brain-entry -->
<!-- domain: scriber -->
<!-- subdomain: generated-documentation -->
<!-- tags: templated-help, computed-facts, conditional-wording, documentation, p-value-formatting, case-studies, validation -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
In an analysis application, write user-facing help and case-study text as templates filled from facts computed on the current data bundle, with wording conditional on significance flags. Hard-coded numbers, positional claims (such as "the top three hubs are...") and unconditional conclusions go stale or become false when the data change.

## Knowledge
**Pattern.**
- Facts are computed by the same functions that serve the API (one shared function, so the page and the help agree).
- Every number in the text is a template slot filled from a fact; wording that states a result is conditional (for example "closer than expected" only when the BH-adjusted flag is set, otherwise a neutral sentence).
- Never reference positional facts (top-k labels, "the first row"); use named facts ("hub A has degree N") or a computed list from the flagged set.
- Titles of case studies are neutral, not conclusions.
- Render on each request (cheap), so wording edits appear without a restart.
- A checker validates the content: every referenced fact exists, every preset key is a known view key, templates compile; it runs as a test and as a script with zero errors required.

**What worked.** A content sweep to conditional wording, facts computed from BH-significant lists, a checker over every preset key.

**What required manual intervention.** Final review found: hard-coded result claims that contradicted the data after reanalysis; positional hub facts; an unconditional takeaway comparing two groups; uncorrected p-values quoted for a family of tests; inconsistent p/FDR formatting. Each was fixed with a test going red then green (p-value format test, family-correction test, named-hub facts test, preset-key validation test).

**Evidence.** Rewriting the content after the data re-run changed several previously stated conclusions, which is why the template route was needed; a computed-fact failure at startup was noted as a risk (wrap in a handled error so the help endpoint fails alone, not the whole app).

## When to Use
Any documentation or tutorial layer that quotes results of a pipeline that may be re-run with new data or settings.

## Example
```python
facts = compute_facts(bundle)                   # shared with the API
text = render("{{ group }} is closer to the target than random "
              "(z = {{ z|fmt }}, FDR {{ fdr|pfmt }}).") if facts.sig["group"] \
       else render("{{ group }} is not distinguishable from random "
                   "(z = {{ z|fmt }}, FDR {{ fdr|pfmt }}).")
```

## Pitfalls
- A wrapped fact-computation failure should degrade the help page only.
- Keep one p-value formatter; mixed formats look like different tests.
- Template conditionals must be exercised by tests on both branches.
