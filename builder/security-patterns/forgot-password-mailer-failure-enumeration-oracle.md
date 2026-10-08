# Mailer Failures Can Turn Forgot-Password into an Account-Enumeration Oracle

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: security-patterns -->
<!-- tags: account-enumeration, forgot-password, error-handling, mailer, authentication, transactions, rollback -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
A forgot-password endpoint usually returns the same success status for known and unknown addresses. That protection breaks when email delivery can fail only for known accounts: the known path throws and returns 500, while the unknown path returns 202. Make delivery failure invisible to the caller and roll back the token.

## Knowledge
Flow: for an unknown address the handler skips everything and returns 202. For a known address it creates a reset token, then sends mail. If the mail transport throws (unconfigured transport, provider outage, a stub that always throws), only the known path fails. An attacker compares status codes, or timing, and learns which addresses are registered.

Fix pattern:

1. Wrap token creation and mail send in one transaction. Convert transport errors into a typed delivery error that aborts the transaction, so no usable token is left behind.
2. In the forgot-password handler, catch that delivery error, log it server-side, record an audit event (for example `delivery_failed`), and still return the same 202 body and similar timing as the unknown-address path.
3. For flows where the caller is already authenticated and allowed to learn the outcome (for example an admin sending an invite), return an explicit, fixed 503 and roll back, instead of an unmapped 500.
4. Fail startup in production when the configured mail transport is a known stub. A config that accepts a transport that always throws is the root cause of this class of bug.
5. Add a test that forces the mailer to fail and asserts the response for a known account is byte-identical in status and body to an unknown account.

## When to Use
Any password reset, magic link, invite or email-verification endpoint that intentionally hides whether an address is registered.

## Example
```ts
try {
  await tx(async (db) => {
    const token = await db.resetToken.create(...);   // inside the same transaction
    await mailer.send(user.email, token);            // throws MailDeliveryError
  });
} catch (e) {
  if (!(e instanceof MailDeliveryError)) throw e;
  log.error({ err: e }, "reset delivery failed");    // no address in the log line
  await audit("password_reset.delivery_failed");
}
return reply.code(202).send({ status: "accepted" }); // identical on every path
```

## Pitfalls
- Equalise timing as well as status where feasible, for example by doing equivalent work for unknown addresses or sending asynchronously via a queue.
- Logging the address on failure puts personal data in logs. Log an opaque id.
- Invite endpoints that return 409 for an existing address in any tenant are a separate enumeration oracle, inherent in one-address-one-account designs. Document it and restrict it to privileged actors.
- A queue-based mailer removes the synchronous failure but needs its own dead-letter handling.
