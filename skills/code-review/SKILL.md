---
description: >
  Multi-model code review ensuring engineering excellence across 5 AI reviewers.
  Use this skill when the user says 'code review', 'review my code', 'review changes',
  'review this PR', 'multi-model review', 'review before merge', 'pre-commit review',
  or 'quality review'. Each model reviews independently, findings are consolidated,
  and all issues are addressed before committing.
---

# Code Review — Multi-Model Engineering Excellence Review

## Context

All code changes must be reviewed by 5 AI models before committing. Each model brings a different perspective — architecture, correctness, code quality, security, and problem-solution fit. This skill orchestrates the full review pipeline.

## When to Use

- Before committing significant code changes
- Before opening a pull request
- After implementing a feature or bug fix
- When the user says "code review", "review my code", "review changes"

## Review Models

Every review uses all 5 models. Each has a designated focus area:

| # | Model | Focus Area | Priority |
|---|-------|------------|----------|
| 1 | **GPT-5.5** | Architecture, scalability, design patterns, SOLID principles | Structure |
| 2 | **Claude Opus 4.7** | Deep logic analysis, correctness, completeness, edge cases | Depth |
| 3 | **Claude Opus 4.6** | Problem-solution fit, readability, extensibility, testability | Balance |
| 4 | **GPT-5.3 Codex** | Code quality, idiomatic patterns, performance, DRY | Code |
| 5 | **GPT-5.4** | Security, error handling, dependency impact, breaking changes | Safety |

## Review Checklist

Every reviewer must evaluate ALL of the following:

### 1. Project & Solution Standards
- [ ] Follows the project's defined coding standards and conventions
- [ ] Naming conventions are consistent with the codebase
- [ ] File structure follows the established project layout
- [ ] Configuration follows the project's patterns (appsettings, env vars, etc.)

### 2. Code Coverage & Testing
- [ ] New code has sufficient test coverage (≥80% line coverage)
- [ ] Every public API has at least one test
- [ ] Happy paths, error paths, and edge cases are tested
- [ ] Bug fixes include a regression test that fails without the fix
- [ ] Tests follow Arrange-Act-Assert pattern
- [ ] Tests verify behavior, not implementation details
- [ ] No existing tests were modified to make them pass (tests are sacred)

### 3. Engineering Best Practices & Design Patterns
- [ ] Appropriate design patterns are used (not over-engineered)
- [ ] Code follows SOLID principles:
  - **S** — Single Responsibility: each class/method does one thing
  - **O** — Open/Closed: extensible without modifying existing code
  - **L** — Liskov Substitution: subtypes are substitutable
  - **I** — Interface Segregation: no forced dependency on unused interfaces
  - **D** — Dependency Inversion: depends on abstractions, not concretions
- [ ] DRY — no duplicated logic
- [ ] KISS — solution is as simple as possible, but no simpler
- [ ] YAGNI — no speculative features or unused code

### 4. Code Quality
- [ ] Code is readable — a new team member can understand it without the author's help
- [ ] Code is testable — dependencies are injectable, logic is separable
- [ ] Code is extensible — new requirements won't require rewriting
- [ ] Complex logic is broken into well-named helper methods
- [ ] No magic numbers or hardcoded values without explanation
- [ ] No debug code left in (console.log, debugger, print, etc.)

### 5. Documentation
- [ ] Public APIs are documented with descriptions, parameters, and return values
- [ ] Complex business logic has explanatory comments (why, not what)
- [ ] Non-obvious design decisions are documented
- [ ] README is updated if setup or usage is affected
- [ ] All TODOs have linked issues — no orphaned TODOs
- [ ] Every change is reflected in relevant documentation
- [ ] Affected design docs, feature specs, or bug reports are updated

### 6. No Breaking Changes
- [ ] Public API contracts are not broken
- [ ] Database schema changes are backward-compatible or have migrations
- [ ] Configuration changes have defaults for backward compatibility
- [ ] Dependencies are not removed or upgraded with breaking changes
- [ ] All downstream consumers have been considered

### 7. Dependencies & Impact
- [ ] New dependencies are justified and from trusted sources
- [ ] Dependency versions are pinned in lock files
- [ ] No circular dependencies introduced
- [ ] Impact on other modules/services has been assessed
- [ ] Cross-cutting concerns (auth, logging, error handling) are consistent

### 8. Security
- [ ] No secrets, credentials, or API keys in code
- [ ] User input is validated and sanitized
- [ ] SQL queries are parameterized
- [ ] Authentication and authorization checks are in place
- [ ] Error messages don't expose internal details
- [ ] CORS, CSP, and security headers are configured correctly

### 9. Logical Correctness
- [ ] All code paths produce correct results
- [ ] Edge cases are handled (null, empty, boundary values, overflow)
- [ ] Error handling is comprehensive — no swallowed exceptions
- [ ] Async operations are properly awaited
- [ ] Race conditions and concurrency issues are addressed
- [ ] Resource cleanup is proper (using/dispose, connection pooling)

### 10. Design & Plan Compliance
- [ ] The implementation fully matches the approved design/plan
- [ ] No scope creep — only planned changes are included
- [ ] All acceptance criteria from the task/story are met
- [ ] No unrelated changes bundled in

## Workflow

### Step 1: Gather Changes

Collect the diff to review:

```powershell
# For uncommitted changes
git diff --cached  # staged
git diff           # unstaged

# For a branch comparison
git diff main..HEAD

# For a specific commit
git show <commit-sha>
```

Present a summary of files changed:
```
📁 Files changed: N
  - src/Services/MyService.cs (modified)
  - src/Models/MyModel.cs (added)
  - tests/MyServiceTests.cs (added)
```

### Step 2: Run All 5 Reviews in Parallel

Launch all 5 reviewers as background agents simultaneously:

```
task(agent_type: "rubber-duck", model: "gpt-5.5", name: "review-architecture", prompt: "<diff + checklist>")
task(agent_type: "rubber-duck", model: "claude-opus-4.7", name: "review-correctness", prompt: "<diff + checklist>")
task(agent_type: "rubber-duck", model: "claude-opus-4.6", name: "review-balance", prompt: "<diff + checklist>")
task(agent_type: "rubber-duck", model: "gpt-5.3-codex", name: "review-code-quality", prompt: "<diff + checklist>")
task(agent_type: "rubber-duck", model: "gpt-5.4", name: "review-security", prompt: "<diff + checklist>")
```

Each agent receives:
1. The full diff
2. The complete review checklist (all 10 sections)
3. Any project-specific context (architecture docs, conventions)
4. Instruction to focus on their designated area but flag issues in any area

### Step 3: Consolidate Findings

Collect results from all 5 agents and consolidate:

1. **Deduplicate** — merge identical findings from different models
2. **Classify severity:**
   - 🔴 **Blocking** — must fix before commit (bugs, security, breaking changes)
   - 🟡 **Important** — should fix, significant quality impact
   - 🔵 **Suggestion** — nice to have, non-blocking
3. **Attribute** — note which model(s) flagged each issue

### Step 4: Present Review Report

```markdown
## 📋 Code Review Report

### Summary
| Model | Issues Found | Blocking | Important | Suggestions |
|-------|-------------|----------|-----------|-------------|
| GPT-5.5 (Architecture) | N | N | N | N |
| Claude Opus 4.7 (Correctness) | N | N | N | N |
| Claude Opus 4.6 (Balance) | N | N | N | N |
| GPT-5.3 Codex (Code) | N | N | N | N |
| GPT-5.4 (Safety) | N | N | N | N |
| **Total (deduplicated)** | **N** | **N** | **N** | **N** |

### 🔴 Blocking Issues
| # | Issue | File:Line | Flagged By | Checklist Item |
|---|-------|-----------|------------|----------------|
| 1 | {description} | {location} | {models} | {checklist ref} |

### 🟡 Important Issues
| # | Issue | File:Line | Flagged By | Checklist Item |
|---|-------|-----------|------------|----------------|
| 1 | {description} | {location} | {models} | {checklist ref} |

### 🔵 Suggestions
| # | Suggestion | File:Line | Flagged By |
|---|------------|-----------|------------|
| 1 | {description} | {location} | {models} |

### ✅ Checklist Status
| Category | Status |
|----------|--------|
| Project Standards | ✅ / ⚠️ / ❌ |
| Code Coverage | ✅ / ⚠️ / ❌ |
| SOLID & Design Patterns | ✅ / ⚠️ / ❌ |
| Code Quality | ✅ / ⚠️ / ❌ |
| Documentation | ✅ / ⚠️ / ❌ |
| No Breaking Changes | ✅ / ⚠️ / ❌ |
| Dependencies | ✅ / ⚠️ / ❌ |
| Security | ✅ / ⚠️ / ❌ |
| Logical Correctness | ✅ / ⚠️ / ❌ |
| Design Compliance | ✅ / ⚠️ / ❌ |
```

### Step 5: Address Issues

1. **Blocking issues** — must be fixed. Fix them and re-run review on affected files.
2. **Important issues** — should be fixed. Ask user if they want to address now or create follow-up tasks.
3. **Suggestions** — present to user for consideration. No action required.

### Step 6: Final Verdict

After all blocking issues are resolved:

```markdown
## ✅ Code Review Passed

All 5 models approve. {N} blocking issues resolved, {M} suggestions noted.
Ready to commit.
```

Or if issues remain:

```markdown
## ❌ Code Review — Blocking Issues Remain

{N} blocking issues still unresolved. Address them before committing.
```

## Notes

- **All 5 models must pass** — no exceptions for blocking issues
- Reviews run in **parallel** for speed (~30-60 seconds total)
- Conflicting feedback between models is resolved by favoring the more conservative/secure approach
- The implementing model (current session) addresses all findings
- For trivial changes (typos, comments), a single-model quick review is acceptable — ask the user
- This skill integrates with the `engineering-task` skill as the review phase
