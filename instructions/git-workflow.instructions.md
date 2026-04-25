---
description: "Git workflow, branching strategy, PR requirements, commit conventions, and merge policies. Enforces that all code reaches main via pull request."
---

# Git Workflow & Conventions

## Branching Strategy

### Protected Branches

- `main` is the **production branch** — always deployable, always protected
- Direct pushes to `main` are **forbidden** — all changes arrive via pull request

### Branch Naming

**Official repositories** (team/org repos) use a user-scoped prefix:

| Pattern | Use Case | Example |
|---|---|---|
| `user/pingugi/feature/{short-desc}` | New features | `user/pingugi/feature/user-dashboard` |
| `user/pingugi/bugfix/{issue-id}-{short-desc}` | Bug fixes | `user/pingugi/bugfix/142-login-timeout` |
| `user/pingugi/hotfix/{short-desc}` | Production emergencies | `user/pingugi/hotfix/fix-payment-crash` |
| `user/pingugi/release/{version}` | Release preparation | `user/pingugi/release/2.4.0` |

**Personal repositories** may use shorter names without the user prefix:

| Pattern | Use Case | Example |
|---|---|---|
| `feature/{short-desc}` | New features | `feature/user-dashboard` |
| `bugfix/{issue-id}-{short-desc}` | Bug fixes | `bugfix/142-login-timeout` |

> **Rule:** When in doubt, use `user/pingugi/` prefix — it's always safe and identifies your branches in shared repos.

### Branch Lifecycle

- ✅ **DO** branch from `main` for all new work
- ✅ **DO** delete feature branches immediately after merge
- ✅ **DO** keep branches short-lived — under 1 week ideal, 2 weeks maximum
- ✅ **DO** rebase on `main` regularly to avoid large merge conflicts
- ❌ **DON'T** branch from other feature branches (creates dependency chains)
- ❌ **DON'T** let branches go stale — if it's older than 2 weeks, close or rebase it

## Pull Request Requirements

### Every PR Must Have

1. **Title** following Conventional Commits: `type(scope): description`
   - Example: `feat(auth): add two-factor authentication support`
   - Example: `fix(api): handle null response from payment gateway`
2. **Description** that includes:
   - What changed and why
   - How it was tested
   - Any breaking changes (with migration steps)
   - Screenshots/recordings for UI changes
3. **Linked work items** — every PR references an issue or task
4. **At least 1 approval** before merge
5. **All CI checks passing** — build, tests, lint, security scan

### PR Hygiene

- ✅ **DO** keep PRs focused — one logical change per PR
- ✅ **DO** self-review before requesting others
- ✅ **DO** respond to all review comments before re-requesting review
- ✅ **DO** update PR description if scope changes during review
- ❌ **DON'T** bundle unrelated changes in a single PR
- ❌ **DON'T** force-push after reviews have started (use fixup commits, squash at merge)
- ❌ **DON'T** merge your own PR without at least one other approval

## Commit Conventions (Conventional Commits)

### Commit Types

| Type | Purpose | Example |
|---|---|---|
| `feat` | New feature | `feat(dashboard): add export-to-CSV button` |
| `fix` | Bug fix | `fix(auth): prevent session fixation on login` |
| `docs` | Documentation only | `docs(api): add rate-limit section to README` |
| `test` | Adding or modifying tests | `test(cart): add edge case for empty cart total` |
| `refactor` | Code change — no bug fix, no feature | `refactor(db): extract query builder from repository` |
| `perf` | Performance improvement | `perf(search): add index for full-text queries` |
| `chore` | Build process, deps, tooling | `chore(deps): upgrade express to 4.19` |
| `ci` | CI/CD configuration | `ci: add codeql analysis to PR pipeline` |

### Commit Message Format

```
type(scope): concise imperative description

Body: Explain WHY, not WHAT. The diff shows what changed.
Provide context that helps future readers understand the decision.

Footer:
Fixes #123
Relates to #456
Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

### Commit Rules

- ✅ **DO** use imperative mood: "add feature" not "added feature"
- ✅ **DO** keep subject line under 72 characters
- ✅ **DO** include `Co-authored-by:` trailer when AI-assisted
- ✅ **DO** reference issues in the footer
- ✅ **DO** explain the reasoning in the body when the change is non-obvious
- ❌ **DON'T** use vague messages: "fix bug", "update code", "WIP"
- ❌ **DON'T** commit multiple unrelated changes in one commit

## Pre-Push Checklist

Before pushing any branch, verify all of the following:

- [ ] Code builds successfully locally
- [ ] All existing tests pass (`npm test`, `dotnet test`, etc.)
- [ ] New code has corresponding tests
- [ ] No secrets, credentials, API keys, or connection strings committed
- [ ] No `.env` files, certificates, or private keys included
- [ ] Code is formatted (`prettier`, `dotnet format`, etc.)
- [ ] Linter passes with no new warnings
- [ ] Self-review completed — read your own diff before pushing
- [ ] Commit messages follow Conventional Commits format

## Merge Strategy

| Branch Type | Merge Method | Rationale |
|---|---|---|
| Feature branches → `main` | **Squash merge** | Clean, linear main history |
| Release branches → `main` | **Merge commit** | Preserve release context |
| Hotfix branches → `main` | **Fast-forward** | Minimal overhead for urgent fixes |

### After Merge

- ✅ **DO** delete the source branch immediately
- ✅ **DO** verify the merge didn't break CI on `main`
- ✅ **DO** close linked issues/work items
- ❌ **DON'T** leave merged branches lingering in the remote

## Conflict Resolution

- Rebase your branch on `main` before opening a PR
- If conflicts arise during review, resolve them via rebase (not merge commits)
- When in doubt about conflict resolution, discuss with the original author
- Never resolve conflicts by deleting someone else's code without understanding it
