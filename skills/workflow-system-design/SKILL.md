---
description: "System design workflow for architecting new systems, services, and large-scale features. Use this skill when the user says 'system design', 'design system', 'architect this', 'design a service', 'scalability design', or when a task is tagged 'system-design'. Guides through requirements, capacity planning, API design, data modeling, architecture, and trade-off analysis."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# System Design Workflow

## Context
System design tasks go beyond feature-level engineering. They require thinking about the full picture — scale, reliability, data flow, API contracts, storage choices, and operational concerns. This workflow provides a structured framework for designing systems from scratch or redesigning existing ones, producing comprehensive design documents that guide implementation.

## When to Use
- Designing a new service, system, or platform
- Redesigning or scaling an existing system
- Breaking a monolith into microservices
- Designing data pipelines or event-driven architectures
- Any task requiring end-to-end architectural thinking
- System design practice or interview preparation

## Workflow

### Phase 1: Requirements & Scope

#### 1.1 Functional Requirements
Clarify what the system must do:
- **Core use cases:** What are the primary user actions?
- **User roles:** Who interacts with the system?
- **Input/Output:** What data flows in and out?
- **Business rules:** What constraints or policies apply?

Template:
```markdown
## Functional Requirements
| ID | Requirement | Priority | Notes |
|----|------------|----------|-------|
| FR-1 | [User can do X] | Must-have | |
| FR-2 | [System processes Y] | Must-have | |
| FR-3 | [Admin can configure Z] | Nice-to-have | |
```

#### 1.2 Non-Functional Requirements
Define quality attributes:
- **Availability:** What uptime is required? (e.g., 99.9% = ~8.7h downtime/year)
- **Latency:** What response times are acceptable? (p50, p95, p99)
- **Throughput:** How many requests/events per second?
- **Consistency:** Strong vs. eventual consistency?
- **Durability:** What data loss is acceptable?
- **Security:** Authentication, authorization, encryption requirements?
- **Compliance:** Regulatory requirements (GDPR, SOC2, etc.)?

Template:
```markdown
## Non-Functional Requirements
| Attribute | Target | Rationale |
|-----------|--------|-----------|
| Availability | 99.9% | Business-critical service |
| Latency (p99) | < 200ms | User-facing API |
| Throughput | 10K req/s | Peak traffic estimate |
| Data durability | 99.999999% | Financial data |
```

### Phase 2: Capacity Estimation

#### 2.1 Traffic Estimates
Calculate expected load:
```markdown
## Traffic Estimates
- DAU (Daily Active Users): [X]
- Read:Write ratio: [e.g., 10:1]
- Requests per user per day: [Y]
- Peak multiplier: [e.g., 3x average]

### Calculations
- Daily requests: DAU × requests/user = [Z] req/day
- Average QPS: Z / 86,400 = [A] req/s
- Peak QPS: A × peak_multiplier = [B] req/s
```

#### 2.2 Storage Estimates
Calculate data growth:
```markdown
## Storage Estimates
- Average object size: [X] KB
- New objects per day: [Y]
- Retention period: [Z] years

### Calculations
- Daily storage: X × Y = [A] GB/day
- Annual storage: A × 365 = [B] TB/year
- Total (with retention): B × Z = [C] TB
```

#### 2.3 Bandwidth Estimates
```markdown
## Bandwidth Estimates
- Incoming: [QPS × avg_request_size]
- Outgoing: [QPS × avg_response_size]
- Peak bandwidth: [peak_QPS × max_response_size]
```

### Phase 3: API Design

#### 3.1 Define API Contracts
For each core operation, define the contract:
```markdown
## API Endpoints

### POST /api/v1/{resource}
- **Purpose:** Create a new [resource]
- **Auth:** Bearer token (JWT)
- **Request Body:**
  ```json
  {
    "field1": "string (required)",
    "field2": 123
  }
  ```
- **Response (201):**
  ```json
  {
    "id": "uuid",
    "field1": "string",
    "createdAt": "ISO-8601"
  }
  ```
- **Error Responses:** 400 (validation), 401 (unauth), 409 (conflict)
- **Rate Limit:** 100 req/min per user
```

#### 3.2 API Design Principles
- Use RESTful conventions (or document why not)
- Version APIs (`/api/v1/`)
- Pagination for list endpoints (cursor-based for large datasets)
- Idempotency keys for write operations
- Consistent error response format

### Phase 4: Data Model & Storage

#### 4.1 Data Model Design
Define entities, relationships, and access patterns:
```markdown
## Data Model

### Entities
| Entity | Key Fields | Relationships |
|--------|-----------|---------------|
| User | id, email, name, role | Has many Orders |
| Order | id, userId, status, total | Belongs to User, has many Items |
| Item | id, orderId, productId, qty | Belongs to Order |

### Access Patterns
| Pattern | Query | Frequency | Latency SLA |
|---------|-------|-----------|-------------|
| Get user by ID | PK lookup | Very high | < 10ms |
| List user orders | userId index | High | < 50ms |
| Search orders by date range | GSI on createdAt | Medium | < 200ms |
```

#### 4.2 Storage Technology Selection
Evaluate options based on access patterns:

| Criteria | SQL (PostgreSQL) | NoSQL (CosmosDB/DynamoDB) | Cache (Redis) |
|----------|-----------------|--------------------------|---------------|
| Complex queries | ✅ Excellent | ⚠️ Limited | ❌ Not suited |
| Horizontal scale | ⚠️ With sharding | ✅ Built-in | ✅ Clustered |
| Transactions | ✅ ACID | ⚠️ Limited | ❌ No |
| Latency | Good (5-20ms) | Good (5-10ms) | Excellent (<1ms) |
| Cost at scale | Moderate | Pay-per-use | Memory-bound |

Document the choice and rationale.

### Phase 5: High-Level Architecture

#### 5.1 Architecture Diagram
Create a Mermaid `graph TD` showing all components (clients, load balancer, API gateway, services, databases, caches, queues, workers) and their connections.

#### 5.2 Component Responsibilities
For each component, document:
```markdown
## Components

### API Gateway
- **Responsibility:** Request routing, rate limiting, auth validation
- **Technology:** [e.g., Azure API Management, Kong]
- **Scaling:** Horizontal, auto-scaled based on CPU/request count

### Service A
- **Responsibility:** [Core business logic]
- **Technology:** [e.g., .NET 8, Node.js]
- **Dependencies:** Primary DB, Redis Cache, Message Queue
- **Scaling:** Horizontal, stateless
```

#### 5.3 Data Flow
For each critical path, create a Mermaid `sequenceDiagram` showing the request flow through all components (client → gateway → service → DB/queue → workers), including async processing.

### Phase 6: Deep Dives

Pick 2-3 critical areas for detailed design:

#### 6.1 Scalability
- **Horizontal scaling strategy:** Stateless services, database sharding, read replicas
- **Caching strategy:** Cache-aside, write-through, TTL policies
- **CDN/Edge:** Static content, geographic distribution
- **Database scaling:** Read replicas, partitioning, connection pooling

#### 6.2 Reliability & Fault Tolerance
- **Redundancy:** Multi-AZ, multi-region
- **Circuit breakers:** Prevent cascade failures
- **Retry policies:** Exponential backoff with jitter
- **Graceful degradation:** What features can be disabled under load?
- **Health checks:** Liveness vs. readiness probes

#### 6.3 Monitoring & Observability
- **Metrics:** Latency, error rate, throughput, saturation (USE/RED methods)
- **Logging:** Structured logging, correlation IDs
- **Tracing:** Distributed tracing across services
- **Alerting:** SLO-based alerts, PagerDuty/on-call integration

#### 6.4 Security
Design-level security considerations:
- Authentication and authorization architecture (OAuth2/OIDC, mTLS between services)
- Data protection strategy (encryption at rest and in transit)
- Threat modeling using STRIDE framework
- Audit logging architecture
- Reference the `engineering-checklist` §1 for implementation-level security checklist

#### 6.5 Deployment & Rollout Strategy
- Deployment pipeline design (CI/CD stages)
- Rollout pattern selection (rolling/blue-green/canary/feature flags)
- Rollback strategy and automation triggers
- Zero-downtime deployment requirements (backward-compatible APIs, expand-contract migrations)
- Environment strategy (dev → staging → canary → production)
- Reference the `engineering-checklist` §3 for deployment checklist

#### 6.6 Testing Strategy
- Testing pyramid for the system (unit/integration/E2E/performance)
- Contract testing strategy for service integrations
- Chaos engineering approach (failure injection)
- Reference the `engineering-checklist` §2 for testing checklist

### Phase 7: Trade-off Analysis

Document key decisions and their trade-offs:
```markdown
## Trade-off Analysis

### Decision 1: [e.g., SQL vs NoSQL for primary storage]
| Factor | Option A: SQL | Option B: NoSQL |
|--------|--------------|-----------------|
| Consistency | ✅ Strong ACID | ⚠️ Eventual |
| Scale | ⚠️ Vertical + sharding | ✅ Horizontal |
| Query flexibility | ✅ Full SQL | ⚠️ Key-value patterns |
| Operational cost | ⚠️ DBA needed | ✅ Managed service |
| **Decision** | **→ Option A** | |
| **Rationale** | Need complex queries and transactions for financial data | |

### Decision 2: [e.g., Sync vs Async processing]
...
```

### Phase 8: Design Document & Review

1. **Compile the design document:**
   Save to `system-documentation/system-design.md` combining all phases above

2. **Knowledge capture:**
   Invoke the `session-knowledge` skill to extract reusable knowledge — architectural patterns, capacity planning heuristics, infrastructure decisions, and integration patterns discovered during this design.

3. **Multi-model review** (same as engineering-task workflow):
   - **Architecture review** — `model: "gemini-3-pro-preview"`: Scalability and architecture fit
   - **Edge case review** — `model: "gpt-5.4"`: Failure modes and security
   - **Completeness review** — `model: "claude-sonnet-4.6"`: Missing requirements

3. **Stakeholder review:**
   - Share with relevant teams for feedback
   - Document feedback and decisions in the design doc

4. **Create follow-up tasks:**
   - Break the system design into implementable engineering tasks
   - Each task should reference the design document
   - Tag follow-up tasks with `engineering` for the engineering workflow

5. **Run mandatory completion review:**
   Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.

6. **Complete the task:**
   ```
   DailyPlanner-complete_task(taskId: "[task_id]", summary: "[system designed, N follow-up tasks created]")
   ```

## Quick Reference: Estimation Cheat Sheet

| Metric | Value |
|--------|-------|
| Seconds in a day | ~86,400 (~100K) |
| Seconds in a year | ~31.5M (~30M) |
| 1 million req/day | ~12 QPS |
| 1 billion req/day | ~12K QPS |
| 1 KB × 1M = 1 GB | |
| 1 KB × 1B = 1 TB | |
| 1 MB × 1M = 1 TB | |

| Power of 2 | Approximate Value |
|-----------|------------------|
| 2^10 | 1 Thousand (1 KB) |
| 2^20 | 1 Million (1 MB) |
| 2^30 | 1 Billion (1 GB) |
| 2^40 | 1 Trillion (1 TB) |

## Integration Points
- **Engineering Checklist:** Reference `engineering-checklist` skill for detailed quality gates (security, testing, deployment, monitoring)
- **Architecture Decision:** Document major decisions with `architecture-decision` skill
- **Engineering Task:** Create follow-up implementation tasks using `engineering-task` workflow
- **Tech Docs:** Generate final documentation using `tech-docs` skill
- **Impact Tracker:** If task is tagged "official", document impact on completion
- **Activity Log:** Log progress at each phase via `DailyPlanner-add_activity_log`

## Rules
1. ✅ Always start with requirements — don't jump to architecture
2. ✅ Back-of-envelope math before detailed design
3. ✅ Document trade-offs explicitly — show alternatives considered
4. ✅ Design for the expected scale, not infinite scale
5. ✅ Break the design into implementable engineering tasks
6. ⛔ Don't over-engineer — start simple, evolve as needed
7. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
8. ✅ Any code produced must be documented and have unit tests
