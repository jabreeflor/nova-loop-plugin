---
name: cancel-nova
description: Cancel an active Nova Loop and clean up the worktree.
user-invocable: true
---

# Cancel Nova Loop

Stop the currently running Nova Loop and clean up all associated resources.

## Instructions

1. Stop any in-progress build, test, or review work immediately.

2. Check for Nova Loop worktrees:
```bash
git worktree list | grep nova
```

3. Check for open PRs from Nova Loop branches:
```bash
gh pr list --head nova/ --state open --json number,title,headRefName,url
```

4. Report what was found:
   - List each worktree path and branch name
   - List each open PR with its number, title, and URL

5. For each Nova Loop worktree found, ask the user:
   - **Keep**: Leave the worktree and branch intact for manual inspection
   - **Remove**: Clean up the worktree and optionally delete the branch

6. If the user chooses to remove:
```bash
git worktree remove <worktree-path>
git branch -d nova/<branch-name>  # only if user confirms branch deletion
```

7. For open PRs, ask the user:
   - **Leave open**: Keep the PR for review
   - **Close**: Close the PR without merging
```bash
gh pr close <number>  # only if user confirms
```

8. Report final cleanup summary:
   - Worktrees removed or kept
   - Branches deleted or preserved
   - PRs closed or left open
   - Current repo state (confirm back on main branch if needed)
