# Pausing a Task Scheduler for a Maintenance Window With an Explicit Catch-Up Table

<!-- brain-entry -->
<!-- domain: general -->
<!-- subdomain: operations-scheduling -->
<!-- tags: celery, celery-beat, scheduler, pause, catch-up, maintenance-window, periodic-tasks, runbook -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-11-20 -->

## Summary
When a periodic-task scheduler must be stopped for a long window, record the pause start time, enforce a maximum pause, and before restart compute which entries would have fired. Check each of them was executed exactly once after the restart and have a human initial each row. Do not assume the scheduler's start-up behaviour is what you want.

## Knowledge
1. **Inventory before the pause.** Dump the scheduler's entry names (for Celery beat, `sorted(settings.CELERY_BEAT_SCHEDULE)`) from the exact release being deployed and save the list. Compare it with the live list at restart; a changed list invalidates the catch-up logic.
2. **Record the pause start** as an ISO 8601 timestamp with offset in a file, and write down the deadline (start plus the agreed maximum pause). Agree the maximum in numbers; an answer such as "a couple of days" must be turned into hours before it is relied on.
3. **Also list what else stops.** The pause silently stops nightly jobs, weekly digests and periodic reference-data updates. List them in the runbook, and state in which of them a missed run is harmless (not due in the window, for example weekly or monthly jobs) and which need a catch-up.
4. **Compute the missed set from the schedule**, using a small read-only helper that takes the pause start and returns every entry that would have fired at least once since then. Write it to a catch-up table with one row per entry: entry name, expected action, task id observed after restart, outcome, owner initials.
5. **Restart, then verify.** Start the scheduler, wait, and for each missed entry find the executed task (id and state). Interval entries (for example a reaper running every few minutes) will simply resume; one-per-night entries should show exactly one start-up firing, not several. Record any pre-existing start-up warning so it is not blamed on the pause.
6. **Human sign-off.** The owner initials each row; empty rows block closing the window.
7. **Do not restart early without consequence.** If the window overruns the maximum pause, restarting the scheduler can write data that breaks the data freeze; the runbook must state the redo scope in that case.

## When to Use
Maintenance, migration or bulk re-score windows in which a periodic scheduler must not write into the system being frozen, and the window can last longer than one cycle of the nightly jobs.

## Example
```
pause_start = <pause start, ISO 8601 with offset>   deadline = pause_start + agreed maximum
missed = helper(pause_start)  -> the missed subset of entries (the rest: weekly/monthly, not due)
restart scheduler -> each missed entry fired once, 0 errors in scheduler and workers
catch-up table: entry | task id | state | initials   (all rows initialled, restart before the deadline)
```

## Pitfalls
- The start-up burst may fire missed entries immediately and together; check worker capacity, and do not re-send them manually.
- A scheduler's own persistence (a database-backed schedule) can change what fires at start-up; test on a rehearsal copy first.
- Task ids are for the table only; do not paste them into shared notes if the system is sensitive.
