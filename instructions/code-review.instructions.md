---
description: "Code review standards, review checklist, approval requirements, and multi-model review process for ensuring code quality before merging."
---

# Code Review Standards

## Review Philosophy

- Code review is a **quality gate AND a knowledge-sharing tool** — both matter equally
- Review the **design**, not just the syntax
- Be respectful and constructive — **critique code, not people**
- Approve with confidence — if you're unsure, request changes or ask questions
- Every review makes the codebase better AND makes the team smarter

## What Reviewers Must Check

### 1. Correctness

- ✅ Does the code actually do what the PR description claims?
- ✅ Are all edge cases handled (nulls, empty collections, boundary values)?
- ✅ Is the logic sound — trace through the critical paths mentally
- ❌ Don't assume it works because it compiles

### 2. Design & Architecture

- ✅ Does this fit the existing architecture and patterns?
- ✅ Is the abstraction level appropriate — not too abstract, not too concrete?
- ✅ Are responsibilities correctly separated (single responsibility)?
- ✅ Would this design scale to 10x the current load?
- ❌ Don't approve designs that solve today's problem but create tomorrow's tech debt

### 3. Readability & Maintainability

- ✅ Can another engineer understand this code in 6 months without the author's help?
- ✅ Are names descriptive and consistent with the codebase?
- ✅ Is complex logic broken into well-named helper methods?
- ❌ Don't accept clever code over clear code

### 4. Testing

- ✅ Are there tests for the new behavior?
- ✅ Do tests cover happy paths, error paths, and edge cases?
- ✅ Do tests verify **behavior**, not implementation details?
- ✅ Are existing tests still passing and unmodified (unless genuinely wrong)?
- ❌ Don't approve code that lacks tests for new functionality

### 5. Security

- ✅ Is user input validated and sanitized?
- ✅ Are authentication and authorization checks in place?
- ✅ Are secrets handled properly (not hardcoded, not logged)?
- ✅ Is sensitive data encrypted at rest and in transit?
- ✅ Are SQL queries parameterized (no injection)?
- ❌ Don't skip security review — it's everyone's responsibility

### 6. Performance

- ✅ Any N+1 query patterns?
- ✅ Are there unbounded loops or collections that could grow?
- ✅ Is caching used where appropriate?
- ✅ Are database queries indexed?
- ❌ Don't approve code with obvious memory leaks or resource exhaustion risks

### 7. Error Handling

- ✅ Are errors caught, logged with context, and handled gracefully?
- ✅ Are error messages helpful for debugging (include what, where, why)?
- ✅ Do users see friendly messages (not stack traces or internal details)?
- ✅ Are retries implemented for transient failures where appropriate?
- ❌ Don't swallow exceptions silently

### 8. Documentation

- ✅ Are public APIs documented with descriptions, parameters, and return values?
- ✅ Is complex business logic explained with comments?
- ✅ Are non-obvious design decisions documented (why, not what)?
- ✅ Is the README updated if the change affects setup or usage?
- ❌ Don't document the obvious — `// increment i` adds no value

## Review Checklist (Per PR)

Apply this checklist to every PR before approving:

- [ ] Changes match the PR description — no undocumented scope creep
- [ ] No unrelated changes bundled in (refactors, formatting, etc.)
- [ ] Tests added or updated for all changes
- [ ] Existing tests still pass and were not modified to accommodate new code
- [ ] No secrets, credentials, PII, or internal URLs committed
- [ ] Error messages are helpful and user-safe (no stack traces to end users)
- [ ] Logging added for key operations (with appropriate log levels)
- [ ] Breaking changes are documented with migration path
- [ ] All TODOs have linked issues — no orphaned TODOs
- [ ] No debug code left in (`console.log`, `debugger`, `print()`, etc.)
- [ ] Dependencies added are justified and from trusted sources

## Multi-Model Review Process

**All code must be reviewed by multiple AI models before committing.** This catches different classes of issues that a single perspective would miss.

### Required Reviewers (All 5)

| Model | Review Focus | Priority |
|---|---|---|
| **Gemini** | Architecture fit, scalability, design patterns | Structure |
| **Claude Opus** | Deep logic analysis, correctness, completeness | Depth |
| **Claude Sonnet** | Problem-solution fit, readability, edge cases | Balance |
| **Codex** | Code quality, idiomatic patterns, performance | Code |
| **GPT** | Security, error handling, cross-cutting concerns | Safety |

### Review Workflow

1. **Author completes implementation and self-reviews**
2. **Submit for multi-model review** — all 5 models review the diff
3. **Consolidate findings** — merge feedback from all models, deduplicate
4. **Address all issues** — every finding gets a code fix or explicit rationale
5. **Re-review if significant changes** — major fixes trigger another review round
6. **Commit only after all models approve** — no exceptions

### When Multi-Model Review Is Required

- ✅ **Always** — all code changes go through multi-model review before commit
- For trivial changes (typos, comments), a single-model quick review is acceptable

### Review Responsibilities

- **All findings must be addressed by the implementing model (Opus)** — reviewers identify issues, Opus fixes them
- Conflicting feedback between models is resolved by favoring the more conservative/secure approach
- The author (human) has final say on design disagreements between models

## Review Etiquette

### For Reviewers

- ✅ **DO** explain the "why" behind every change request
- ✅ **DO** suggest specific alternatives, not just "this is wrong"
- ✅ **DO** acknowledge good patterns — "Nice use of the strategy pattern here"
- ✅ **DO** use prefixes to signal intent:
  - `nit:` — Non-blocking style suggestion, approve anyway
  - `question:` — Seeking clarification, not necessarily blocking
  - `suggestion:` — Offering an alternative approach, author decides
  - `issue:` — Blocking concern that must be addressed
- ❌ **DON'T** hold PRs hostage for style preferences covered by linters
- ❌ **DON'T** rewrite the author's approach — suggest improvements instead
- ❌ **DON'T** leave drive-by comments without reviewing the full PR

### For Authors

- ✅ **DO** respond to every comment — even if just "Done" or "Acknowledged"
- ✅ **DO** explain your reasoning when you disagree with feedback
- ✅ **DO** update the PR description if the scope evolves during review
- ✅ **DO** re-request review after addressing all comments
- ❌ **DON'T** take feedback personally — it's about the code, not you
- ❌ **DON'T** resolve conversations unilaterally — let the reviewer confirm

## Response Time Expectations

| PR Size | Expected Review Time | Priority |
|---|---|---|
| Small (< 100 lines) | Within 4 hours | High — quick turnaround keeps velocity |
| Medium (100–400 lines) | Within 24 hours | Normal |
| Large (400+ lines) | Within 48 hours | Consider splitting the PR |
| Hotfix / production issue | Immediate | Drop everything |

### Escalation

- PRs without review for **3+ business days** are overdue — escalate to the team
- Review small PRs first to unblock teammates quickly
- If you can't review in time, say so and suggest another reviewer

## Approval Standards

- **1 approval minimum** for standard changes
- **2 approvals** for security-sensitive changes, infrastructure, or shared library changes
- **Tech lead approval** for architectural changes or new patterns
- Approvers must have actually read and understood the code — rubber-stamp approvals are worse than no review
