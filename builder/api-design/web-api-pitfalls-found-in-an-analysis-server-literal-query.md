# Web API Pitfalls Found in an Analysis Server: Literal Query Parameters, Non-Finite Floats and Payload Size

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: api-design -->
<!-- tags: fastapi, query-parameters, literal, json-serialisation, infinity, nan, etag, gzip, payload-size, python -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
Three recurring server-side defects in a FastAPI analysis backend: Literal integer query parameters reject the string form, non-finite floats break JSON and return 500, and uncompressed large graph payloads dominate load time.

## Knowledge
**Pattern.**
- A FastAPI query parameter typed Literal[2, 3] (or Literal[1, 2]) rejects the string '2' sent in a URL with a 422. Declare an int with range constraints (ge=2, le=3) or validate against the values present in the data, which still returns 422 for out-of-range values. This was hit twice in one project before the rule was applied by default.
- A tabular statistic can be infinite (an odds ratio with a zero cell) or NaN. Serialising it raises or yields invalid JSON and an HTTP 500. Convert non-finite values to null in one table-to-JSON function; the test asserts null.
- The full graph endpoint was 2,067,982 bytes uncompressed; gzip brought it to 325,031 bytes. An ETag computed as a hash of the response body is correct for help edits, query changes and data swaps, at the cost of computing the body on revalidation; a strong ETag shared between gzip and identity encodings, with no Vary header, is a known gap.
- Joining node attributes by name is safer than by position when two stores use different orders (graph JSON in the graph library's order, parquet in name order).

**What worked.** Tests that hit the real endpoint with a string-valued parameter.

**What required manual intervention.** The Literal rejection and the infinity 500 were found only when real data or a browser request exercised them.

**Evidence.** Red-then-green test for non-finite values (suite 220/220 after); payload sizes above.

## When to Use
FastAPI or similar schema-validating Python servers returning scientific tables to a browser.

## Example
```python
dim: int = Query(2, ge=2, le=3)          # not Literal[2, 3]
def table_json(df): return df.replace([np.inf, -np.inf, np.nan], None).to_dict("records")
```

## Pitfalls
- A weak ETag with Vary on encoding is the safer default when serving both gzip and identity.
- Caching expensive per-pair endpoints server-side was left as a deferred item; measure before shipping.
