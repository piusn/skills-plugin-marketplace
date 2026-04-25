---
description: "Universal testing philosophy, coverage requirements, test-driven development approach, and test naming conventions. Applies across all languages and frameworks."
---

# Testing Standards

## Testing Philosophy

- Tests are **NOT optional** — they are a first-class deliverable alongside production code
- Test **behavior**, not implementation details — tests should survive refactors
- Tests are **living documentation** of how the system is expected to behave
- A feature without tests is not complete — it's a prototype
- If you can't write a test for it, the design needs rethinking

## Coverage Requirements

| Category | Minimum Coverage | Notes |
|---|---|---|
| New code | **80%** line coverage | Non-negotiable for new PRs |
| Public APIs | **100%** | Every public method must have at least one test |
| Bug fixes | **100%** of the fix | Every fix includes a regression test that fails without the fix |
| Critical paths | Unit **+** integration | Auth, payments, data mutations need layered coverage |
| Utilities / helpers | **90%+** | Widely-used code has outsized blast radius |

## Existing Tests Are Sacred

- ⛔ **NEVER modify an existing test to make it pass after your change** — this hides regressions
- If a test fails after your change, **your change is wrong**, not the test
- The only exception: the test was genuinely incorrect, which requires:
  1. Explicit explanation of why the test was wrong
  2. The original expected behavior documented
  3. Approval from the code owner or tech lead
- ✅ **DO** run the full test suite before every commit
- ✅ **DO** investigate every test failure — never skip or ignore

## New Code Must Be Tested

For every new piece of code, provide:

- ✅ At least **one happy-path test** per function/method
- ✅ At least **one test per error path** (exceptions, validation failures, timeouts)
- ✅ At least **one test per edge case** (empty inputs, nulls, boundary values, overflow)
- ✅ At least **one test per business rule** (the test name should read like the requirement)
- ❌ Don't ship code that "works in manual testing" — automate it

## Test Structure: Arrange-Act-Assert

Every test follows a clear three-phase structure:

```
// Arrange — Set up the preconditions and inputs
//   Create objects, mock dependencies, prepare test data

// Act — Execute the single action being tested
//   Call the method, trigger the event, make the request

// Assert — Verify the expected outcome
//   Check return values, state changes, side effects, exceptions
```

### Rules

- ✅ **DO** keep each phase visually separated (blank line or comment)
- ✅ **DO** have exactly one logical Act per test
- ✅ **DO** assert on specific expected values, not just "not null"
- ❌ **DON'T** put logic in tests (no if/else, no loops, no try/catch)
- ❌ **DON'T** assert on more than one behavior per test

## Test Naming Convention

### Pattern: `MethodName_Scenario_ExpectedBehavior`

Names should read like documentation — a developer should understand the expected behavior without reading the test body.

**Good Examples:**

```
CalculateTotal_WithEmptyCart_ReturnsZero
CalculateTotal_WithSingleItem_ReturnsItemPrice
CalculateTotal_WithDiscount_AppliesPercentageCorrectly
Login_WithInvalidPassword_ThrowsAuthenticationException
Login_WithLockedAccount_ReturnsAccountLockedError
Login_WithValidCredentials_ReturnsAccessToken
RenderDashboard_WhenDataIsLoading_ShowsSpinner
RenderDashboard_WhenDataLoadFails_ShowsErrorMessage
ParseConfig_WithMissingRequiredField_ThrowsValidationError
```

**Bad Examples:**

```
Test1                          // meaningless
TestCalculateTotal             // no scenario or expectation
ItWorks                        // tells nothing
ShouldReturnCorrectValue       // what value? under what conditions?
```

### Language-Specific Variations

- **C# / .NET:** `MethodName_Scenario_ExpectedBehavior` using `[Fact]` or `[Theory]`
- **JavaScript / TypeScript:** `describe('MethodName', () => { it('should X when Y', ...) })`
- **Python:** `test_method_name_scenario_expected_behavior` (snake_case)
- **Go:** `TestMethodName_Scenario_ExpectedBehavior`

The pattern is consistent across languages — adapt casing to the language convention.

## Test Categories

### Unit Tests

- Test a **single function or method** in isolation
- **No external dependencies** — mock everything outside the unit
- **Fast** — each test completes in under 100ms
- Run on every commit, every CI build
- Make up the majority of your test suite (70%+)

### Integration Tests

- Test **multiple components working together**
- May use test databases, test APIs, in-memory queues
- Slower than unit tests but higher confidence
- Run on every PR, every CI build
- Cover the critical paths through the system (20% of suite)

### End-to-End Tests

- Test **full user flows** from input to output
- Use real (or realistic) infrastructure
- Slowest but highest confidence
- Run on merge to main and before releases
- Cover the most critical user journeys only (10% of suite)

### Regression Tests

- Written **specifically to reproduce a bug** that was fixed
- The test must **fail without the fix** and **pass with it**
- Include a comment referencing the original bug/issue
- Never delete regression tests — they prevent re-introduction

## What NOT to Test

Not everything needs a test. Skip testing:

- ❌ Framework internals — trust that Express routes work, React renders, etc.
- ❌ Third-party library behavior — test YOUR integration, not their code
- ❌ Simple getters/setters with zero logic — no value in `assertEquals(getName(), "name")`
- ❌ Auto-generated code — DTOs, migrations, scaffolding
- ❌ Constants and configuration values — unless there's validation logic

If in doubt, ask: "Would this test catch a real bug?" If no, skip it.

## Test Quality Guidelines

### Tests Must Be

- **Independent** — no shared mutable state between tests; each test sets up its own world
- **Deterministic** — same input always produces same result; no flaky tests
- **Fast** — unit tests under 100ms, integration tests under 5s
- **Readable** — a new team member can understand what's being tested and why
- **Maintainable** — use test helpers and factories to reduce duplication

### Tests Must NOT Be

- ❌ **Flaky** — a flaky test is worse than no test (erodes trust in the suite)
- ❌ **Slow** — slow tests get skipped; keep the feedback loop tight
- ❌ **Coupled to implementation** — testing private methods, internal state, call order
- ❌ **Copy-pasted** — use parameterized tests for variations of the same scenario

## Test Maintenance

- ✅ **DO** delete tests for deleted features — dead tests breed confusion
- ✅ **DO** refactor tests when production code is refactored
- ✅ **DO** update test data when domain models change
- ✅ **DO** treat test code with the same quality standards as production code
- ❌ **DON'T** leave `@skip`, `@ignore`, or `.skip()` tests without a linked issue and deadline
- ❌ **DON'T** keep tests that are permanently disabled — delete or fix them

## Test Data Management

- ✅ **DO** use factories or builders to create test data
- ✅ **DO** use descriptive test data that makes the scenario obvious
- ✅ **DO** clean up test data after each test (use setup/teardown)
- ❌ **DON'T** use production data in tests
- ❌ **DON'T** hardcode magic values without explanation — name your constants
- ❌ **DON'T** share test data across test classes

## Mocking Guidelines

- ✅ **DO** mock external dependencies (APIs, databases, file systems, clocks)
- ✅ **DO** use the simplest mock possible — stubs over spies over full mocks
- ✅ **DO** verify interactions only when the interaction IS the behavior being tested
- ❌ **DON'T** mock the system under test
- ❌ **DON'T** mock value objects or simple data structures
- ❌ **DON'T** create mocks that replicate the implementation (your test becomes a tautology)
