# Two-Stage Production Go-Live of a Classification Engine With Zero-Tolerance Regression Gates

<!-- brain-entry -->
<!-- domain: general -->
<!-- subdomain: deployment-runbooks -->
<!-- tags: go-live, runbook, staged-deployment, regression-gate, classification-engine, rehearsal, owner-approval, rollback -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
Split a release that changes a rule-based classification engine into a code stage (new behaviour switched off) and a data/behaviour stage (switch on, full re-score), and gate each stage on a zero-change regression over the full catalogue. Mark every runbook row as agent-run, agent-with-approval or owner-only so that consent is explicit.

## Knowledge
1. **Stage 1, code live, behaviour off.** Deploy the new code with the new feature switch off. Gate: re-score a pinned reference set and the full catalogue with the old and new code, and require `n_changed == 0`. This proves the deploy alone is behaviour-neutral. Record the count of items compared as well as the changed count, so that an empty comparison cannot pass.
2. **Stage 2, behaviour on.** Freeze code and data, run read-only dry runs (preview re-scores) that report every changed item with its old and new class, and attribute each change to a cause (engine rule change, reference-data change, interaction of both). The human reviewer accepts the attributed change list before anything is written. Only then write, switch on, and re-score; the final regression after the switch must again show zero unexplained changes (explained differences are listed individually).
3. **Row ownership column.** Each runbook row carries an executor class: `agent` (read-only or local), `agent-with-approval` (state-changing, human says yes per row), `owner-only` (signatures, sudo, anything a permission classifier may refuse). Expect the permission layer to refuse some production commands even when approved in chat; plan for the owner to run them and paste the output as evidence.
4. **Rehearse on a disposable cluster.** Run every Stage 2 row verbatim against a copy before production. Rehearsals in this run found a read-only mount that broke the engine's cache write, and measured the real stop time of a long-running inline job so the stop timeout could be set from data rather than a guess. A rehearsal that reproduces the full known-failure set of the later suite (see the separate known-failure-set entry) is a cheap predictor of the production suite result.
5. **Sign-off role, not a person.** Name the sign-off role and the verifiable fields (signer id, verdict, restrictions) in the row, not a specific person; otherwise a legitimate signer needs an ad hoc ruling.
6. **Rollback honesty.** After any unrelated merge that carries a schema migration, the pre-deploy database dump is no longer a clean rollback point. State this in a caveat box.
7. **Variables, not literals.** Baselines (frozen source fingerprint, base commit, reference run id, new run id) must be variables set once and re-read at each gate. Hard-coded baseline names forced every later gate to be mentally re-read after the baseline moved.

## When to Use
Releases where a deterministic classifier, scoring engine or rule table changes and the outputs are consumed downstream (clinical, financial, regulatory), so that any unexplained output change is unacceptable.

## Example
```
Stage 1  deploy code, switch OFF  -> full re-score old vs new: n_compared = N, n_changed = 0   (gate)
Stage 2  freeze                   -> dry runs A/B/C: changed list + cause attribution          (human accepts)
         write + sign reference run -> switch ON -> full re-score: unexplained changes = 0     (gate)
Row format: | id | action | executor: agent / agent-with-approval / owner-only | evidence | pass criterion |
```

## Pitfalls
- A zero-change gate on a subset proves nothing about the rest; compare the whole catalogue and print the compared count.
- Do not plan sub-steps mid-run. A conditional row (for example a reference-data promotion triggered by a warning from a preview build) should be in the runbook from the start.
- Do not edit the runbook in place while executing if a byte-identity check depends on it; record numbered amendments in a separate decision log and fold them in afterwards.
