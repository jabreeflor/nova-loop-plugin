---
name: nova-loop
description: "Autonomous build → verify → fix → publish → review loop. Takes a feature spec and ships it to a PR-ready state."
user-invocable: true
args: "<path-to-spec.md> [--cycles N] [--retries N]"
---

# Nova Loop — Dual-Agent Autonomous Feature Builder

You are the orchestrator of the Nova Loop. Given a feature spec, you will coordinate a **Builder agent** and a **Reviewer agent** to autonomously build, verify, fix, publish, and review the feature until it is PR-ready or the cycle limit is reached.

## Configuration

Parse arguments from `$ARGUMENTS`:
- First positional arg: path to the feature spec `.md` file (required)
- `--cycles N`: max outer cycles (build→review), default 3
- `--retries N`: max inner retries (verify→fix), default 3

If no spec path is provided, ask the user for one.

## Phase 0: SETUP

### 0a. Read the spec
Read the feature spec file. This is the source of truth for what to build.

### 0b. Create a worktree
Create an isolated worktree so the main branch is never touched:
```bash
git worktree add .claude/worktrees/nova-$(date +%s) -b nova/feature-$(date +%s)
```
Change your working directory into the new worktree.

### 0c. Dual-agent codebase learning

Spawn **two parallel Explore agents** to learn the codebase concurrently. Each agent focuses on different aspects to build specialized knowledge.

**Agent 1 — Builder Perspective** (subagent_type: "Explore"):
```
Prompt: "Study this codebase from a BUILDER perspective. Report back:
1. Project structure — source dirs, entry points, module organization
2. Build system — package manager, build commands, scripts
3. Coding conventions — naming, imports, patterns, formatting
4. Test framework — test runner, test file locations, test patterns, assertion style
5. Dependencies — key libraries, versions, what's available
Return a structured summary I can reference when implementing features."
```

**Agent 2 — Reviewer Perspective** (subagent_type: "Explore"):
```
Prompt: "Study this codebase from a REVIEWER perspective. Report back:
1. Architecture patterns — layering, separation of concerns, module boundaries
2. Testing standards — coverage expectations, test quality norms, what gets tested
3. Security patterns — input validation, auth patterns, data sanitization
4. Code quality norms — error handling, logging, documentation standards
5. Common anti-patterns — anything in the codebase that should NOT be replicated
Return a structured summary I can reference when reviewing PRs."
```

Both agents run in parallel. Wait for both to complete.

### 0d. Store codebase knowledge
Combine both agents' findings into your working context:
- **Builder knowledge**: structure, build system, conventions, test framework
- **Reviewer knowledge**: architecture patterns, testing standards, security norms, quality expectations

You will use Builder knowledge during BUILD phases and pass Reviewer knowledge to the Reviewer agent during REVIEW phases.

## Phases 1–4: THE LOOP

The loop has an **inner loop** (build → verify → fix) and an **outer loop** (publish → review → next cycle).

```
OUTER LOOP (up to N cycles):
│
│  INNER LOOP (up to N retries):
│  │  1. BUILD  — Plan + write code (TDD style)
│  │  2. VERIFY — Run tests/lint/types
│  │  2b. FIX  — On fail, read errors, fix root cause, re-verify
│  │
│  3. PUBLISH — Commit, push, create/update PR
│  4. REVIEW  — Spawn read-only Reviewer agent
│
│  If PASS → DONE
│  If FAIL → Extract findings, feed back to BUILD
```

### Phase 1: BUILD

Plan your approach based on:
- The feature spec
- Builder codebase knowledge from Phase 0c
- Review findings from previous cycles (if any)

Then implement using TDD style:
1. Write or update tests first that define the expected behavior
2. Write the implementation code to make tests pass
3. Follow existing project conventions exactly

### Phase 2: VERIFY

Run the project's test suite and any linting/type-checking:
```bash
# Detect and run appropriate test command
# npm test, pytest, cargo test, go test, etc.
```

Also verify:
- [ ] All new tests pass
- [ ] Existing tests still pass
- [ ] No linting errors
- [ ] No type errors
- [ ] The implementation matches the spec

**If verification passes** → proceed to Phase 3.
**If verification fails** → enter Phase 2b.

### Phase 2b: FIX (inner loop, up to N retries)

Read the failure output carefully. Fix the root cause — don't patch symptoms. Then re-run verification.

If you exhaust retries, proceed to Phase 3 anyway but note the failures in the PR description.

### Phase 3: PUBLISH

Use the `/commit-push-pr` skill to:
1. Commit all changes with a descriptive message
2. Push to the remote branch
3. Create or update the PR

When called from the Nova Loop, include the cycle number in the commit message and PR description:
- Commit message: prefix with `[nova cycle N]`
- PR description: include a "Nova Loop Status" section showing the current cycle number and total allowed

### Phase 4: REVIEW — Read-Only Reviewer Agent

Spawn an **Explore-type agent** (subagent_type: "Explore") as the Reviewer. This enforces read-only access — the Reviewer has no Edit or Write tools.

Pass the Reviewer this prompt:

```
You are the Reviewer agent in the Nova Loop. Review the current PR using ONLY GitHub CLI commands.

<reviewer-instructions>
{contents of .claude/skills/reviewer-instructions.md}
</reviewer-instructions>

<reviewer-codebase-knowledge>
{Reviewer knowledge from Phase 0c}
</reviewer-codebase-knowledge>

<feature-spec>
{The original feature spec}
</feature-spec>

Run `gh pr diff` and `gh pr view` to inspect the PR. Then return your verdict in the exact format specified in the reviewer instructions.
```

**Important**: Read the file `.claude/skills/reviewer-instructions.md` and include its contents in the Reviewer agent's prompt. This file contains the review criteria, finding categories, and verdict format.

### Processing the Review Verdict

Parse the Reviewer's response:

- **If VERDICT: PASS** → The PR is ready. Proceed to Completion.
- **If VERDICT: FAIL** → Extract the FINDINGS list. These become the input for the next cycle's BUILD phase.

Format findings for the next BUILD cycle:
```
Review findings from cycle N:
- <category> <file>:<line> — <description> → <suggested action>
- ...
Address all major findings. Minor findings are optional.
```

## Completion

When the loop finishes (either review passed or cycles exhausted):

1. **Print a cycle summary table**:
```
┌───────┬──────────┬─────────┬────────────────┐
│ Cycle │ Build    │ Verify  │ Review         │
├───────┼──────────┼─────────┼────────────────┤
│ 1     │ ✓ done   │ ✗ 2 fix │ FAIL (2 major) │
│ 2     │ ✓ done   │ ✓ pass  │ PASS           │
└───────┴──────────┴─────────┴────────────────┘
```

2. **Print the final PR URL**

3. **If cycles exhausted with remaining findings**: Post the unresolved findings as a PR comment using:
```bash
gh pr comment --body "$(cat <<'EOF'
## Nova Loop — Unresolved Findings

The following findings were not resolved within the cycle limit:

<list findings here>
EOF
)"
```

## Important Rules

- **Never touch the main/master branch directly**
- **Never force-push**
- **Never skip test verification**
- **Keep the worktree isolated** — all work happens there
- **The Reviewer agent is read-only** — it uses `gh pr diff` and `gh pr view` only, never reads files directly
- **The Reviewer agent is a separate Explore-type agent** — not the orchestrator switching hats
- **Builder and Reviewer knowledge are separated** — each agent gets the codebase knowledge relevant to its role
