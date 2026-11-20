# Disk Headroom Pre-Check Before Long Test Suites and Exports in Containerised Hosts

<!-- brain-entry -->
<!-- domain: tester -->
<!-- subdomain: environment-preflight -->
<!-- tags: disk-space, headroom, docker, postgres, test-suite, preflight, idempotent-recovery, git-archive -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
Before a multi-hour test suite or a tree export on a host whose root volume also holds the container data root and the production database, check free space, set a minimum, and export to a data volume. A full volume turns into failed database writes in unrelated production workers.

## Knowledge
1. **Add a "headroom" row** before each stage and before each long suite: `df -h /` and the container runtime's data root. Set an agreed minimum (a fixed number of GB free, raised after any incident) and list in advance what may be deleted and who approves.
2. **Know what shares the volume.** The home directory, the system temp directory, the container image/layer store and the production database volume were all on one filesystem. A test suite with a fresh test database, build caches and a copied tree can fill it within the suite's runtime.
3. **Do not export trees onto the root volume.** Use `git archive` of the exact commit to a path on a data volume, and verify the exported tree equals the deployed tree for the paths that matter.
4. **Hold, do not push on, when free space is low and stable.** A volume at 100 per cent with almost nothing free was stable but unsafe; the suite was held until space was freed.
5. **Symptoms in production.** Database `DiskFull` errors on UPDATE statements in background workers that were unrelated to the suite. After freeing space, grep worker logs for the error and the time it stopped.
6. **Make background mutations idempotent and recomputable**, so failed updates can be found (by the error log) and recomputed by re-running the refresh for the affected entities. Here a handful of updates were lost and recomputed with a per-entity refresh command; no data was lost.
7. **Clean-up limits.** Root-owned files created by containers cannot be deleted by the operator; list them for a privileged clean-up rather than attempting it.

## When to Use
Any long CI-style run executed on the same host and volume as production services, including containerised test databases and tree exports for audit.

## Example
```
preflight: df -h / ; docker info | grep 'Docker Root Dir' ; df -h <docker root>
rule:      free >= <agreed minimum GB> else STOP and request approval to delete listed scratch directories
export:    git archive <commit> | tar -x -C /data_volume/suite_tree      # never onto /
after:     grep -c DiskFull worker logs since <time>; refresh affected entities; recheck
```

## Pitfalls
- A single large export taken "just for convenience" tipped an already almost-full volume over the edge.
- Test databases not created with a keep flag are dropped at the end, but only after the suite has consumed the space.
- Deleting old scratch is irreversible; get explicit approval and identify what is deleted by age and ownership.
