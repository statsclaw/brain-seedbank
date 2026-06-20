### MNL-Limit Sanity Check for Nested Logit Second-Derivative Formulas

<!-- brain-entry -->
<!-- domain: numerical-methods -->
<!-- subdomain: discrete-choice -->
<!-- tags: nested-logit, multinomial-logit, hessian, limit-check, algebra-verification, second-derivatives, derivation-validation -->
<!-- contributor: @fpcordeiro -->
<!-- contributed: 2026-06-19 -->

## Summary

When deriving analytical second derivatives (Hessian) for a Nested Logit model, verify
every block by substituting lambda = 1, which collapses the NL to a standard Multinomial
Logit. The MNL Hessian has well-known closed-form entries, so this limit check catches
algebra errors analytically — before writing any code and before running any numeric
oracle.

## Knowledge

The Nested Logit model reduces to the Multinomial Logit at lambda = 1 for all nests
because the within-nest conditional softmax and the between-nest marginal softmax become
one flat softmax. This means every formula derived for the NL Hessian must satisfy the
following MNL limit identities:

**V-space Hessian (Hessian of -log P w.r.t. the utility vector):**

At lambda = 1, the diagonal entry for alternative a must equal:
```
Q[a, a] = P_a * (1 - P_a)
```
and the off-diagonal entry for alternatives a != b must equal:
```
Q[a, b] = -P_a * P_b
```

This is the standard result for the MNL information matrix contribution.

**Beta-beta block:**

At lambda = 1, the per-individual contribution must equal:
```
H_bb = w_i * (X^T diag(P_i) X - (X^T P_i)(X^T P_i)^T)
```

**Lambda-related blocks (lambda-beta, lambda-delta, lambda-lambda):**

At lambda = 1 for all nests, these blocks should produce values consistent with
an MNL model that has no lambda parameters. The lambda-diagonal entry at lambda = 1:
```
H_{lambda, lambda} = w_i * P_k * (1 - P_k) * T_k^2 + (P_k * VarV_k) / 1^3
```
reduces to the information contribution of a log-inclusive-value coefficient, which can
be cross-checked against a simpler separate derivation.

**How to apply the check:**

1. Derive the NL formula symbolically (paper or spec document).
2. Substitute lambda_k = 1 for all nests k in the symbolic expression.
3. Confirm the result equals the known MNL formula for the same block.
4. If they differ, the algebra is wrong — identify which step introduced the discrepancy
   before proceeding to implementation.

**What this check caught in practice:**

The V-space Hessian Q[a,b] formula initially had an extra factor of `P(b|k)` on the
diagonal term `1_{a=b}/lambda_k`. The correct formula is:

```
Q[a, b] = P_ia * { 1_{k(a)==k(b)} * P(b|k(a)) * [1 + (1_{a=b} - P(a|k(a))) / lambda_{k(a)}]
                   - P_ib }
         - correction_term_for_chosen_nest
```

At lambda = 1, the diagonal entry a = b should yield:
```
Q[a, a] = P_ia * { P(a|k) * [1 + (1 - P(a|k)) / 1] - P_ia } - 0
         = P_ia * { P(a|k) * (2 - P(a|k)) - P_ia }
```

With P_ia = P_k * P(a|k) and at lambda = 1 (so P_k = 1 for a flat model), P_ia = P(a|k).
Substituting: P_a * P_a * (2 - P_a) - P_a^2 = P_a^2 (which is NOT P_a(1-P_a)) — this
reveals the bug. The correct formula must give P_a(1-P_a) at lambda = 1, which verifies
the fix.

A sign-convention error in the beta-beta block (Terms A/B/C were derived for the
positive log-likelihood but the implementation needs the negated log-likelihood) is also
caught by checking whether the resulting matrix is positive semidefinite at the MNL
limit, rather than negative semidefinite.

## When to Use

- Deriving any analytical Hessian for a model that generalizes a simpler model with a
  known Hessian (e.g. NL → MNL, correlated probit → independent probit, mixed logit
  at zero variance → conditional logit).
- Writing a spec or mathematical document for a second-derivative formula before handing
  it to an implementer.
- Reviewing a derivation document: the limit check is a fast manual audit that does not
  require running any code.
- When a numeric oracle is available but its accuracy degrades in extreme parameter
  regions (e.g. near-zero lambda), the limit check validates the analytical formula in
  a regime where it can be verified by hand.

## Example

Generic: deriving the Hessian of the (negated) log-likelihood for a two-level model
where parameter `s` is a scale or dissimilarity parameter, and `s = 1` reduces the model
to a flat baseline.

```
# Verify the V-space Hessian diagonal at s = 1
# NL formula: Q[a,a] = P_a * { P(a|k) * [1 + (1 - P(a|k)) / s] - P_a } - correction
# At s = 1, P(a|k) = P_a (flat model): Q[a,a] = P_a * (P_a * 2 - P_a^2 - P_a) + ...
#   -> must simplify to P_a * (1 - P_a)
# If it does not, the formula has an error
```

Concrete check in R (after implementing the analytical Hessian alongside a numeric oracle):

```r
# Check MNL limit: set all nesting parameters to 1
theta_mnl <- c(beta_values, rep(1, n_lambda), asc_values)
H_analytical <- analytical_hessian_fn(theta_mnl, ...)
H_numeric    <- numeric_hessian_fn(theta_mnl, ...)
# This checks the implementation, but the symbolic check was done first on paper
max(abs(H_analytical - H_numeric))  # should be near machine epsilon at lambda=1
```

## Pitfalls

- The limit check validates the **formula**, not the **implementation** — a correctly
  derived formula can still be implemented wrongly. The MNL limit spot-check at lambda = 1
  in the running code (comparing analytical vs numeric oracle) is a separate step.
- Some generalizations have non-smooth limits (e.g. lambda -> 0) or limits that require
  a separate derivation. The lambda = 1 direction for NL is well-behaved and closed-form.
- Sign conventions matter: "Hessian of the log-likelihood" (negative semidefinite at MLE)
  vs "Hessian of the negated log-likelihood" (positive semidefinite at MLE). The MNL
  reference formula must use the same convention as the formula being checked.
- The limit check cannot catch errors that cancel out at the limit but diverge elsewhere
  (e.g. a term that is O(1 - lambda) and therefore zero at lambda = 1 but wrong for
  lambda != 1). Complement the limit check with at least one non-degenerate numerical
  comparison (lambda = 0.7 or similar).

---
