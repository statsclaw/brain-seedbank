# Make a Code and Data Freeze a Detected Condition, Not a Request

<!-- brain-entry -->
<!-- domain: general -->
<!-- subdomain: change-control -->
<!-- tags: freeze, fingerprint, drift-detection, shared-main, breach-procedure, deployment, reproducibility -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
Asking other developers or automated sessions not to merge during a deployment window does not work. Record a fingerprint of everything the window depends on at the start, compare it at defined checkpoints, and write the breach procedure into the runbook beforehand.

## Knowledge
1. **Fingerprint** at freeze start and store it: the head commit of the shared branch, a hash over the source files whose content affects results (the "result-defining sources"), the schema migration leaf, the engine version, and a checksum of the data files read by the engine.
2. **Compare at checkpoints**: before each long step, at the end of each long step, before the final go decision and before the switch-over. A mismatch stops the window.
3. **Bake the fingerprint into the run itself.** The reference run stores the source hash at creation and the preflight check requires it to equal the current one, so a mid-window change makes the run unusable rather than silently stale.
4. **Predefine the breach procedure**: (a) the agent changes nothing and asks the owner; (b) the owner chooses accept-and-redo or revert; (c) on accept, set a new baseline variable, supersede (do not sign) the run built on the old baseline, and rerun the dry runs and the reference run; (d) if the new dry-run outputs (counts, transitions, per-item changed sets and attribution) are identical to the first round, the earlier human acceptance carries over; any difference goes back to the reviewer.
5. **Cost model.** In this run one unrequested merge forced a couple of hours of machine time to repeat; cheap compared with signing a stale run. Redo cost should be stated in the runbook so the owner can decide quickly.
6. **Where a rule can be enforced technically** (a protected branch, a deploy lock), do that as well; detection is the backstop, not the only layer.

## When to Use
Any shared deployment target where several people or sessions can merge, restart containers or write reference data while a validated, signed run is being produced.

## Example
```
freeze_start: main = <sha_a>, source_hash = <h1>, migration_leaf = <leaf>, data_sha = <d1>
checkpoint:   main = <sha_b>, source_hash = <h2>  -> MISMATCH -> stop, ask owner
owner: accept -> baseline := <sha_b>/<h2>; supersede the old reference run; redo dry runs + a new reference run;
       dry-run outputs identical -> earlier acceptance stands
```

## Pitfalls
- Hash only files that change results; hashing the whole tree makes documentation-only commits look like breaches.
- A docs-only commit made directly on the shared branch after the breach still moves the head; refresh the baseline variable.
- Literal baseline names in the runbook make every later gate ambiguous after a breach; use variables.
