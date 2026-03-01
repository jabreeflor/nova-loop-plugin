---
name: help
description: Explain how the Nova Loop plugin works and list available commands.
user-invocable: true
---

# Nova Loop Plugin — Help

Explain the Nova Loop plugin to the user. Use the following information:

## Overview

The Nova Loop is an autonomous build-verify-fix-publish-review cycle for Claude Code. It takes a feature spec and ships it to a PR-ready state with minimal human intervention.

## Available Commands

| Command | Description |
|---------|-------------|
| `/nova-loop <spec.md>` | Start the autonomous loop on a feature spec |
| `/commit-push-pr` | Commit, push, and create/update a PR in one step |
| `/cancel-nova` | Stop an active loop and clean up worktrees |
| `/help` | Show this help message |

## How It Works

### Setup
1. **Spec** — You write a `.md` file describing the feature you want built
2. **Worktree** — An isolated copy of your repo is created (main branch stays clean)
3. **Learning** — The AI studies your codebase before writing any code

### The Loop (up to N cycles)

```
BUILD → VERIFY → FIX (retry) → PUBLISH → REVIEW
  ↑                                          |
  └──── findings sent back if review fails ──┘
```

1. **Build** — Plans the approach, writes tests first (TDD), then implements
2. **Verify** — Runs tests, linting, and type-checking
3. **Fix** — If verification fails, reads errors and fixes them (up to N retries)
4. **Publish** — Commits, pushes, creates/updates the PR
5. **Review** — Reviews the PR diff as a separate reviewer (read-only, can't touch code)

If the review passes → **Done!** PR is ready for human review.
If the review finds issues → Findings are sent back to the builder for the next cycle.

### Options

- `--cycles N` — Maximum outer cycles (default: 3)
- `--retries N` — Maximum inner fix retries (default: 3)

### Example

```bash
/nova-loop features/add-auth.md --cycles 5 --retries 3
```
