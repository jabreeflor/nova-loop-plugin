---
name: cancel-nova
description: Cancel an active Nova Loop and clean up the worktree.
user-invocable: true
---

# Cancel Nova Loop

Stop the currently running Nova Loop and clean up.

## Instructions

1. Stop any in-progress build, test, or review work immediately.

2. Check for Nova Loop worktrees:
```bash
git worktree list | grep nova
```

3. For each Nova Loop worktree found, ask the user:
   - **Keep**: Leave the worktree and branch intact for manual inspection
   - **Remove**: Clean up the worktree and optionally delete the branch

4. If the user chooses to remove:
```bash
git worktree remove <worktree-path>
git branch -d nova/<branch-name>  # only if user confirms
```

5. Report what was cleaned up and the current state of any open PRs from the loop.
