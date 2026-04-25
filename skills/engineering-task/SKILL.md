---
description: "Full engineering lifecycle for tasks and features — from design through deployment. Use this skill when the user says 'engineering task', 'design feature', 'implement feature', 'full engineering workflow', 'design and build', or 'engineering lifecycle'. Orchestrates multi-model design, review, implementation, and deployment."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Engineering Workflow

## Context
Engineering work follows a rigorous lifecycle: planning → design → review → implementation → code review → testing → deployment. This workflow orchestrates the full process using multiple AI models for design quality and review thoroughness.

**Note:** Workspace setup and task tracking are handled by the `start-task` orchestrator. This workflow focuses on the engineering phases.

## Scope Guard
This workflow is appropriate when:
- The task requires **design decisions** or **architectural changes**
- Multiple components or files are affected
- The change benefits from **multi-model review**
- A design document would help future maintainers

For small bug fixes or patches, use the `workflow-quickfix` workflow instead.
If implementation reveals the need for a broader architectural redesign, pause and consider creating a new task with `workflow-design-proposal`.

## Workflow

### Phase 1: Planning & Analysis
1. **Problem space analysis:**
   - Use `explore` agents to understand the current codebase state
   - Identify affected components and dependencies
   - Map integration points and data flows
   - Check existing test coverage for affected code

2. **Existing test audit:**
   - Any untested code that will be refactored **MUST** get tests first
   - Run the existing test suite to establish a baseline
   - Document current coverage gaps

3. **Check for ADR opportunities:**
   - If this task involves significant architectural choices, suggest the `architecture-decision` skill to document the decision

4. **Create the implementation plan:**
   Before any code is written, create a plan document at `system-documentation/plan.md`:
   ```markdown
   # Implementation Plan: [Feature/Task Title]

   ## Problem Statement
   [Clear definition of the problem being solved]

   ## Proposed Approach
   [High-level approach — what will be built/changed and why]

   ## Scope
   - **In scope:** [what this task covers]
   - **Out of scope:** [what this task does NOT cover]

   ## Tasks Breakdown
   1. [Sub-task 1 — concrete, implementable step]
   2. [Sub-task 2]
   3. [Sub-task 3]

   ## Affected Components
   - [Component/file 1 — what changes]
   - [Component/file 2 — what changes]

   ## Dependencies & Risks
   - [Dependencies on other teams/services]
   - [Known risks and mitigations]

   ## Testing Approach
   - [What will be tested and how]

   ## Definition of Done
   - [Reference DoD from workspace README]
   ```

5. **Mandatory plan review (4-model):**
   Submit the plan for review using parallel `general-purpose` agents:

   | Model | Focus |
   |-------|-------|
   | Gemini 3 Pro (`gemini-3-pro-preview`) | Feasibility, scope completeness, approach fit |
   | Claude Opus 4.6 (`claude-opus-4.6`) | Architecture implications, quality of breakdown |
   | Claude Sonnet 4.6 (`claude-sonnet-4.6`) | Missing requirements, edge cases, risks |
   | GPT 5.4 (`gpt-5.4`) | Security considerations, practical implementation concerns |

   Consolidate feedback, update the plan, and **get user approval before proceeding to design or implementation.**

   ⛔ **No coding without an approved plan.**

### Phase 2: Technical Design (use Opus model)
Use a `general-purpose` agent with `model: "claude-opus-4.6"` for thorough design:

1. **Create technical design document:**
   Save to `system-documentation/technical-design.md`:
   ```markdown
   # Technical Design: [Feature/Task Title]

   ## Problem Statement
   [Clear problem definition]

   ## Proposed Solution
   [Architecture and approach]

   ## Architecture Diagram
   ` ``mermaid
   graph TD
     A[Component A] --> B[Component B]
     B --> C[Database]
   ` ``

   ## Data Flow
   ` ``mermaid
   sequenceDiagram
     participant UI
     participant API
     participant DB
     UI->>API: Request
     API->>DB: Query
     DB-->>API: Results
     API-->>UI: Response
   ` ``

   ## API Changes
   [New/modified endpoints with request/response schemas]

   ## Database Changes
   [Schema migrations if any]

   ## Testing Strategy
   - Unit tests: [what to test]
   - Integration tests: [what to test]
   - E2E validation: [steps for testers]

   ## UI Test Plan (mandatory for UI-facing changes)
   Define the UI tests to be executed via `ui-testing-agent` MCP after implementation:

   | # | Test Scenario | Steps | Expected Result | Auth Required |
   |---|--------------|-------|----------------|---------------|
   | 1 | [e.g., Login flow] | Navigate to /login, enter credentials, click Submit | Dashboard loads, user name displayed | Yes — username/password |
   | 2 | [e.g., Create item] | Click "New", fill form, submit | Item appears in list, success toast shown | Yes — existing session |
   | 3 | [e.g., Error handling] | Submit empty form | Validation errors displayed for required fields | No |
   | 4 | [e.g., Responsive layout] | Resize to mobile viewport | Navigation collapses to hamburger menu | No |

   **For each test, specify:**
   - **Pre-conditions:** What state must exist (data, auth, config)
   - **Selectors:** HTML elements to interact with (`#submit-btn`, `input[name="email"]`, `.nav-menu`)
   - **Assertions:** What to validate (`isVisible()`, `hasText()`, `hasValue()`, `URL contains`)
   - **Auth strategy:** How to authenticate (see UI Testing Authentication below)

   ⛔ **This section is mandatory for any change that affects the UI.** It must be defined during design and executed after code review.

   ## Risks & Mitigations
   [Known risks and how to handle them]
   ```

### Phase 3: Design Review (Multi-Model)
Submit the design for review using parallel `general-purpose` agents:

1. **Architecture review** — `model: "gemini-3-pro-preview"`:
   - Architecture fit and scalability
   - Consistency with existing patterns
   - Performance implications

2. **Edge case review** — `model: "gpt-5.4"`:
   - Implementation approach and edge cases
   - Security and input validation concerns
   - Error handling completeness

3. **Completeness review** — `model: "claude-sonnet-4.6"`:
   - Problem-solution fit
   - Missing requirements or scenarios
   - Testing strategy gaps

**Prompt template for reviewers:**
```
Review this technical design document for [specific focus area].

Design document:
[paste design content]

Check:
1. Does the design solve the stated problem?
2. Are there edge cases or failure modes missed?
3. Is the approach consistent with existing architecture?
4. Are there security or performance concerns?
5. Is the testing strategy comprehensive?

Provide specific, actionable feedback only. No style comments.
```

Consolidate feedback, update the design document, and get user approval before proceeding.

### Phase 4: Implementation (use Opus model)
Once design is approved:

1. **Create feature branch:**
   Check existing branches for naming patterns. Use the convention:
   ```
   git checkout -b feature/{integer-task-id}-{short-description}
   ```
   Example: `feature/142-build-email-template`
   
   For bug fixes use: `fix/{integer-task-id}-{short-description}`
   
   ⛔ **NEVER push directly to main.** All work goes through feature/fix branches.

2. **Implement per design:**
   - Follow the approved design document precisely
   - Write unit tests alongside code (TDD when possible)
   - Follow existing code conventions and patterns
   - Document any deviations from design with rationale

3. **Testing strategy:**

   **Unit tests (required for all new code):**
   - Test each function/method in isolation
   - Cover happy path, edge cases (null, empty, boundary), and error paths
   - Use mocks/stubs for external dependencies, not for the code under test
   - Aim for 80%+ coverage on new code

   **Integration tests (required for API/DB changes):**
   - Test API endpoints end-to-end with realistic data
   - Verify database interactions (CRUD, transactions, constraints)
   - Test authentication/authorization flows
   - Verify error responses (status codes, error bodies)

   **E2E tests (recommended for user-facing changes):**
   - Cover critical user journeys
   - Validate UI interactions if applicable

   **Rules:**
   - All new code must have unit tests
   - Existing code being modified must have tests before modification
   - **NEVER modify an existing test to make it pass — ask the user first**
   - Tests must be deterministic — no flaky tests

4. **Progress logging:**
   ```
   DailyPlanner-add_activity_log(taskId: "[task_id]", log: "Implementation complete: [summary]")
   ```

### Phase 5: Code Review (Multi-Model)
Before committing, run parallel `code-review` agents:

1. **Code quality** — `model: "gemini-3-pro-preview"`:
   - Code patterns and conventions
   - Readability and maintainability
   
2. **Logic and correctness** — `model: "claude-sonnet-4.6"`:
   - Logic errors and edge cases
   - Test comprehensiveness

3. **Security and performance** — `model: "gpt-5.4"`:
   - Security vulnerabilities
   - Performance bottlenecks
   - Resource management

**All fixes are implemented by Opus** (use `general-purpose` agent with `model: "claude-opus-4.6"`).

### Phase 5b: UI Testing (Inner Loop)
After code review passes, validate the UI using the `ui-testing-agent` MCP. This is **mandatory for any change that affects the UI**.

#### 1. Start Local Environment
Before testing, start the backend and frontend locally:
```
# Start backend (e.g., API server)
[project-specific command — e.g., dotnet run, npm run dev:api, python manage.py runserver]

# Start frontend (e.g., React/Next.js dev server)
[project-specific command — e.g., npm run dev, npm start]
```
Wait for both to be healthy before proceeding.

#### 2. Execute UI Test Plan
Run each test from the UI Test Plan defined in the design document using `ui-testing-agent`:

```
ui-testing-agent: Launch browser at [local URL]
```

For each test scenario:
1. **Navigate** to the target page
2. **Authenticate** if required (see auth strategies below)
3. **Execute** the test steps (click, type, select, scroll)
4. **Validate** assertions (element visible, text content, URL, state changes)
5. **Capture** screenshot on failure for debugging

#### 3. Authentication Strategies
Handle different auth scenarios:

| Strategy | When to Use | How |
|----------|-----------|-----|
| **Username/Password** | Local dev with test accounts | Use `ui-testing-agent` to fill login form with test credentials |
| **Token injection** | API-first auth | Generate token via API call, inject into browser localStorage/cookies |
| **Existing session** | Session-based auth | Login once, reuse session across tests |
| **Google/OAuth login** | OAuth required | ⚠️ Prompt user: "Google login required — please complete the login in the browser, then I'll continue testing" |
| **Service account** | CI/CD pipelines | Use service principal or API key — no interactive login needed |

**For Google/OAuth in automated environments (GitHub Actions, Azure DevOps):**
> 💡 **Recommendations for CI/CD:**
> - Use **service accounts** with API keys instead of OAuth where possible
> - Set up a **test user with username/password auth** that bypasses OAuth for testing
> - Use **auth token injection** — generate tokens via API/CLI, inject into the test browser
> - Consider **mock auth middleware** in test environments that auto-authenticates with a test identity
> - For Azure DevOps: use **Managed Identity** or **Service Principal** tokens
> - Store test credentials in **pipeline secrets** (GitHub Secrets / Azure Key Vault), never in code

#### 4. Test Result Handling
After all tests complete:

```markdown
## 🧪 UI Test Results

| # | Scenario | Status | Notes |
|---|----------|--------|-------|
| 1 | Login flow | ✅ Pass | — |
| 2 | Create item | ✅ Pass | — |
| 3 | Error handling | ❌ Fail | Validation error not shown for empty email |
| 4 | Responsive layout | ✅ Pass | — |

**Overall: 3/4 passed**
```

- **All pass:** Proceed to Phase 6 (Ship)
- **Failures:** Fix the issues, re-run code review for the fixes, then re-run UI tests
- ⛔ **Do NOT proceed to shipping with failing UI tests**

#### 5. Feedback on `ui-testing-agent` MCP
Since the MCP is in active development, capture improvement feedback:
- Note any issues encountered (timing, selectors, auth problems)
- Suggest new capabilities needed (e.g., visual regression, accessibility checks, performance metrics)
- Log feedback:
  ```
  DailyPlanner-add_activity_log(taskId: "[task_id]", log: "ui-testing-agent feedback: [issue/suggestion]")
  ```

### Phase 7: Knowledge Capture

Before closing this task, extract reusable knowledge from the session:

1. **Invoke the `session-knowledge` skill** to scan the session for:
   - Deployment procedures discovered or used
   - Environment configurations and setup steps
   - Debugging patterns and troubleshooting steps
   - Architectural decisions and patterns established
   - New tool usage, SDK patterns, or API conventions

2. The skill will persist findings as:
   - **Copilot instruction files** (global or repo-specific)
   - **Skill updates** (if an existing skill should be enhanced)
   - **New skills** (if a repeatable multi-step workflow was established)
   - **Notion team page documentation** (for team-specific operational knowledge)

3. This ensures future engineering tasks in the same repo or domain automatically benefit from the knowledge gained in this session.

> **Note:** Knowledge capture is particularly valuable after engineering tasks that involve deployment, infrastructure, or new integrations.
  ```

### Phase 6: Ship

1. **Run the `engineering-checklist` quality gate:**
   Before shipping, invoke the `engineering-checklist` skill or manually verify:
   - Security: input validation, auth, data protection, dependency vulnerabilities
   - Testing: all tests pass, coverage adequate, no flaky tests
   - Monitoring: metrics, logging, and alerts configured for new code paths

2. **Final build and test pass** — all tests must pass

3. **Commit with descriptive message:**
   ```
   git add -A && git commit -m "feat: [short description]

   [Detailed description of changes]
   [Link to design doc if applicable]

   Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
   ```

4. **Run mandatory 4-model completion review:**
   Run the multi-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address all critical findings before proceeding.

5. **Push to feature branch:**
   ```
   git push -u origin feature/{integer-task-id}-{short-description}
   ```

6. **Create Pull Request to main:**
   - Create a PR from the feature branch to `main`
   - Include in the PR description: task summary, changes made, testing done, DoD checklist status
   - ⛔ **Do NOT auto-merge.** The PR requires user review.

7. **Prompt user to review the PR:**
   ```
   ✅ Pull request created: {PR URL}
   
   📋 Please review the PR when ready. You can:
   - Approve and merge manually on GitHub
   - Ask me to merge it for you after your review
   ```
   **Wait for user response.** Do not proceed until the user has reviewed.

8. **Post-merge: Switch to main:**
   Once the PR is merged (by user or on user's instruction):
   ```
   git checkout main && git pull origin main
   ```
   Inform the user:
   ```
   ✅ Merged and switched to main branch.
   ⚠️ Do not continue building on the merged feature branch.
   ```
   ⛔ **Do NOT continue work on the merged branch** unless the user explicitly consents.

9. **Rollout & deployment (if applicable):**
   - Follow the `engineering-checklist` Deployment & Rollout section (§3)
   - Choose rollout strategy based on risk level
   - Monitor key metrics during and after deployment

10. **Complete the task:**
    ```
    DailyPlanner-complete_task(taskId: "[task_id]", summary: "[what was done]", testingInstructions: "[how to validate]")
    ```

## Integration Points
- **Engineering Checklist:** Invoke `engineering-checklist` skill as quality gate before shipping
- **Impact Tracker:** If task is tagged "official", invoke `impact-tracker` skill at start and completion
- **Architecture Decision:** Suggest `architecture-decision` skill when significant design choices are made
- **Tech Docs:** Suggest `tech-docs` skill if the feature needs documentation
- **Activity Log:** Log progress at each phase via `DailyPlanner-add_activity_log`

## Graceful Fallback
- If DailyPlanner is unavailable, continue the engineering workflow without task tracking — warn the user
- If Notion is unavailable, save all documents locally in the workspace
- If a review model is unavailable, proceed with available models and note the gap

## Critical Rules
1. ⛔ **No coding without an approved plan** — plan must be 4-model reviewed and user-approved
2. ⛔ **NEVER change an existing test to make it pass without explicit user approval**
3. ⛔ **No code ships without multi-model review**
4. ⛔ **All code must be documented** — functions, classes, modules need docstrings; non-obvious logic needs inline comments
5. ✅ **All plans, designs, and documentation live in `system-documentation/`** in the repository
6. ✅ **All new code must have comprehensive unit tests** (happy path, edge cases, error paths, 80%+ coverage)
7. ✅ **Design document must exist before implementation starts**
8. ✅ **Staging validation before production merge**
9. ✅ **Design documents are living docs — update them when implementation deviates**
10. ✅ **Mandatory 4-model completion review** before marking task done (see `start-task` Critical Rules §3)
