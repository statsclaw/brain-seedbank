# R Package CRAN Compliance Checklist

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: r-packaging -->
<!-- tags: R, CRAN, packaging, compliance, R CMD check, submission -->
<!-- contributor: @YeWang1576 -->
<!-- contributed: 2026-04-10 -->

## Summary

A systematic checklist of common issues that cause R CMD check failures when preparing an R package for CRAN submission. Covers code style mandates, documentation requirements, data packaging rules, and NAMESPACE management.

## Knowledge

### Code Style (ERRORs and WARNINGs)

1. **`cat()` → `message()`**: CRAN policy requires user-facing informational output use `message()` (suppressible) instead of `cat()` (not suppressible). Only use `cat()` inside `print.*` and `summary.*` methods.
2. **`T`/`F` → `TRUE`/`FALSE`**: Bare `T` and `F` are not reserved words — they can be overwritten. CRAN requires the full forms.
3. **`1:n` → `seq_len(n)`**: Using `1:n` fails when `n = 0` (produces `c(1, 0)`). Use `seq_len()` or `seq_along()`.
4. **S3 method signatures must match generics**: `print(x, ...)`, `summary(object, ...)`, `plot(x, ...)`. Using `result` as the first argument will cause check failures.

### DESCRIPTION File

5. **Use `Authors@R`** with `person()` and roles (`"aut"`, `"cre"`) instead of plain `Author`/`Maintainer` fields. Exactly one person must have the `"cre"` role.
6. **Description field**: Must be a proper paragraph (not start with "This package"). Use third person ("Implements..." not "We implement...").
7. **`LazyData: true`**: Required if you have data in `data/`.
8. **`License: MIT + file LICENSE`**: The `LICENSE` file must exist with `YEAR` and `COPYRIGHT HOLDER` fields.

### Data

9. **All objects in `data/*.RData` must be documented** with `.Rd` files.
10. **No duplicate object names** across `.RData` files. If `data/foo.RData` and `data/bar.RData` both contain an object named `dat`, R CMD check issues a WARNING. Rename internal objects to match file names.

### NAMESPACE and Imports

11. **All non-base functions must be imported** via `importFrom()` in NAMESPACE. This includes `stats::var`, `stats::cov`, `stats::qnorm`, etc.
12. **Suppress `no visible binding` NOTEs** for ggplot2 aesthetics using `utils::globalVariables()` in a `zzz.R` file, or use `NULL` variable assignments at the top of plot functions.

### Examples and Vignettes

13. **Wrap slow examples** in `\donttest{}` (not `\dontrun{}`). CRAN runs `\donttest{}` examples but with a generous time limit.
14. **Vignettes with tikz/heavy LaTeX** won't build on CRAN — use standard Rmd with HTML output instead.

## When to Use

When preparing any R package for first-time CRAN submission, or when debugging R CMD check failures on an existing package. Apply this checklist before running `R CMD check --as-cran`.

## Example

```r
# BAD: cat for informational output
my_function <- function(data) {
  cat("Processing", nrow(data), "rows\n")
}

# GOOD: message for informational output
my_function <- function(data) {
  message("Processing ", nrow(data), " rows")
}

# BAD: S3 print method with non-standard first argument
print.MyClass <- function(result, ...) { ... }

# GOOD: S3 print method matching generic
print.MyClass <- function(x, ...) { ... }
```

## Pitfalls

- The HTML validation NOTEs in R CMD check (e.g., `<main> is not recognized!`) are caused by R's built-in HTML tidy checker version mismatch and affect all packages — they are not blocking for CRAN acceptance.
- `\donttest{}` examples ARE run by CRAN (unlike `\dontrun{}`), just with more time. Ensure they actually work.
- When using `LazyData: true`, data objects are available without `data()` calls, but examples should still use `data()` for clarity.
