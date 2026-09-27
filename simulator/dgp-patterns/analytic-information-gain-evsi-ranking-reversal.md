# Prove an information-gain versus EVSI ranking reversal before simulation

<!-- brain-entry -->
<!-- domain: simulator -->
<!-- subdomain: dgp-patterns -->
<!-- tags: EVSI, information gain, decision boundary, analytical design, simulation -->
<!-- contributor: withheld -->
<!-- contributed: 2026-09-27 -->

## Summary

An experiment can resolve more latent uncertainty yet have less decision value when its posterior outcomes remain on one side of the action boundary. Establish the strict ranking algebraically before drawing Monte Carlo samples.

## Knowledge

Construct two independent uncertain response blocks whose action-utility contrasts use the same affine, increasing rule `d(x)=k(x-c)` with `k>0`. Then `E[d(X)|Y]=k(E[X|Y]-c)`, so choosing the alternative by the posterior mean response crossing `c` is justified. Give experiment A a more informative observation channel but place all of its posterior response means above `c`; place experiment B so its positive-probability posterior means fall on both sides of `c`, while its prior mean is on one side. With additive block utilities and independently feasible block actions, A has zero EVSI because its block action stays optimal; B has strictly positive EVSI because its optimal block action changes for some observations. Both blocks have nonzero utility slope `k`. For binary symmetric channels with error probabilities `e_A < e_B < 1/2` and equal binary priors, information gains satisfy `1-h_2(e_A) > 1-h_2(e_B)`, where `h_2` is binary entropy. Enumerate observation probabilities and posterior utility vectors to prove the EVSI ordering before simulation. If action choices are combined across blocks, enumerate the full feasible joint action set and check additivity and feasibility rather than assuming blockwise decisions separate.

For simulation validation, predeclare sample sizes, seeds, tolerances, ranking frequencies, and exact finite-state targets. Compare both an empirical-prior and a fixed-prior estimator: even a candidate with zero population EVSI can have a variable, sometimes negative fixed-prior estimate. Report information in bits and decision value in utility units; a discrete entropy calculation does not establish equivalence to another model's information metric.

## When to Use

Use this construction to test whether an experiment-ranking method responds to decisions rather than only uncertainty reduction, especially in finite-action Bayesian design studies with an analytically tractable toy model.

## Example

Let each block's utility difference between its two options be `k(X-c)`, with `k>0`, and choose the alternative when `E[X|Y]>c`. A's observations sharpen its response estimate, but every posterior mean remains above `c`. B's observations sharpen its estimate less, while positive-probability posterior means straddle `c`. Directly average the best posterior utility over each observation channel, subtract the common prior optimum, and compare those values with each channel's entropy reduction.

## Pitfalls

Making A utility-irrelevant would create a trivial comparison; both responses should enter an actual action contrast. A generic root `d(c)=0` does not justify a posterior-mean threshold: generally `E[d(X)|Y]` differs from `d(E[X|Y])`, and a root alone does not guarantee a single crossing. A strict analytical inequality should precede random simulation, which can otherwise be tuned to a favorable seed. A fixed toy model proves the possibility of a ranking reversal, not its prevalence in other settings. Separate hidden-state regret scoring from the information available to the decision rule.
