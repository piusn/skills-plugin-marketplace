---
description: "Engineering quality checklist covering security, testing, deployment, rollback, and observability. Use this skill when the user says 'quality check', 'pre-ship checklist', 'ready to deploy', 'review checklist', 'security review', or 'deployment checklist'. A reusable quality gate referenced by engineering and system design workflows before shipping."
---

# Engineering Quality Checklist

## Context
This is a shared quality gate — a comprehensive checklist that ensures code, infrastructure, and operational readiness before shipping. It's referenced by the `engineering-task`, `workflow-quickfix`, and `workflow-system-design` workflows, but can also be invoked independently for any pre-ship review.

## When to Use
- Before merging a feature branch
- Before deploying to production
- During code review to check for gaps
- When asked to do a quality/readiness review
- As a pre-launch checklist for new services

## How to Use
Run through each section relevant to the change. Not every item applies to every change — use judgment. For quick fixes, focus on Testing and Security. For new services, cover all sections.

---

## 1. Security Checklist

### Authentication & Authorization
- [ ] All endpoints require appropriate authentication
- [ ] Authorization checks enforce least-privilege access
- [ ] API keys, tokens, and secrets are not hardcoded — use environment variables or secret stores
- [ ] Service-to-service communication uses mTLS or managed identity where applicable

### Input Validation
- [ ] All user inputs are validated and sanitized
- [ ] SQL queries use parameterized statements (no string concatenation)
- [ ] File uploads are validated (type, size, content)
- [ ] JSON/XML payloads are schema-validated

### Data Protection
- [ ] Sensitive data is encrypted at rest (database, storage)
- [ ] Data in transit uses TLS 1.2+
- [ ] PII is not logged or is masked in logs
- [ ] Data retention policies are implemented

### Common Vulnerabilities
- [ ] No XSS vulnerabilities (output encoding applied)
- [ ] CSRF protection on state-changing operations
- [ ] Rate limiting on public-facing endpoints
- [ ] Error messages don't leak internal details (stack traces, paths, versions)
- [ ] Dependencies scanned for known vulnerabilities (Dependabot, Snyk, etc.)

### Compliance
- [ ] Relevant compliance requirements met (GDPR, SOC2, etc.)
- [ ] Audit logging captures security-relevant events
- [ ] Access reviews documented for privileged operations

---

## 2. Testing Checklist

### Unit Testing
- [ ] All new functions/methods have unit tests
- [ ] Edge cases are tested (null, empty, boundary values, overflow)
- [ ] Error paths are tested (exceptions, timeouts, invalid input)
- [ ] Mocks/stubs used appropriately — not mocking the thing under test
- [ ] Test names clearly describe the scenario being tested
- [ ] Code coverage meets project standard (aim for 80%+ on new code)

### Integration Testing
- [ ] API endpoints tested end-to-end with real (or realistic) dependencies
- [ ] Database interactions tested (CRUD, transactions, constraints)
- [ ] External service integrations tested with contract tests or stubs
- [ ] Authentication/authorization flows tested
- [ ] Error responses verified (status codes, error bodies)

### End-to-End (E2E) Testing
- [ ] Critical user journeys have automated E2E tests
- [ ] UI interactions validated (if applicable)
- [ ] Cross-browser/cross-platform tested (if applicable)
- [ ] Performance under realistic data volumes verified

### UI Testing (via `ui-testing-agent` MCP)
- [ ] UI test plan defined in design document (mandatory for UI changes)
- [ ] Local backend and frontend running before testing
- [ ] All UI test scenarios executed via `ui-testing-agent`
- [ ] Authentication tested (login flow, token management)
- [ ] Form validations verified (required fields, error messages)
- [ ] Navigation flows verified (routing, breadcrumbs, back buttons)
- [ ] Responsive design checked (mobile, tablet, desktop viewpoints)
- [ ] Edge cases tested (empty states, error states, loading states)
- [ ] Screenshots captured for failures
- [ ] All UI tests passing before merge

### Test Quality
- [ ] Tests are deterministic — no flaky tests introduced
- [ ] Tests are independent — no order-dependent execution
- [ ] No existing tests were modified to pass (without explicit approval)
- [ ] Test data is isolated and cleaned up after test runs

---

## 3. Deployment & Rollout Checklist

### Pre-Deployment
- [ ] All tests pass in CI/CD pipeline
- [ ] Code reviewed and approved
- [ ] Database migrations tested (forward AND rollback)
- [ ] Configuration changes documented and applied
- [ ] Feature flags in place for risky changes
- [ ] Deployment runbook updated (if applicable)

### Rollout Strategy
Choose the appropriate strategy based on risk:

| Strategy | Risk Level | Use When |
|----------|-----------|----------|
| **Direct deploy** | Low | Config changes, minor fixes |
| **Rolling update** | Medium | Standard deployments, stateless services |
| **Blue-green** | Medium-High | Zero-downtime required, easy rollback |
| **Canary** | High | Large changes, new features, critical services |
| **Feature flag** | Variable | Gradual rollout, A/B testing, kill switch needed |

- [ ] Rollout strategy selected and documented
- [ ] Rollout percentage/stages defined (for canary/feature flag)
- [ ] Success criteria defined (metrics to watch during rollout)
- [ ] Rollout timeline communicated to stakeholders

### Rollback Strategy
- [ ] Rollback procedure documented and tested
- [ ] Database rollback scripts prepared (if schema changes)
- [ ] Rollback can be executed in < 15 minutes
- [ ] Rollback decision criteria defined (error rate > X%, latency > Y ms)
- [ ] On-call team aware of the deployment and rollback procedure

### Post-Deployment
- [ ] Smoke tests pass in production
- [ ] Key metrics verified (latency, error rate, throughput)
- [ ] No unexpected log patterns (errors, warnings)
- [ ] Deployment logged in activity tracker

---

## 4. Monitoring & Observability Checklist

### Metrics (RED Method)
- [ ] **Rate** — request throughput tracked per endpoint
- [ ] **Errors** — error rate tracked (4xx, 5xx separately)
- [ ] **Duration** — latency tracked (p50, p95, p99)
- [ ] Custom business metrics added where relevant (orders/min, sign-ups, etc.)

### Logging
- [ ] Structured logging used (JSON format with consistent fields)
- [ ] Correlation IDs propagated across service boundaries
- [ ] Log levels used appropriately (ERROR for failures, WARN for degradation, INFO for operations, DEBUG for troubleshooting)
- [ ] No sensitive data in logs (passwords, tokens, PII)
- [ ] Log retention and rotation configured

### Distributed Tracing
- [ ] Traces instrumented for cross-service calls
- [ ] Span names are descriptive and consistent
- [ ] Key attributes attached to spans (userId, orderId, etc.)
- [ ] Slow transaction traces captured (> threshold)

### Alerting
- [ ] SLO-based alerts configured (error budget burn rate)
- [ ] Alerts have clear runbook links
- [ ] Alert thresholds tuned to avoid alert fatigue
- [ ] Escalation path defined (PagerDuty, on-call rotation)
- [ ] Dashboard created/updated with key metrics

### Health Checks
- [ ] Liveness probe configured (is the process alive?)
- [ ] Readiness probe configured (can it handle traffic?)
- [ ] Dependency health checks included (DB, cache, downstream services)
- [ ] Health endpoint returns meaningful status (not just 200 OK)

---

## 5. Operational Readiness

### Documentation
- [ ] README updated with setup, build, and run instructions
- [ ] API documentation current (OpenAPI/Swagger if applicable)
- [ ] Architecture decision records (ADRs) created for significant choices
- [ ] Runbook/TSG exists for on-call troubleshooting

### Capacity
- [ ] Resource limits configured (CPU, memory, connections)
- [ ] Auto-scaling rules defined (if applicable)
- [ ] Load testing performed for capacity-critical changes
- [ ] Database connection pooling configured

### Disaster Recovery
- [ ] Backup strategy in place for critical data
- [ ] Recovery procedure documented and tested
- [ ] RTO (Recovery Time Objective) and RPO (Recovery Point Objective) defined
- [ ] Multi-region/multi-AZ failover configured (if required)

---

## Quick Reference: Which Sections Apply?

| Change Type | Security | Testing | Deployment | Monitoring | Ops Readiness |
|------------|----------|---------|------------|------------|---------------|
| Quick fix / bug patch | ✅ Input validation | ✅ Unit + regression | ✅ Direct deploy | ⚠️ If relevant | ❌ |
| Feature addition | ✅ Full | ✅ Full | ✅ Rolling/Canary | ✅ Full | ⚠️ If new service |
| New service / system | ✅ Full | ✅ Full | ✅ Blue-green/Canary | ✅ Full | ✅ Full |
| Config change | ⚠️ Secrets only | ⚠️ Smoke test | ✅ Direct deploy | ⚠️ If relevant | ❌ |
| Database migration | ✅ Data protection | ✅ Migration + rollback | ✅ Blue-green | ✅ Query metrics | ⚠️ Backup |

## Integration Points
- Referenced by `engineering-task` workflow before Phase 6 (Ship)
- Referenced by `workflow-quickfix` before Phase 4 (Ship)
- Referenced by `workflow-system-design` during Phase 6 (Deep Dives)
- Can be invoked standalone: "run the pre-ship checklist"
