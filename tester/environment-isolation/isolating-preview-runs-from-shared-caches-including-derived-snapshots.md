# Isolating Preview Runs From Shared Caches, Including Derived Snapshots

<!-- brain-entry -->
<!-- domain: tester -->
<!-- subdomain: environment-isolation -->
<!-- tags: preview-run, dry-run, cache-isolation, bind-mount, mtime, snapshot, checksum, containers -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
A "read-only" preview that runs the production engine in a scratch container can still overwrite production state if the engine writes derived snapshots to a shared cache directory. Isolate every cache location the engine can write, and prove non-interference with checksums before and after.

## Knowledge
1. **Enumerate every writable location** of the engine: the obvious cache directory, but also derived-table snapshots, API response caches and temporary files. Do this by reading the code that loads each table, not by reading the runbook.
2. **Beware mtime-based reuse.** A loader that reuses a snapshot only if `snapshot_mtime >= source_mtime` will re-parse and rewrite the snapshot whenever the preview supplies a fresher source file (for example a newly built scratch table). The rewritten snapshot is then newer than production's source and is picked up by the next production bootstrap; it does not self-heal.
3. **Bind a scratch copy over each location**: seed it from production with a copy that preserves content, compare with `cmp` or checksums, and run the preview against the copy. Mounting the production cache read-only is not a substitute: a read-only mount made the engine fail on its first write.
4. **Checksum before and after.** Take a checksum (and size and mtime) of the shared cache directory before the first preview, after every dry run and after the real run. Any difference stops the window under a written stop rule.
5. **If it happens anyway:** check that there are no active jobs and no worker has bootstrapped since the overwrite, save a copy of the bad file as evidence, remove it, restart the services that may have loaded it, let the next bootstrap regenerate it from the authoritative source, and verify (for example by row count and by presence or absence of a known marker entry).
6. **Reasoning shortcut for later previews:** if the preview's configuration is byte-identical to production's and the sources have not changed, the loader will not rewrite the snapshots; still verify with the checksum rather than rely on the argument.

## When to Use
Previews, dry runs, shadow re-scoring or complement proofs that start the production engine image against a modified reference-data bundle, especially when production runs from the same container image and host paths.

## Example
```
before:  sha256sum shared_cache/*.json > snap_before.sha
run preview with:  -v scratch_cache:/engine/.cache  -v scratch_snapshots:/shared_cache
after:   sha256sum shared_cache/*.json | diff - snap_before.sha   # must be empty
```

## Pitfalls
- Isolating only the "main" cache path while leaving the snapshot directory shared was the original defect.
- Seed copies made by a root-owned container may be unreadable to the operator; create the scratch directory on a path the operator owns.
- A control run (old configuration, isolated cache) is the right way to prove that an observed staleness predates the change under test.
