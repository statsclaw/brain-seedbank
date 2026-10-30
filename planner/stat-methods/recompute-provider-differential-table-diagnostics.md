# Recomputing a Provider's Differential Table: Diagnostics That Separate "Nominal p Was Used" From "A Different Model", and Why a Hypothesised Selection Filter Needs Its Own Test

<!-- brain-entry -->
<!-- domain: planner -->
<!-- subdomain: stat-methods -->
<!-- tags: limma, benjamini-hochberg, fdr, reproducibility, differential-analysis, nominal-p, selection-filter, large-scale-testing, moderated-t -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
When a data provider ships a differential table filtered at nominal p < 0.05 and you recompute it with limma plus Benjamini-Hochberg over every tested feature, zero hits at FDR 0.05 does not mean your model differs from theirs. A short set of agreement diagnostics on the provided rows shows whether you reproduced their model, and a separate test is needed before explaining why their table is smaller than your nominal set.

## Knowledge
**Pattern.** Setting: millions of per-feature scores (here, an edge score per sample for each regulator-target pair), 6 samples versus 6 samples, a provider table of "significant" features with no multiple-testing correction. Recompute: `lmFit` and `eBayes` with design `~ group`, coefficient = group difference, then BH across all tested features.

Diagnostics to compute on the provided rows joined to your recomputed rows (all cheap, all stored with the run so a reader can see them):
1. **Effect agreement.** Maximum absolute difference in logFC between provided and recomputed. A value near float precision (here 2.6e-7) means the same contrast and the same data.
2. **p-value agreement.** Spearman correlation of provided p versus recomputed p (here 0.99). High rank agreement with imperfect equality is expected from a different but equivalent variance moderation or rounding; it rules out a different model.
3. **Coverage of the provided set.** Share of provided features whose recomputed nominal p is below 0.05. Near 1 means "the provider used nominal p".
4. **Reverse coverage.** Provided count divided by recomputed nominal count. Here the provided table held fewer than half of the recomputed nominal features, so nominal p alone does not define their set.
5. **Observed versus expected nominal count.** Nominal hits (538,282) against 0.05 times the number tested (285,043). The excess is about 1.9-fold, which looks like signal, yet the lowest BH-adjusted p was 0.53 and zero features reached FDR 0.05. With millions of dependent features and 6 versus 6 samples there is excess at nominal level but no power at FDR level. Whether dependence between features that share a regulator explains the lack of power was not tested here; state it as a possibility only.

**Test a hypothesised selection filter before documenting it.** The provided set was larger in effect size than the recomputed nominal set, so a minimum |logFC| filter (about 0.56, the smallest |logFC| in the provided table) was suspected. Computing it refuted the idea: 537,654 of the 538,282 recomputed nominal features already passed that threshold, so it could not explain why the provided table was a minority. The final description records the provided table as "a subset of the nominal set selected by a rule that is not stated", with the two diagnostics (share of provided in nominal, minimum |logFC| in provided) shown. An unsupported statement that the missing features were "mostly expected by chance" was also removed because nobody had computed it.

**Policy that held up.** Keep the pre-specified FDR 0.05 for the recomputed version; do not loosen the threshold until something is significant. Show the provided table separately and label it nominal. Make the empty result an explained state in the interface (what was tested, lowest FDR, nominal count against expected), not a blank view.

**Constant features.** limma's moderated variance gives t = 0 and p = 1 to a feature with no variance across all 12 samples. Count it in the BH denominator as the provider's method does; exclude only NaN p (no data). A planning assumption that constants would be NaN was overturned by checking the real output and the tests were renamed to the actual behaviour (BH matches scipy's `false_discovery_control` on the same vector).

**What required manual intervention.** The independent final reviewer proposed the |logFC| filter as the explanation; running the numbers disproved it. The rule is to compute a premise before writing it into the methods.

## When to Use
Reanalysing public or collaborator differential tables from high-throughput data (expression, edge scores, methylation, proteomics) where the provider's inclusion rule is not documented, and you want a corrected version next to theirs.

## Example
```python
def provider_diagnostics(recomputed, provided, fdr=0.05):
    ok = recomputed[recomputed.P.notna()]                 # NaN p = no data; constants have p = 1 and stay
    j = provided.merge(ok[["a", "b", "logFC", "P"]], on=["a", "b"], suffixes=("_prov", "_rec"))
    return {
        "n_tested": len(ok),
        "n_fdr": int((ok.adj_P < fdr).sum()), "min_fdr": float(ok.adj_P.min()),
        "n_nominal": int((ok.P < 0.05).sum()), "expected_nominal": 0.05 * len(ok),
        "logFC_max_abs_diff": float((j.logFC_prov - j.logFC_rec).abs().max()),
        "p_spearman": float(spearmanr(j.P_prov, j.P_rec).statistic),
        "provided_in_nominal": float((j.P_rec < 0.05).mean()),
        "provided_min_abs_logFC": float(provided.logFC.abs().min()),
    }
```

## Pitfalls
- Do not read "zero FDR hits" as a failed reproduction until diagnostics 1 and 2 are checked.
- A provider subset can follow a rule you cannot recover (random subsampling, a different statistic, an effect cutoff on another scale). Document it as unstated rather than inventing a rule.
- Do not drop NaN and constant features inconsistently: the BH denominator changes and the adjusted p values move.
- Python and R versions of limma and the R package versions belong in the run manifest, since moderated p values depend on them.
