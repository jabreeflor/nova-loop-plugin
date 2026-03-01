# Nova Loop — Claude Code Plugin

Autonomous **build → verify → fix → publish → review** loop for [Claude Code](https://claude.com/claude-code). Takes a feature spec and ships it to a PR-ready state with minimal human intervention.

## Install

```bash
claude plugin add jabreeflor/nova-loop-plugin
```

Or clone and link locally:

```bash
git clone https://github.com/jabreeflor/nova-loop-plugin.git
cd nova-loop-plugin
claude plugin link .
```

## Commands

| Command | Description |
|---------|-------------|
| `/nova-loop <spec.md> [--cycles N] [--retries N]` | Start the autonomous loop |
| `/commit-push-pr` | Commit + push + create/update PR in one step |
| `/cancel-nova` | Stop an active loop and clean up |
| `/help` | Show plugin help |

## Architecture

```
SETUP
─────
  Feature spec → Worktree (isolated copy) → AI learns codebase

THE LOOP (up to N cycles)
─────────────────────────

  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
  │  BUILD  │───>│ VERIFY  │───>│   FIX   │───>│ PUBLISH │───>│ REVIEW  │
  │         │    │         │    │ (retry) │    │         │    │         │
  │ TDD     │    │ tests   │    │ read    │    │ commit  │    │ PR diff │
  │ style   │    │ lint    │    │ errors, │    │ push    │    │ only    │
  │         │    │ types   │    │ fix     │    │ PR      │    │ (read-  │
  └─────────┘    └─────────┘    └─────────┘    └─────────┘    │  only)  │
       ↑                                                       └────┬────┘
       │                                                            │
       │         pass → DONE (PR ready for human review)            │
       │                                                            │
       └──── fail → findings extracted and fed back to builder ─────┘
```

### Build
Plans the approach from the spec, writes tests first (TDD), then implements the feature following existing project conventions.

### Verify
Runs the project's test suite, linting, and type-checking. Checks that the implementation matches the spec.

### Fix (inner loop)
If verification fails, reads error output and fixes root causes. Retries up to N times before proceeding.

### Publish
Uses `/commit-push-pr` to commit all changes, push to the feature branch, and create or update the pull request.

### Review
Switches to reviewer mode. Reads the PR diff via GitHub CLI only — cannot touch the code. Evaluates correctness, architecture, testing, security, and code quality. Categorizes findings by severity and sends major findings back to the builder.

## Feature Spec Format

Create a markdown file describing what you want built:

```markdown
# Add User Authentication

## Requirements
- JWT-based auth with refresh tokens
- Login and registration endpoints
- Password hashing with bcrypt
- Protected route middleware

## Acceptance Criteria
- [ ] POST /auth/register creates a new user
- [ ] POST /auth/login returns JWT + refresh token
- [ ] Protected routes return 401 without valid token
- [ ] Refresh token endpoint issues new JWT
- [ ] All auth endpoints have tests
```

Then run:

```bash
/nova-loop features/add-auth.md --cycles 5
```

## License

MIT
