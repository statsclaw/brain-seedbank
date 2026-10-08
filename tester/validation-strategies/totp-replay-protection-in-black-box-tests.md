# Black-Box TOTP Tests Must Account for Replay Protection Across Suites

<!-- brain-entry -->
<!-- domain: tester -->
<!-- subdomain: validation-strategies -->
<!-- tags: totp, mfa, replay-protection, flaky-tests, test-ordering, integration-tests, ci -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
When a server accepts each TOTP time step only once per user, any earlier test suite that logs in as the same seeded user consumes the current step. The next suite's first login with a freshly computed code then fails intermittently. Tests must compute codes for the next unused step, or retry once on a fresh step.

## Knowledge
Replay protection stores the last accepted time step per user and rejects any code whose step is less than or equal to it. Test infrastructure often seeds MFA users with one fixed secret and logs in with a code computed for "now". Failure mode: an integration suite runs just before an acceptance suite in CI, logs in as the same user within the same 30-second window, and consumes that step. The acceptance suite's first login then returns an invalid-code error. It looks like a flaky test, passes locally when suites run apart, and fails in CI.

Remedies, best first:

1. Maintain a helper `nextTotp(secret)` that remembers the last step it used per user within the process and returns a code for `max(now_step, last_used_step + 1)`. The server's verification window (typically plus or minus one step) must be wide enough to accept the future step; otherwise sleep until the next step boundary.
2. Use distinct seeded users per suite, if the fixture allows.
3. Retry the login once on `invalid_code` after waiting for the next step boundary. Keep the retry in the test helper only, never in the product code.
4. Never relax the replay protection to make tests pass.

Also: the seeding script and the server under test must share the same encryption key for the stored secret, or every TOTP login fails for a different reason.

## When to Use
Any test suite that logs in via TOTP against a real server or database, especially when several suites share seeded accounts, or CI runs unit, integration and acceptance stages back to back.

## Example
```ts
const lastStep = new Map<string, number>();
export function nextTotp(user: string, secret: string, now = Date.now()) {
  const step = Math.max(Math.floor(now / 30000), (lastStep.get(user) ?? 0) + 1);
  lastStep.set(user, step);
  return hotp(secret, step);
}
```
This only helps within one process. Across processes, fall back to the single retry after the next step boundary.

## Pitfalls
- A per-process counter does not help when two processes use the same account at the same time. Give each process its own account.
- Clock skew between runner and server shifts which step is current. Allow for one step of skew.
- Failed-login counters (lockout) can be tripped by the retry. Raise the lockout threshold in the test environment instead of weakening the production default.
