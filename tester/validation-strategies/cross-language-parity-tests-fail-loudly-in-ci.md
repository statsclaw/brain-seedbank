# Cross-Language Parity Tests Must Probe Dependencies and Fail Loudly in CI

<!-- brain-entry -->
<!-- domain: tester -->
<!-- subdomain: validation-strategies -->
<!-- tags: parity-testing, cross-language, subprocess, skip, ci, python, typescript, test-guards -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
A test that shells out to another language runtime to compare outputs (for example a TypeScript suite calling a Python reference implementation) should probe that the dependencies are importable, skip with a clear message locally, and fail loudly when running in CI. Probing for only the executable lets the test fail with an obscure import error, and silent skips in CI hide a lost guarantee.

## Knowledge
Observed sequence: the parity test found a Python executable on PATH, ran it, and failed in CI on a missing third-party package. After a first fix that made it skip, the opposite risk appeared: a skipped test in CI looks green while the parity guarantee is gone.

Robust guard:

1. Build a candidate list of interpreters: an explicit environment variable, a project virtual environment, then `python3`, then `python`; include the Windows launcher names when the suite runs on Windows.
2. For each candidate run a probe with the same environment the test will use (same `PYTHONPATH`, same working directory): `python -c "import the_reference_module"`. The first candidate whose probe exits 0 wins.
3. If none works: when `CI` is set, throw an error naming the missing module and how CI should install it. When `CI` is not set, mark the test skipped with the same message.
4. Make the CI workflow install the reference environment with pinned versions and hash-locked requirements, so the test cannot silently regress to skipping.
5. Verify the guard itself with three manual scenarios: no interpreter, interpreter without dependencies (empty virtual environment), and the full locked environment.

The same pattern fits other self-skipping tests (for example infrastructure validators that need a binary on PATH): such a skip is acceptable only if another CI job covers the same check. Shallow checkouts also hide master-relative diff checks, so those should skip only when the base ref is absent.

## When to Use
Any test that depends on a tool outside its own language runtime and could be skipped because the tool is missing.

## Example
```ts
function findReferencePython(): string | null {
  for (const exe of candidates()) {
    const r = spawnSync(exe, ["-c", "import reference_pkg.cli"], { env: testEnv, encoding: "utf8" });
    if (r.status === 0) return exe;
  }
  return null;
}
const py = findReferencePython();
if (!py && process.env.CI) throw new Error("reference_pkg not importable in CI: install locked requirements");
const maybe = py ? it : it.skip;
```

## Pitfalls
- Probe with the same environment the test uses. A probe in a different environment gives a false "available".
- Count skips in the CI summary and watch for them growing.
- Hash-locked requirements need a matching interpreter minor version. Pin both.
