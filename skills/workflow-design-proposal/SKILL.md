---
description: "Design proposal workflow for proposing new designs or redesigns of existing systems. Use this skill when the user says 'propose design', 'redesign', 'design proposal', 'improve architecture', 'migration plan', 'propose improvements', or when a task is tagged 'design-proposal'. Analyzes current state, identifies gaps, and proposes improvements with clear before/after comparisons."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Design Proposal Workflow

## Context
As an architect, you frequently need to evaluate existing systems and propose improvements — whether it's a full redesign, a migration to a new architecture, or targeted improvements to address specific gaps. This workflow bridges analysis and design: you start by deeply understanding what exists, then systematically propose what should change and why.

The output is a proposal document that clearly maps the current state to the proposed state, with explicit gap analysis, improvement rationale, and implementation roadmap.

## When to Use
- Proposing architectural improvements to an existing system
- Planning a migration (monolith → microservices, on-prem → cloud, framework upgrade)
- Redesigning a component or service that has outgrown its original design
- Responding to performance, reliability, or scalability issues with structural solutions
- Creating an improvement roadmap for a team's product

## Workflow

### Phase 1: Current State Analysis

#### 1.1 Understand the Existing System
Use the `workflow-system-docs` skill or its techniques to document the current state:
- **Architecture:** Component diagram, dependency graph, layer structure
- **Data model:** ER diagrams, access patterns, storage technologies
- **APIs:** Endpoint inventory, contracts, integration points
- **Infrastructure:** Deployment topology, scaling approach, environments
- **Observability:** Current monitoring, logging, alerting state

If documentation already exists, validate it against the code — docs may be outdated.

#### 1.2 Stakeholder Requirements
Gather what's driving the change:
```markdown
## Change Drivers
| Driver | Source | Priority |
|--------|--------|----------|
| [e.g., Latency > 500ms at peak] | [Monitoring data] | Critical |
| [e.g., Can't scale beyond 1K RPS] | [Load test results] | High |
| [e.g., Deployment takes 4 hours] | [Team feedback] | Medium |
| [e.g., On-call burden too high] | [Incident reports] | Medium |
```

#### 1.3 Metrics Baseline
Establish measurable baselines for the current system:
```markdown
## Current Performance Baseline
| Metric | Current Value | Target Value | Gap |
|--------|--------------|-------------|-----|
| Latency (p99) | 800ms | < 200ms | 4x improvement needed |
| Throughput | 500 RPS | 5,000 RPS | 10x improvement needed |
| Availability | 99.5% | 99.9% | 4.4h → 8.7h less downtime/year |
| Deploy time | 4 hours | 15 minutes | 16x improvement needed |
| Test coverage | 30% | 80% | 50% gap |
| MTTR | 2 hours | 15 minutes | 8x improvement needed |
```

### Phase 2: Gap Analysis

#### 2.1 Engineering Gaps Assessment
Run a comprehensive audit using the `engineering-checklist` skill framework. Assess each area:

```markdown
## Gap Analysis Summary

### Architecture
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| Tight coupling between services | Deployments require coordination | Shared database, synchronous calls | Event-driven, independent deployments |
| No API versioning | Breaking changes affect consumers | Direct endpoint changes | Versioned APIs with deprecation policy |
| Monolithic deployment | 4-hour deploy, all-or-nothing | Single deployable unit | Independent service deployments |

### Security
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |

### Testing
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |

### Deployment & Release
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |

### Observability
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |

### Performance & Scalability
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |

### Operational Readiness
| Gap | Impact | Current State | Desired State |
|-----|--------|--------------|---------------|
| [gap] | [impact] | [current] | [desired] |
```

#### 2.2 Root Cause Mapping
Create a Mermaid `graph TD` mapping symptoms → intermediate causes → root causes, showing how multiple symptoms trace back to shared architectural root causes.

### Phase 3: Proposed Design

#### 3.1 Design Overview
Present the proposed architecture with clear rationale:

```markdown
## Proposed Architecture

### Design Principles
1. [e.g., Loose coupling — services communicate via events]
2. [e.g., Independent deployability — each service has its own pipeline]
3. [e.g., Observability first — every service emits metrics, logs, traces]
4. [e.g., Fail gracefully — circuit breakers, retries, fallbacks]
```

#### 3.2 Architecture Diagram (Proposed)
Create a Mermaid `graph TD` showing the proposed architecture with all services, databases, caches, event buses, and their connections.

#### 3.3 Before/After Comparison
For each major architectural aspect, show the change:

```markdown
## Before/After Comparison

Create a Mermaid `graph LR` with BEFORE and AFTER subgraphs showing the architectural transformation side-by-side.

### Detailed Comparison
| Aspect | Before | After | Improvement |
|--------|--------|-------|-------------|
| Deployment | Monolithic, 4h | Per-service, 15min | 16x faster |
| Scaling | Vertical only | Horizontal per service | Independent scaling |
| Fault isolation | One failure = total outage | Failure contained to service | Blast radius reduced |
| Database | Shared, tightly coupled | Per-service, right-sized | Data ownership clear |
| Team autonomy | All teams share codebase | Each team owns their service | Parallel development |
| Testing | Full regression required | Service-level testing | Faster feedback |
```

#### 3.4 Component-Level Changes
For each component that changes, document:
```markdown
### [Component Name] Changes

**Current:** [What it does now, how it works]
**Proposed:** [What it will do, how it will work]
**Migration:** [How to get from current to proposed]

#### New Responsibilities
- [New responsibility 1]
- [New responsibility 2]

#### Removed Responsibilities
- [Responsibility moving to another component]

#### Interface Changes
- [API changes, event changes, data contract changes]
```

### Phase 4: Design Best Practices Validation

Validate the proposed design against the `engineering-checklist` quality gates and these design-level concerns:

- [ ] **Scalability:** Stateless services, database scaling strategy, caching with invalidation, async for non-critical paths
- [ ] **Reliability:** No single points of failure, circuit breakers, retry with backoff, graceful degradation, health checks
- [ ] **Security:** Auth at gateway, mTLS between services, secrets in vault, input validation, audit logging
- [ ] **Observability:** Structured logging with correlation IDs, RED metrics, distributed tracing, SLO-based alerting
- [ ] **Deployment:** CI/CD per deployable unit, blue-green/canary capability, automated rollback, feature flags
- [ ] **Data:** Clear data ownership, consistency model documented, backup/recovery strategy, schema evolution plan

For detailed implementation checklists, reference the `engineering-checklist` skill (§1-5).

### Phase 5: Migration Roadmap

#### 5.1 Migration Strategy
Choose an approach:
| Strategy | Risk | Duration | Best When |
|----------|------|----------|-----------|
| **Big bang** | 🔴 High | Short | Small system, downtime acceptable |
| **Strangler fig** | 🟢 Low | Long | Large system, gradual decomposition |
| **Parallel run** | 🟡 Medium | Medium | Data-critical, need validation |
| **Branch by abstraction** | 🟢 Low | Medium | Internal refactoring, same repo |

#### 5.2 Migration Phases
Break the migration into deliverable phases:
```markdown
## Migration Roadmap

### Phase 1: Foundation (Weeks 1-4)
- [ ] Set up CI/CD pipeline
- [ ] Implement observability baseline
- [ ] Create API gateway
- **Risk:** [risks and mitigations]
- **Rollback:** [how to revert this phase]

### Phase 2: Extract [Service A] (Weeks 5-8)
- [ ] Extract service from monolith
- [ ] Implement database-per-service
- [ ] Set up event-driven communication
- **Risk:** [risks and mitigations]
- **Rollback:** [how to revert this phase]

### Phase 3: Extract [Service B] (Weeks 9-12)
...

### Phase N: Decommission Legacy
- [ ] Remove old code paths
- [ ] Migrate remaining data
- [ ] Shut down legacy infrastructure
```

#### 5.3 Risk Register
```markdown
## Risk Register
| Risk | Likelihood | Impact | Mitigation | Owner |
|------|-----------|--------|------------|-------|
| Data loss during migration | Low | Critical | Parallel run with validation | [team] |
| Performance regression | Medium | High | Load test each phase | [team] |
| Extended timeline | High | Medium | Phased approach with independent value | [team] |
```

### Phase 6: Review & Present

#### 6.1 Multi-Model Design Review
Submit the proposal for review using parallel agents:
1. **Architecture fit** — `model: "gemini-3-pro-preview"`: Scalability, patterns, consistency
2. **Gap coverage** — `model: "gpt-5.4"`: Are all identified gaps addressed? Missing risks?
3. **Feasibility** — `model: "claude-sonnet-4.6"`: Is the migration plan realistic? Dependencies?

#### 6.2 Proposal Document
Compile everything into `system-documentation/proposal.md`:
```
system-documentation/
├── proposal.md                    # Full proposal document
├── current-state-architecture.md  # Current system documentation
├── gap-analysis.md                # Detailed gap analysis
├── proposed-architecture.md       # Proposed design with diagrams
├── migration-roadmap.md           # Phased migration plan
└── risk-register.md               # Risk assessment
```

#### 6.3 Presentation Deck
For stakeholder presentation, use the `md2pptx` skill to generate slides from the proposal.

#### 6.4 Follow-Up Tasks
Create implementation tasks from the migration roadmap:
```
DailyPlanner-create_task for each migration phase
Tag with: engineering, system-design (as appropriate)
Link back to the proposal document
```

#### 6.5 Mandatory Completion Review
Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.

#### 6.6 Complete
```
DailyPlanner-complete_task(taskId: "[task_id]", summary: "[proposal complete: N gaps identified, M phases planned]")
```

## Integration Points
- **System Docs:** Use `workflow-system-docs` to document current state
- **System Design:** Reference `workflow-system-design` templates for proposed design
- **Engineering Checklist:** Use `engineering-checklist` for gap analysis framework
- **Architecture Decision:** Document key design decisions with `architecture-decision` skill
- **MD to PPTX:** Generate stakeholder presentations with `md2pptx` skill
- **Impact Tracker:** If tagged "official", document impact on completion

## Graceful Fallback
- If DailyPlanner is unavailable, continue the design workflow without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all proposals and documents locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Always document the current state before proposing changes
2. ✅ Every proposed change must map to a specific gap or improvement
3. ✅ Include before/after comparisons for every major change
4. ✅ Validate against best practices checklist
5. ✅ Migration must be phased with independent rollback per phase
6. ⛔ Don't propose changes without understanding the constraints that led to the current design
7. ⛔ Don't ignore migration risk — a great design with an impossible migration is useless
8. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
9. ✅ Any code produced must be documented and have unit tests
