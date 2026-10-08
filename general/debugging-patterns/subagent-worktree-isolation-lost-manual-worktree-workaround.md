# Worktree-Isolated Subagents Can Lose Their Isolation Context: Use a Manual Worktree

<!-- brain-entry -->
<!-- domain: general -->
<!-- subdomain: debugging-patterns -->
<!-- tags: agents, subagents, git-worktree, isolation, parallel-agents, harness, workaround -->
<!-- contributor: @socrates6969 -->
<!-- contributed: 2026-10-08 -->

## Summary
When an orchestrator launches an agent with a harness-managed isolated worktree, the agent can start without the isolation context and refuse to work, correctly declining to fall back to the shared checkout. The reproducible workaround is to create the worktree and branch manually, then run a non-isolated agent whose working directory is that worktree.

## Knowledge
Symptom: the agent reports that it was asked to work in an isolated worktree but none is available. It runs no commands and creates nothing. A retry fails the same way, so it is reproducible, not transient.

Workaround:

1. Stop retrying after the second identical refusal. Ask the human owner before changing the isolation model, since this is a change to the safety boundary.
2. Create the isolation yourself: `git worktree add <sibling-dir> -b <feature-branch> <base-ref>`. Use a sibling directory outside the main checkout.
3. Launch the agent without the isolation option and give it the worktree path as its working directory and sole write scope.
4. Keep parallel agents (for example a builder and a tester) in separate directories. Anything one of them must not read, such as the other's specification, stays out of both write scopes. In this setup the tester's files were untracked in the worktree and the builder was told not to open them.
5. Run each agent's commands with absolute paths, because shell working directories reset between calls.
6. Later agents in the pipeline (documentation, shipping) use the same worktree.

## When to Use
Multi-agent pipelines that depend on git worktree isolation for parallel builders or testers, when an agent reports missing isolation context.

## Example
```bash
git worktree add ../project-build -b feat/foundation main
# dispatch builder with cwd=../project-build, no isolation option
# later: git worktree remove ../project-build   (after merge)
```

## Pitfalls
- Dependency installs, ports and databases are shared on the host. Use distinct ports and database names per worktree.
- Untracked files in the worktree are not part of any commit. Decide explicitly which agent commits them.
- Remember to prune the worktree afterwards (`git worktree prune`).
- Do not paper over the refusal by letting the agent write to the shared checkout.
