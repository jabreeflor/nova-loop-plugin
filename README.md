# Nova Loop — Claude Code Plugin

Dual-agent autonomous **build → verify → fix → publish → review** loop for [Claude Code](https://claude.com/claude-code). A Builder agent implements features while a read-only Reviewer agent audits PRs, shipping specs to PR-ready state with minimal human intervention.

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
| `/nova-help` | Show plugin help |

## Architecture

```
SETUP
═══════════════════════════════════════════════════════════════════════

 ┌──────────┐    ┌──────────┐    ┌──────────────┐  ┌──────────────┐
 │   READ   │    │  CREATE  │    │   BUILDER    │  │  REVIEWER    │
 │   SPEC   │───>│ WORKTREE │───>│   SCOUT      │  │   SCOUT      │
 │          │    │          │    │              │  │              │
 │ feature  │    │ isolated │    │ structure,   │  │ architecture,│
 │ spec.md  │    │ branch   │    │ conventions, │  │ security,    │
 │          │    │          │    │ build system │  │ quality norms│
 └──────────┘    └──────────┘    └──────────────┘  └──────────────┘
                                       │                  │
                                       └──── parallel ────┘

THE LOOP (up to N cycles)
═══════════════════════════════════════════════════════════════════════

  INNER LOOP (up to N retries)
  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │  ┌─────────┐    ┌─────────┐    ┌─────────┐     │
  │  │  BUILD  │───>│ VERIFY  │──┬>│   FIX   │──┐  │
  │  │         │    │         │  │ │         │  │  │
  │  │ TDD     │    │ tests   │  │ │ read    │  │  │
  │  │ style   │    │ lint    │  │ │ errors, │  │  │
  │  │         │    │ types   │  │ │ fix     │  │  │
  │  └─────────┘    └─────────┘  │ └─────────┘  │  │
  │       ↑                      │       │       │  │
  │       │                 fail │       └───────┘  │
  │       │                      │     retry loop   │
  │       │                      │                  │
  └───────│──────────────────────│──────────────────┘
          │                 pass │
          │                      ▼
          │              ┌──────────┐    ┌──────────────────┐
          │              │ PUBLISH  │───>│     REVIEW       │
          │              │          │    │                  │
          │              │ commit   │    │ Read-only agent  │
          │              │ push     │    │ gh pr diff ONLY  │
          │              │ PR       │    │ gh pr view ONLY  │
          │              └──────────┘    └────────┬─────────┘
          │                                       │
          │         pass → DONE (PR ready)        │
          │                                       │
          └──── fail → findings fed back ─────────┘
```

### Builder Agent

Full read/write access. Plans the approach from the spec, writes tests first (TDD), then implements the feature following existing project conventions. Fixes issues from verification failures and review findings.

### Reviewer Agent

Strictly read-only (Explore-type agent). Spawned each cycle to audit the PR. Can only use `gh pr diff` and `gh pr view` — cannot read files directly or edit code. Returns a structured verdict (PASS/FAIL) with categorized findings by severity.

### How the Agents Communicate

The agents don't talk to each other directly. The orchestrator mediates:

1. **Setup** — Two parallel Explore agents scout the codebase. Builder scout reports structure and conventions; Reviewer scout reports architecture and quality norms.
2. **Build** — The orchestrator uses Builder knowledge + any prior review findings to implement.
3. **Review** — A fresh Explore agent receives Reviewer knowledge + the feature spec, then audits the PR via GitHub CLI.
4. **Feedback** — If the review fails, the orchestrator extracts findings and feeds them into the next BUILD cycle.

### Review Categories

| Category | Description |
|----------|-------------|
| `bug/major` | Will cause runtime failures |
| `bug/minor` | Edge case bug |
| `architecture/major` | Wrong layer, needs restructuring |
| `architecture/minor` | Could be better, works fine |
| `testing/major` | Missing critical coverage |
| `testing/minor` | Missing edge case test |
| `security/major` | Exploitable vulnerability |
| `security/minor` | Defensive improvement |
| `style/minor` | Naming, formatting nits |

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
