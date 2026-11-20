# Known-Failure-Set Comparison as a Test-Suite Gate

<!-- brain-entry -->
<!-- domain: tester -->
<!-- subdomain: test-gating -->
<!-- tags: test-suite, known-failures, baseline, fixtures, regression-fixtures, exit-code, gate -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
When a suite has pre-existing failures that cannot be fixed within the release, gate on set equality of failing test ids against a recorded baseline list rather than on the process exit code. Pair it with explicit handling of fixtures that pin old engine outputs.

## Knowledge
1. **Record the baseline.** Run the full suite at the current production version and save the exact failing test ids to a file (a recorded set of known failures). Do this in the same environment as the gate (same image, same required-no-skip flag).
2. **Gate on the set, not the code.** The suite exit code is non-zero with any known failure, so it carries no information. Extract failing ids, sort, and `diff` against the expected list. Report four numbers: new failures, fixed failures, total run, skipped.
3. **Forbid silent skips.** Set an environment variable (or equivalent) that turns skips of critical identity or parity tests into failures, otherwise a missing dependency passes the gate by skipping.
4. **Classify expected differences at the new version.** Fixtures that pin the old engine's outputs will fail against the new engine (here, a larger block of extra failures at the new engine). Save that second list too, and run the suite at the new version in a rehearsal to confirm the list is exactly reproduced.
5. **Regenerate fixtures only after the switch.** Regenerating them earlier would break the suite at the old version during the earlier stage. After the switch, regenerate, check byte-identity against the reviewed fixture branch, fast-forward the shared branch to it, and expect the final failing set to equal the original baseline minus the fixtures that were regenerated, plus any listed environment-dependent test.
6. **Explain each id that is neither in the baseline nor regenerated.** Typical causes: a test reading a live registry whose version moved in the same release, or a closed list in a test that a later unrelated merge extended. Record a ruling per id (fix the test on the fixture branch, or list it as expected).
7. **Final result format.** "<N> tests, <k> failures, all in the expected list, 0 errors, <s> skipped (none in the critical set)".

## When to Use
Release gates for repositories with long-lived known failures (environment-dependent, data-dependent, or pinned-output fixtures), where fixing them is out of scope but regressions must still be caught.

## Example
```
baseline_failures.txt  (sorted ids)           # recorded at old version
expected_at_new.txt    = baseline + engine-tracking fixture ids
run suite -> failing_ids.txt (sorted)
diff failing_ids.txt expected_at_new.txt       # empty = pass
```

## Pitfalls
- Parametrised or subtest failures: compare at the level of the test method id, and record the subtest count separately, otherwise the lists are unstable.
- A baseline taken without the no-skip flag hides tests that fail only when they actually run.
- Stale baselines: refresh the file deliberately and review the diff; never auto-update it from the run being gated.
