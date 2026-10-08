# Escape LIKE Wildcards Yourself When Using ORM "contains" Filters

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: data-access-patterns -->
<!-- tags: prisma, orm, sql, like, wildcard, escaping, search, postgres, typescript -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
In Prisma 7 the `contains`, `startsWith` and `endsWith` string filters do not escape the SQL LIKE metacharacters `%`, `_` and `\`. A user-supplied search term is therefore interpreted as a pattern, not as literal text. Escape these characters before passing the term to the filter.

## Knowledge
The ORM docs and common assumption say a `contains` filter performs a "literal substring search". On PostgreSQL the term is bound as a parameter and wrapped as `%term%`. The term's own `%` and `_` are not escaped. Consequences:

- Searching for `%` or `_` matches every row (or every row with at least one character).
- A term like `a_c` matches `abc`, so "literal" results are wrong.
- Wildcard-heavy terms cost more to evaluate, which gives a cheap way to degrade the database.

Fix: escape the three metacharacters, backslash first, before building the filter:

```ts
// Order matters: escape the escape character first.
export function escapeLike(term: string): string {
  return term.replace(/[\\%_]/g, (ch) => "\\" + ch);
}

const where = { name: { contains: escapeLike(userTerm), mode: "insensitive" } };
```

On PostgreSQL the default LIKE escape character is the backslash, so no `ESCAPE` clause is needed. Check the dialect default when using another database.

## When to Use
- Any user-facing search or filter field built on `contains`, `startsWith` or `endsWith` against SQL databases.
- Any place that interpolates user text into a LIKE or ILIKE pattern, ORM or not.

## Example
Test the behaviour on a real database, not a mock, since mocks hide it:

```ts
await create({ name: "100% done" }, { name: "100 percent" });
const hits = await findMany({ where: { name: { contains: escapeLike("100%") } } });
expect(hits.map((h) => h.name)).toEqual(["100% done"]);   // without escaping: both rows
```

## Pitfalls
- Behaviour was found on a real PostgreSQL instance during integration testing. A spec written from the docs had assumed escaping was automatic. Re-verify after major ORM upgrades, since this could change.
- Escape only the user term, not the surrounding pattern characters you add yourself.
- Case-insensitive mode and trigram indexes do not change the escaping rules.
- Escaping is for correctness and cost control. It is not an injection fix (the value is already parameterised).
