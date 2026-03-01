---
name: help
description: Explain how the Nova Loop plugin works and list available commands.
user-invocable: true
---

# Nova Loop Plugin v2 — Help

Explain the Nova Loop plugin to the user. Use the following information:

## Overview

The Nova Loop is a **dual-agent** autonomous build-verify-fix-publish-review cycle for Claude Code. It takes a feature spec and ships it to a PR-ready state with minimal human intervention.

Two specialized agents work together:
- **Builder Agent** — Has full read/write access. Implements features, writes tests, fixes issues.
- **Reviewer Agent** — Strictly read-only. Audits PRs via GitHub CLI only (`gh pr diff`, `gh pr view`). Cannot touch code.

## Available Commands

| Command | Description |
|---------|-------------|
| `/nova-loop <spec.md>` | Start the autonomous loop on a feature spec |
| `/commit-push-pr` | Commit, push, and create/update a PR in one step |
| `/cancel-nova` | Stop an active loop and clean up worktrees/PRs |
| `/nova-help` | Show this help message |

## How It Works

### Setup
1. **Spec** — You write a `.md` file describing the feature you want built
2. **Worktree** — An isolated copy of your repo is created (main branch stays clean)
3. **Builder scouts** — An Explore agent studies the codebase for build context (structure, conventions, test framework)
4. **Reviewer scouts** — A parallel Explore agent studies the codebase for review context (architecture, security, quality norms)

### The Loop (up to N cycles)

```
OUTER LOOP
│
│  INNER LOOP (up to N retries)
│  ├── BUILD   ── Orchestrator plans + writes code (TDD style)
│  ├── VERIFY  ── Orchestrator runs tests/lint/types
│  └── FIX     ── On fail: read errors, fix root cause, re-verify
│
├── PUBLISH ── /commit-push-pr (commit, push, create/update PR)
├── REVIEW  ── Read-only Reviewer agent audits PR via gh CLI
│
├── PASS? → DONE (PR ready for human review)
└── FAIL? → Findings fed back to BUILD for next cycle
```

### Agent Roles

**Builder (general-purpose agent)**
- Plans the approach from the spec and codebase knowledge
- Writes tests first (TDD), then implements the feature
- Fixes issues found by VERIFY or reported by the Reviewer
- Has full access to edit files, run commands, and write code

**Reviewer (Explore agent — read-only)**
- Spawned fresh each cycle to audit the PR
- Can ONLY run `gh pr diff` and `gh pr view`
- Cannot read files directly, edit code, or run tests
- Returns a structured verdict: PASS or FAIL with categorized findings
- Findings are fed back to the Builder if the review fails

### Options

- `--cycles N` — Maximum outer cycles (default: 3)
- `--retries N` — Maximum inner fix retries (default: 3)

### Example

```bash
/nova-loop features/add-auth.md --cycles 5 --retries 3
```
