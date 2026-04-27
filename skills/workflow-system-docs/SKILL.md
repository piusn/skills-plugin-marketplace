---
description: "System documentation workflow for reverse-engineering and documenting existing codebases and products. Use this skill when the user says 'document this system', 'reverse engineer docs', 'map this codebase', 'system documentation', 'architecture docs', 'document existing system', or when a task is tagged 'system-docs'. Produces comprehensive markdown documentation with Mermaid diagrams from code analysis."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# System Documentation Workflow

## Context
As an architect, you often encounter products and services with sparse or missing documentation. This workflow provides a structured approach to analyzing an existing codebase and producing comprehensive system documentation — architecture diagrams, component maps, data flows, API surfaces, and dependency graphs — all generated from what the code actually does, not what someone thinks it does.

The output is a set of markdown files with Mermaid diagrams that serve as living documentation.

## When to Use
- Documenting an undocumented or poorly documented system
- Onboarding onto a new codebase and producing reference docs
- Creating architecture documentation for an existing product
- Auditing a system's actual architecture vs. its intended design
- Preparing documentation for a system handoff or knowledge transfer

## Workflow

### Phase 1: Reconnaissance

Before diving deep, get the lay of the land.

#### 1.1 Repository Survey
Use `explore` agents to answer these questions:
- What language(s) and frameworks are used?
- What is the project structure? (monorepo, microservices, layers)
- What build system and package manager are used?
- Is there a CI/CD pipeline? What does it do?
- Are there any existing docs (README, wiki, /docs, comments)?

Save initial findings to `system-documentation/00-overview.md`:
```markdown
# System Overview: [Product Name]

## Quick Facts
| Attribute | Value |
|-----------|-------|
| Language(s) | [e.g., TypeScript, C#, Python] |
| Framework(s) | [e.g., Express, ASP.NET Core, Flask] |
| Package Manager | [e.g., npm, NuGet, pip] |
| Build System | [e.g., webpack, MSBuild, Make] |
| Repository Type | [Monorepo / Multi-repo / Single service] |
| CI/CD | [e.g., GitHub Actions, Azure DevOps] |
| Hosting | [e.g., Azure App Service, AKS, AWS Lambda] |

## Repository Structure
` ``
project-root/
├── src/             # [description]
├── tests/           # [description]
├── docs/            # [description]
├── config/          # [description]
└── ...
` ``

## Existing Documentation
- [List any existing docs found, with assessment of completeness]
```

#### 1.2 Entry Points
Identify the system's entry points:
- **Web servers:** Where does the app start? (main, index, Program.cs, app.py)
- **API routes:** Where are routes/controllers defined?
- **Background jobs:** Are there workers, cron jobs, event handlers?
- **CLI tools:** Any command-line interfaces?
- **Event consumers:** Message queue subscribers, webhook handlers?

### Phase 2: Architecture Discovery

#### 2.1 Component Mapping
Use `explore` agents to trace the codebase and identify logical components:

For each component, document:
```markdown
### [Component Name]
- **Location:** `src/[path]/`
- **Responsibility:** [What this component does]
- **Key files:** [Most important files]
- **External dependencies:** [APIs, services, databases it talks to]
- **Internal dependencies:** [Other components it depends on]
```

Produce a Mermaid `graph TD` component diagram with subgraphs for each layer (frontend, API, business logic, data) showing all discovered components and their connections.

#### 2.2 Dependency Graph
Create a Mermaid `graph LR` showing internal module dependencies and shared libraries.

Also document external dependencies:
```markdown
## External Dependencies
| Dependency | Version | Purpose | Risk Level |
|-----------|---------|---------|------------|
| [package-name] | [version] | [what it does] | [Low/Medium/High] |
```

#### 2.3 Layer Architecture
Create a Mermaid `graph TD` with subgraphs for each architectural layer (presentation, business, data access, infrastructure) showing the components in each layer and cross-layer dependencies.

### Phase 3: API Surface Documentation

#### 3.1 REST/HTTP APIs
For each API endpoint, document:
```markdown
## API Reference

### [Resource Group]

#### `GET /api/v1/[resource]`
- **Purpose:** [Description]
- **Auth:** [Required auth level]
- **Query Params:** [Pagination, filters]
- **Response:** [Shape of response]
- **Status Codes:** 200, 400, 401, 404

#### `POST /api/v1/[resource]`
- **Purpose:** [Description]
- **Auth:** [Required auth level]
- **Request Body:** [Shape of request]
- **Response:** [Shape of response]
- **Status Codes:** 201, 400, 401, 409
```

#### 3.2 Event/Message Contracts
If the system uses messaging:
```markdown
## Events & Messages

### [Event Name]
- **Producer:** [Service/component that emits this]
- **Consumer(s):** [Service(s) that handle this]
- **Channel:** [Queue/topic name]
- **Payload:**
  ```json
  {
    "eventType": "OrderCreated",
    "orderId": "string",
    "timestamp": "ISO-8601"
  }
  ```
- **Retry policy:** [retry count, DLQ]
```

#### 3.3 Internal APIs / Interfaces
Document key internal interfaces, abstract classes, or contracts:
```markdown
## Internal Contracts

### IOrderService
- `createOrder(input: CreateOrderInput): Order`
- `getOrder(id: string): Order | null`
- `cancelOrder(id: string): void`

Implemented by: `OrderService` (`src/services/order.service.ts`)
```

### Phase 4: Data Model Documentation

#### 4.1 Database Schema
Extract and document the data model as a Mermaid `erDiagram` showing all entities, their fields (with PK/FK/UK annotations), and relationships.

#### 4.2 Data Flow Diagrams
For each critical business flow, create a Mermaid `sequenceDiagram` showing how data moves through all participants (client, API, services, DB, queues, workers, external systems).

#### 4.3 State Machines
If entities have lifecycle states, create a Mermaid `stateDiagram-v2` showing all states and transitions with trigger methods.

### Phase 5: Infrastructure & Operations

#### 5.1 Deployment Architecture
Create a Mermaid `graph TD` showing the deployment topology with subgraphs for cloud provider/environment (production instances, databases, caches) and supporting infrastructure (CI/CD, monitoring, logging).

#### 5.2 Configuration & Secrets
```markdown
## Configuration

### Environment Variables
| Variable | Purpose | Required | Default |
|----------|---------|----------|---------|
| `DATABASE_URL` | Database connection string | Yes | — |
| `REDIS_URL` | Cache connection | No | localhost:6379 |
| `LOG_LEVEL` | Logging verbosity | No | info |

### Secrets
| Secret | Store | Rotation |
|--------|-------|----------|
| DB password | Key Vault | 90 days |
| API keys | Key Vault | On demand |
```

#### 5.3 Monitoring & Health
Document what's currently monitored (or what should be):
```markdown
## Observability

### Current State
| Aspect | Status | Tool | Notes |
|--------|--------|------|-------|
| Metrics | ✅ / ⚠️ / ❌ | [tool] | [notes] |
| Logging | ✅ / ⚠️ / ❌ | [tool] | [notes] |
| Tracing | ✅ / ⚠️ / ❌ | [tool] | [notes] |
| Alerting | ✅ / ⚠️ / ❌ | [tool] | [notes] |

### Gaps & Recommendations
- [What's missing and what should be added]
```

### Phase 6: Compile & Review

#### 6.1 Document Structure
Organize the output as a documentation set:
```
system-documentation/
├── 00-overview.md           # System overview, quick facts, repo structure
├── 01-architecture.md       # Component map, layers, dependency graph
├── 02-api-reference.md      # REST APIs, events, internal contracts
├── 03-data-model.md         # ER diagrams, data flows, state machines
├── 04-infrastructure.md     # Deployment, config, monitoring
├── 05-engineering-audit.md  # Gap analysis across all engineering concerns
└── diagrams/                # Exported diagram images (if needed)
```

#### 6.2 Cross-Reference Validation
- Verify diagrams match the actual code (use `explore` agents to spot-check)
- Ensure all components mentioned in architecture appear in API/data docs
- Check for orphaned components (code that's unused or dead)
- Identify undocumented integration points

#### 6.3 Engineering Audit & Gap Analysis
Produce a comprehensive audit across all engineering-centric aspects. Save to `system-documentation/05-engineering-audit.md`:

```markdown
# Engineering Audit: [Product Name]

## Audit Summary
| Area | Health | Critical Gaps | Notes |
|------|--------|--------------|-------|
| Architecture | 🟢 / 🟡 / 🔴 | [count] | |
| Security | 🟢 / 🟡 / 🔴 | [count] | |
| Testing | 🟢 / 🟡 / 🔴 | [count] | |
| Deployment | 🟢 / 🟡 / 🔴 | [count] | |
| Observability | 🟢 / 🟡 / 🔴 | [count] | |
| Data Integrity | 🟢 / 🟡 / 🔴 | [count] | |
| Performance | 🟢 / 🟡 / 🔴 | [count] | |
| Operational Readiness | 🟢 / 🟡 / 🔴 | [count] | |

---

### 1. Architecture Gaps
- [ ] Circular dependencies between components
- [ ] Missing abstraction layers (business logic in controllers, SQL in routes)
- [ ] Tight coupling to specific infrastructure (hardcoded cloud provider calls)
- [ ] Dead code or orphaned modules
- [ ] Missing service boundaries (monolith candidates for decomposition)
- [ ] No clear API versioning strategy
- [ ] Missing error handling patterns (no global error handler, inconsistent responses)

### 2. Security Gaps
- [ ] Endpoints without authentication/authorization
- [ ] Hardcoded secrets, API keys, or connection strings in code
- [ ] Missing input validation or SQL injection vectors
- [ ] No CSRF/XSS protection on user-facing endpoints
- [ ] Missing rate limiting on public APIs
- [ ] Dependencies with known vulnerabilities (check Dependabot/Snyk)
- [ ] No audit logging for security-relevant operations
- [ ] Missing encryption (data at rest or in transit)
- [ ] Overly permissive CORS configuration
- [ ] No security headers (CSP, HSTS, X-Frame-Options)

### 3. Testing Gaps
- [ ] Overall test coverage percentage: [X%]
- [ ] Components with zero test coverage: [list]
- [ ] Missing unit tests for business logic
- [ ] Missing integration tests for API endpoints
- [ ] Missing contract tests for external service integrations
- [ ] No E2E test suite for critical user journeys
- [ ] No performance/load test suite
- [ ] Flaky or disabled tests: [count]
- [ ] Test data management strategy missing
- [ ] No test isolation (tests depend on shared state or ordering)

### 4. Deployment & Release Gaps
- [ ] No CI/CD pipeline or pipeline is incomplete
- [ ] Missing staging/pre-production environment
- [ ] No automated deployment — manual steps required
- [ ] No rollback procedure documented or tested
- [ ] Database migrations not reversible
- [ ] No feature flag infrastructure for gradual rollouts
- [ ] Missing health check endpoints (liveness/readiness)
- [ ] No blue-green or canary deployment capability
- [ ] Container images not scanned for vulnerabilities
- [ ] Deployment requires downtime

### 5. Observability Gaps
- [ ] No structured logging (or inconsistent log formats)
- [ ] No correlation IDs across service boundaries
- [ ] Missing application metrics (request rate, error rate, latency)
- [ ] No distributed tracing
- [ ] No alerting configured (or alerts go to unmonitored channels)
- [ ] No dashboards for service health
- [ ] Sensitive data leaking into logs (PII, passwords, tokens)
- [ ] No SLOs/SLIs defined
- [ ] Missing resource utilization monitoring (CPU, memory, disk, connections)
- [ ] No anomaly detection or trend analysis

### 6. Data Integrity Gaps
- [ ] Missing database constraints (foreign keys, unique, check)
- [ ] No data validation at API boundary
- [ ] Missing database indexes for common query patterns
- [ ] No backup strategy or backup verification
- [ ] No data retention/archival policy
- [ ] Missing transaction boundaries (partial writes possible)
- [ ] No optimistic/pessimistic concurrency control where needed
- [ ] Schema migrations not version-controlled

### 7. Performance Gaps
- [ ] No caching strategy (or cache without invalidation)
- [ ] N+1 query patterns in data access layer
- [ ] Missing database connection pooling
- [ ] No pagination on list endpoints (unbounded queries)
- [ ] Large payloads without compression
- [ ] Synchronous calls where async would be appropriate
- [ ] No CDN for static assets
- [ ] Missing resource limits (memory, CPU, connection caps)
- [ ] No load testing or performance baselines established

### 8. Operational Readiness Gaps
- [ ] No runbook or troubleshooting guide (TSG)
- [ ] No on-call rotation or escalation path
- [ ] Missing disaster recovery plan
- [ ] RTO/RPO not defined
- [ ] No capacity planning or auto-scaling rules
- [ ] Missing documentation for manual operational procedures
- [ ] No chaos engineering or failure testing
- [ ] Configuration changes require code deployment

---

## Prioritized Recommendations

### 🔴 Critical (address immediately)
1. [Most critical gap with rationale]
2. [Second critical gap]

### 🟡 Important (address within sprint)
1. [Important gap]
2. [Important gap]

### 🟢 Improvement (backlog)
1. [Nice-to-have improvement]
2. [Nice-to-have improvement]

## Follow-Up Tasks
Create engineering tasks for each critical and important gap:
| Gap | Task Type | Priority | Workflow |
|-----|-----------|----------|----------|
| [Gap description] | [fix/feature/infra] | [P0/P1/P2] | [engineering/quickfix/system-design] |
```

#### 6.4 Review
- Use a `code-review` agent to validate documentation accuracy against code
- Share with team for feedback
- **Run mandatory completion review:**
  Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.
- If related to a Daily Planner task, complete it:
  ```
  DailyPlanner-complete_task(taskId: "[task_id]", summary: "[system documented: N components, M diagrams, K pages]")
  ```

## Exploration Strategies

### For Large Codebases
Launch multiple `explore` agents in parallel, each focused on a different area:
1. Agent 1: Entry points, routing, middleware
2. Agent 2: Business logic, services, domain models
3. Agent 3: Data access, database schema, migrations
4. Agent 4: Infrastructure, config, deployment
5. Agent 5: External integrations, SDKs, API clients

### For Unfamiliar Tech Stacks
- Start with `package.json` / `*.csproj` / `requirements.txt` to understand dependencies
- Check CI/CD config (`.github/workflows/`, `azure-pipelines.yml`) for build/deploy context
- Look for Docker/container configs for runtime dependencies
- Read test files — they often reveal intended behavior better than source code

### Useful Mermaid Diagram Types
| Diagram | Best For | Mermaid Type |
|---------|----------|-------------|
| System architecture | Component overview | `graph TD` |
| Data relationships | Database schema | `erDiagram` |
| Request flows | API interactions | `sequenceDiagram` |
| State lifecycles | Entity states | `stateDiagram-v2` |
| Process flows | Business processes | `flowchart` |
| Class hierarchies | OOP structure | `classDiagram` |
| Timelines | Deployment history | `timeline` |

## Integration Points
- **Engineering Checklist:** Reference `engineering-checklist` for ops readiness assessment
- **Tech Docs:** Use `tech-docs` skill for publishing to eng.ms or DocFX
- **Architecture Decision:** Create ADRs for significant findings using `architecture-decision` skill
- **Onboarding Guide:** Feed documentation into `onboarding-guide` skill for new team members
- **Markdown to Word:** Export with `markdown-to-word` for stakeholder distribution
- **MD to PPTX:** Create architecture presentations with `md2pptx` skill

## Graceful Fallback
- If DailyPlanner is unavailable, continue the documentation workflow without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all documentation locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Document what the code **actually does**, not what you think it should do
2. ✅ Every diagram must be validated against the code
3. ✅ Include a gap analysis — documentation should reveal problems, not hide them
4. ✅ Use parallel `explore` agents for large codebases — don't go sequentially
5. ✅ Start with the big picture, then drill into details
6. ⛔ Don't guess about behavior — trace the code or flag it as unknown
7. ⛔ Don't document dead code as active — flag it for removal
8. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
9. ✅ Any code produced must be documented and have unit tests
