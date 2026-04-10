# R Data Object Naming Collisions in Packages

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: r-packaging -->
<!-- tags: R, CRAN, data, RData, naming, collision, WARNING -->
<!-- contributor: @YeWang1576 -->
<!-- contributed: 2026-04-10 -->

## Summary

When multiple `.RData` files in an R package's `data/` directory export objects with the same name (e.g., both containing `dat`), R CMD check raises a WARNING. The fix is to rename the internal objects to match their file names.

## Knowledge

R packages load data objects lazily. When two data files export the same symbol, R cannot disambiguate which one `data(dat)` should load. The R CMD check message is:

```
Warning: object 'dat' is created by more than one data call
```

The solution is a one-time rename of the internal objects:

```r
# Fix: rename the object inside each .RData to match the file name
e <- new.env()
load("data/first_dataset.RData", envir = e)
first_dataset <- e$dat
save(first_dataset, file = "data/first_dataset.RData")

e2 <- new.env()
load("data/second_dataset.RData", envir = e2)
second_dataset <- e2$dat
save(second_dataset, file = "data/second_dataset.RData")
```

After renaming, update the `.Rd` documentation to use `\alias{first_dataset}` instead of `\alias{dat}`.

## When to Use

When an R package bundles multiple replication datasets or example datasets that were originally saved with generic variable names like `dat`, `df`, `data`, or `d`.

## Example

```
# Before (causes WARNING):
data/study1.RData  → contains object "dat"
data/study2.RData  → contains object "dat"

# After (clean):
data/study1.RData  → contains object "study1"
data/study2.RData  → contains object "study2"
```

## Pitfalls

- After renaming, any vignette or example code that references the old object name (`dat`) must be updated.
- If users of a pre-existing package relied on `data(study1); dat$column`, the rename is a breaking change. Bump the major version or provide migration notes.
- Use `load(..., envir = new.env())` when inspecting `.RData` files to avoid polluting the global environment.
