---
name: commit-push-pr
description: Commit all changes, push to remote, and create or update a pull request in one step.
user-invocable: true
---

# Commit, Push & Create/Update PR

You are performing the full publish step: commit → push → create or update a PR.

## Instructions

### 1. Analyze changes

Run these in parallel:
- `git status` (never use `-uall`)
- `git diff` and `git diff --cached` to see all staged + unstaged changes
- `git log --oneline -10` for commit message style reference
- `git branch --show-current` to get the current branch name

### 2. Stage and commit

- Stage all relevant changed files by name (avoid `git add -A`; never stage `.env`, credentials, or secrets)
- Write a concise commit message that:
  - Summarizes the **why** not the what
  - Matches the repo's existing commit style
  - Ends with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`
- Use a HEREDOC to pass the message:
```bash
git commit -m "$(cat <<'EOF'
Your message here.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 3. Push

- If the branch does not track a remote yet, push with `-u origin <branch>`
- Otherwise, a normal `git push` is sufficient
- **Never force-push** unless the user explicitly asked

### 4. Create or update the PR

Check if a PR already exists for this branch:
```bash
gh pr view --json number 2>/dev/null
```

**If no PR exists**, create one:
```bash
gh pr create --title "<short title>" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points describing the changes>

## Test plan
- [ ] <testing checklist items>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

**If a PR already exists**, the push is enough — the PR auto-updates. Optionally add a comment summarizing the new changes:
```bash
gh pr comment --body "Updated: <brief summary of new changes>"
```

### 5. Report back

Print the PR URL so the user (or the orchestrating loop) can see it.
