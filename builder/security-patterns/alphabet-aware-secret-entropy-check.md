# Validate Secret Strength with Alphabet-Aware Entropy, Not Distinct-Character Counts

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: security-patterns -->
<!-- tags: secrets, entropy, configuration, startup-guard, false-positive, hex, base64, validation -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
A startup guard that rejects secrets with "too few distinct characters" will reject a large share of genuinely random hex secrets. Estimate entropy from the secret's alphabet and length, and reject only low entropy, short repetition periods and heavy single-character dominance.

## Knowledge
A rule such as "reject if fewer than 16 distinct characters" looks sensible but misfires. A random 64-character hex string draws from only 16 symbols, so it often contains fewer than 16 distinct ones. In a measured run of 100 start-ups with a standard `openssl rand -hex 32` secret, the guard refused roughly one in five (about 15-21% across runs). Operators then "fix" the guard by weakening it, or by pasting a weaker secret.

Alphabet-aware approach:

1. Classify the alphabet actually used: hex (16), base64 (64), base64url (64), or general printable (about 95). Use the smallest class that contains every character.
2. Entropy estimate = `length * log2(alphabetSize)` bits. Require a floor, for example 160 bits for a signing key.
3. Reject repetition: find the smallest period `p` such that the string equals its prefix repeated. Reject when `p <= length / 2`.
4. Keep a loose distinct-character floor (for example 8) and a dominance cap (no single character above 25%) as sanity checks. They catch `aaaa...` and `0123456789` repeated.
5. Reject placeholder patterns separately (see the related placeholder entry).

This is a guard against accidents, not proof of randomness. It accepts patterned but non-periodic strings.

## When to Use
- Production-mode configuration validation for signing keys, pepper values, encryption keys and similar secrets.
- Any validator that must accept machine-generated secrets in several encodings.

## Example
Validate the validator by sampling:

```ts
for (const gen of [hex64, base64_48, base64url_32, base64_32]) {
  for (let i = 0; i < 20000; i++) expect(secretEntropyProblem(gen())).toBeNull();
}
expect(secretEntropyProblem("a".repeat(64))).not.toBeNull();      // dominance
expect(secretEntropyProblem("ab".repeat(32))).not.toBeNull();     // period 2
expect(secretEntropyProblem("0123456789".repeat(7))).not.toBeNull();
```

An independent re-sample of 80,000 random draws across four encodings gave zero false rejections with this approach.

## Pitfalls
- A 20-draw test will not expose a rule that fails 15% of the time on average over many draws. Test with thousands of samples, and use random draws, not hand-written examples.
- Do not use a fixed minimum length alone. Length-only checks accept placeholders.
- Document the accepted encodings, otherwise operators guess.
- Entropy estimates assume the secret was generated randomly in its alphabet. They cannot detect a leaked or reused secret.
