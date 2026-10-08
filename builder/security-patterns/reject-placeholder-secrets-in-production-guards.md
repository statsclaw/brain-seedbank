# Reject Placeholder Secrets Explicitly in Production Guards

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: security-patterns -->
<!-- tags: secrets, env-example, configuration, placeholder, production-guard, startup-validation -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
Example environment files usually carry descriptive placeholder values that are long enough to pass a minimum-length check. If the production guard checks only length, copying the example file to production starts the service with a publicly known secret. Detect placeholder patterns and refuse them in every environment.

## Knowledge
A real gap found during a gap review: the example environment file held a placeholder of 56 characters for a signing secret. The production guard required at least 32 characters, so the placeholder passed. Nothing in the tests parsed the example file, so nobody noticed.

Mitigations, in order of value:

1. Reject placeholder-looking values in all environments (not just production), so developers meet the failure early. Patterns to reject:
   - angle brackets or braces (`<your-secret>`, `${SECRET}`)
   - whitespace inside the value
   - words such as `changeme`, `replace`, `example`, `secret`, `your-`, `placeholder`, `todo`
   - instruction text such as `openssl rand ...`
2. Add a test that loads the example environment file, takes each secret-valued entry, and asserts the guard rejects it. This makes the example file and the guard drift-proof.
3. Combine with an entropy check (see the related entropy entry). Keep placeholder detection separate so error messages say "this is a placeholder", which is more actionable than "too little entropy".
4. Make the default environment production-like (`NODE_ENV` unset means production), so a missing variable fails closed.

## When to Use
Any service whose configuration is validated at startup and whose repository ships an example or template environment file.

## Example
```ts
const PLACEHOLDER = /[<>{}\s]|changeme|replace|example|your[-_]|openssl\s+rand|placeholder|todo/i;
function placeholderProblem(v: string): string | null {
  return PLACEHOLDER.test(v) ? "looks like a placeholder; generate a real secret" : null;
}

// test
for (const [k, v] of parseEnvFile(".env.example")) {
  if (isSecretKey(k)) expect(() => loadConfig({ ...base, [k]: v, NODE_ENV: "production" })).toThrow();
}
```

## Pitfalls
- A deny-list is a heuristic. It stops accidents, not attackers.
- Case-insensitive word matching can reject a random secret that happens to contain a short word. Prefer longer tokens or word boundaries, and test with sampled random secrets.
- Do not log the rejected value. Log the key name and the reason.
