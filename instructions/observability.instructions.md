---
description: "Observability standards covering structured logging, distributed tracing, metrics collection, alerting, and dashboard design using OpenTelemetry and Azure Monitor."
applyTo: "**/logging/**,**/telemetry/**,**/monitoring/**,**/diagnostics/**"
---

# Observability Standards

You cannot improve what you cannot observe. Observability is not an afterthought — it ships with the feature.

---

## The Three Pillars

| Pillar   | Purpose                                    | Tool                          |
|----------|--------------------------------------------|-------------------------------|
| **Logs** | Discrete events with context               | Structured JSON → Azure Monitor / Geneva |
| **Traces** | Request flow across service boundaries   | OpenTelemetry → Application Insights |
| **Metrics** | Aggregated measurements over time        | OpenTelemetry / Geneva Metrics |

All three are required. Logs without traces are noise. Traces without metrics lack trends. Metrics without logs lack detail. Use correlation IDs to link all three.

---

## Structured Logging

### Format
- All logs MUST be structured JSON. No unstructured text logs in production.
- Use the logging framework's structured format — never string interpolation in templates.

### Correct (Semantic/Structured)
```csharp
logger.LogInformation("Order {OrderId} placed by {UserId} for {Amount}",
    order.Id, user.Id, order.Total);
```

### Wrong (String Interpolation)
```csharp
// NEVER DO THIS — destroys structured queryability
logger.LogInformation($"Order {order.Id} placed by {user.Id} for {order.Total}");
```

Why it matters: Structured parameters are indexed and queryable. String interpolation produces flat text that requires regex to parse. The structured version lets you query `WHERE OrderId = 'abc-123'` in Azure Monitor. The interpolated version requires `WHERE message CONTAINS 'abc-123'`.

### Required Fields on Every Log Entry
- `Timestamp` — UTC, ISO 8601 format.
- `Level` — Trace, Debug, Information, Warning, Error, Critical.
- `Message` — Human-readable description with structured placeholders.
- `CorrelationId` — The distributed trace ID linking this log to the request.
- `ServiceName` — Which service emitted this log.
- `Environment` — Production, Staging, Development.

---

## Log Levels — When to Use Each

| Level        | Use For                                                      | Examples |
|--------------|--------------------------------------------------------------|----------|
| **Trace**    | Ultra-verbose diagnostics. Off in production by default.     | Method entry/exit, variable values during debugging |
| **Debug**    | Detailed flow information useful during development.         | Cache hit/miss, query parameters, configuration loaded |
| **Information** | Normal operational events. The "happy path."              | Request processed, job completed, user logged in |
| **Warning**  | Something unexpected but recoverable. Investigate if recurring. | Retry attempt, deprecated API called, approaching rate limit |
| **Error**    | A failure that affects the current operation but not the service. | Failed to process order, external API returned 500, timeout |
| **Critical** | The service is dying. Wake someone up.                       | Unhandled exception, database unreachable, out of memory |

### Rules
- **Information** is the default production level. Not Debug, not Warning.
- **Error** means something broke for a specific request. Log it with full context (request details, stack trace, correlation ID).
- **Critical** means the service is in trouble. This triggers PagerDuty/IcM. Use it sparingly and correctly.
- **Warning** is the "yellow light." If you see it frequently, it should probably be Info (expected) or Error (actually broken).
- Never log at Error/Critical for expected conditions (e.g., 404 for a missing resource is not an error).

---

## Correlation IDs — The Connective Tissue

- Every inbound request MUST receive a correlation ID (trace ID).
- If the caller provides one (via `traceparent` header / W3C Trace Context), use it.
- If not, generate one at the API gateway / entry point.
- **Propagate the correlation ID to every downstream call** — HTTP headers, message queue properties, background job context.
- Include the correlation ID in every log entry, every trace span, and every metric exemplar.
- In responses, return the correlation ID via `X-Request-Id` or `traceparent` header for client-side debugging.

---

## OpenTelemetry Integration

### Traces and Spans
```csharp
using var activity = ActivitySource.StartActivity("ProcessOrder");
activity?.SetTag("order.id", order.Id);
activity?.SetTag("order.amount", order.Total);
activity?.SetTag("customer.tier", customer.Tier);

try
{
    await ProcessPayment(order);
    activity?.SetStatus(ActivityStatusCode.Ok);
}
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    activity?.RecordException(ex);
    throw;
}
```

### Span Naming Conventions
- `{ServiceName}.{Operation}` — e.g., `OrderService.ProcessOrder`, `PaymentGateway.Charge`.
- HTTP spans: `HTTP {METHOD} {route}` — e.g., `HTTP GET /v1/users/{id}`.
- Database spans: `DB {operation} {table}` — e.g., `DB SELECT Users`.
- Message spans: `{queue} send` / `{queue} receive` — e.g., `order-events send`.

### Baggage
- Use OpenTelemetry Baggage for context that should propagate across all services in a request chain (e.g., tenant ID, feature flags).
- Keep baggage small — it adds overhead to every inter-service call.

### SDK Setup
- Register OpenTelemetry in the DI container at startup.
- Export to Azure Monitor (Application Insights) for Microsoft services.
- Export to Geneva for internal Microsoft telemetry pipelines.
- Use auto-instrumentation for HTTP clients, database drivers, and message queues where available.

---

## Metrics

### Types
| Type          | Use For                                      | Example |
|---------------|----------------------------------------------|---------|
| **Counter**   | Monotonically increasing values              | `http_requests_total`, `orders_created_total` |
| **Histogram** | Distribution of values (latencies, sizes)    | `http_request_duration_seconds`, `payload_size_bytes` |
| **Gauge**     | Point-in-time values that go up and down     | `active_connections`, `queue_depth`, `cpu_usage_percent` |

### Naming Conventions
- Use **snake_case** for metric names.
- Include the **unit** as a suffix: `_seconds`, `_bytes`, `_total`.
- Prefix with the **service/subsystem**: `order_service_http_request_duration_seconds`.
- Follow OpenTelemetry/Prometheus naming conventions.

### Required Metrics for Every Service
```
# RED Metrics (Request-oriented)
http_requests_total{method, status_code, endpoint}
http_request_duration_seconds{method, endpoint}
http_request_errors_total{method, status_code, endpoint}

# USE Metrics (Resource-oriented)
process_cpu_usage_percent
process_memory_bytes
db_connection_pool_active
db_connection_pool_idle
message_queue_depth{queue_name}
```

### Metric Cardinality
- **Never use unbounded values as metric labels.** User IDs, request IDs, or timestamps as labels will blow up your metrics storage.
- Safe labels: HTTP method, status code class (2xx/4xx/5xx), endpoint route template, environment.
- Unsafe labels: user ID, email, request path with variable segments, timestamp.

---

## Health Checks

### Liveness Check (`/health/live`)
- "Is the process running and not deadlocked?"
- Should return 200 if the process is alive. Minimal checks — no dependency verification.
- Kubernetes uses this to decide whether to restart the container.
- If this fails, the container is killed and restarted.

### Readiness Check (`/health/ready`)
- "Can this instance handle traffic?"
- Checks critical dependencies: database connectivity, cache connectivity, required config loaded.
- Kubernetes uses this to decide whether to route traffic to this instance.
- If this fails, the instance is removed from the load balancer but NOT restarted.

### Startup Check (`/health/startup`)
- "Has the service finished initializing?"
- For services with slow startup (pre-warming caches, running migrations).
- Prevents liveness checks from killing the container during startup.

### Implementation Rules
- Health check endpoints are **unauthenticated** — Kubernetes probes don't carry auth tokens.
- Health checks should be **fast** (<1s). Use cached dependency status, not live checks.
- Return structured responses:
  ```json
  {
    "status": "Healthy",
    "checks": {
      "database": { "status": "Healthy", "duration": "45ms" },
      "cache": { "status": "Healthy", "duration": "12ms" },
      "messageQueue": { "status": "Degraded", "duration": "250ms", "description": "High latency" }
    }
  }
  ```

---

## Alerting Philosophy

### Core Principles
1. **Alert on symptoms, not causes.** Alert on "error rate > 5%" not "database CPU > 80%." Users feel symptoms, not causes.
2. **Every alert MUST be actionable.** If the on-call engineer can't do anything about it at 3 AM, it's not an alert — it's noise.
3. **Reduce alert fatigue relentlessly.** A team that ignores alerts because most are false positives will miss real incidents.
4. **Use severity levels correctly:**
   - **Sev 1 (Critical)**: Service is down or data loss is occurring. Pages immediately.
   - **Sev 2 (High)**: Significant degradation. Pages during business hours.
   - **Sev 3 (Medium)**: Minor issue. Ticket created, addressed within SLA.
   - **Sev 4 (Low)**: Informational. Dashboard notification only.

### Alert Design
- Include **runbook links** in every alert. The alert message should tell the on-call exactly what to check first.
- Set **appropriate thresholds** — alert on sustained anomalies, not transient spikes. Use sliding windows (5-minute averages, not per-second).
- **Test alerts.** Inject failures in staging and verify alerts fire correctly with the right severity and routing.
- **Review alerts monthly.** Delete alerts that haven't fired in 90 days. Tune thresholds on alerts that fire too often.

---

## SLI / SLO / SLA Definitions

| Term    | Definition                                    | Example |
|---------|-----------------------------------------------|---------|
| **SLI** | Service Level Indicator — the metric          | 99th percentile latency, error rate, availability |
| **SLO** | Service Level Objective — the target          | p99 latency < 200ms, error rate < 0.1%, availability 99.95% |
| **SLA** | Service Level Agreement — the contract + consequences | 99.9% uptime or credits issued |

### Rules
- Define SLIs for every service. No SLI = no way to know if the service is healthy.
- Set SLOs that are ambitious but achievable. 100% is not an SLO — it's a fantasy.
- Use **error budgets**: if your SLO is 99.9%, you have 43.8 minutes of downtime per month. Spend it wisely on deployments and experiments.
- Track SLO compliance on dashboards. Burn rate alerts catch SLO violations before they breach.

---

## Dashboard Design

### RED Method (Request-Driven Services — APIs, Web Apps)
- **Rate**: Requests per second.
- **Errors**: Error rate as a percentage of total requests.
- **Duration**: Latency distribution (p50, p95, p99).

### USE Method (Resource-Driven Infrastructure — Databases, Queues, Caches)
- **Utilization**: How busy is the resource? (CPU %, memory %, disk I/O %).
- **Saturation**: How much work is queued? (Queue depth, thread pool saturation).
- **Errors**: Error count/rate from the resource.

### Dashboard Layout
1. **Top row**: Golden signals — availability, error rate, latency (p50/p99).
2. **Second row**: Traffic — requests/sec, concurrent users, throughput.
3. **Third row**: Dependencies — database latency, cache hit rate, queue depth.
4. **Bottom row**: Infrastructure — CPU, memory, pod count, node health.

### Rules
- Every service MUST have a dashboard. No dashboard = no visibility = no ownership.
- Dashboards show the last 24 hours by default, with easy drill-down to 1h/7d/30d.
- Use consistent color coding: green = healthy, yellow = degraded, red = critical.
- Include SLO burn rate visualization on every service dashboard.

---

## Azure Monitor / Application Insights

- Use Application Insights as the primary APM for Azure-hosted services.
- Enable **distributed tracing** across all services in the call chain.
- Configure **sampling** appropriately: 100% sampling in staging, adaptive sampling in production (target ~5 req/sec per instance for cost management).
- Use **custom dimensions** on telemetry for business-context queries:
  ```csharp
  telemetryClient.TrackEvent("OrderPlaced", new Dictionary<string, string>
  {
      ["OrderId"] = order.Id,
      ["CustomerTier"] = customer.Tier,
      ["PaymentMethod"] = order.PaymentMethod
  });
  ```
- Set up **availability tests** (URL ping tests) for all public-facing endpoints.
- Configure **smart detection alerts** for anomaly detection.
- Use **workbooks** for custom dashboards and **alerts** for automated monitoring.

---

## Geneva Metrics (Microsoft Internal)

- Use Geneva for internal Microsoft services that feed into the standard MSFT monitoring pipeline.
- Emit pre-aggregated metrics to Geneva for efficiency.
- Follow the Geneva metric naming taxonomy for your service tree node.
- Ensure Geneva monitors are configured for your IcM routing rules.
- Use MDM (Metrics Data Manager) dimensions judiciously — same cardinality rules as Prometheus labels.

---

## Log Retention Policies

| Environment  | Retention     | Rationale |
|-------------|---------------|-----------|
| Production  | 90 days hot, 1 year cold storage | Incident investigation, compliance |
| Staging     | 30 days       | Debugging pre-production issues |
| Development | 7 days        | Active development only |

- Archive production logs to cold storage (Azure Blob) for compliance requirements beyond 90 days.
- Set up automated purge jobs for expired logs.
- Cost-optimize by adjusting sampling and retention by log level: keep all Error/Critical for 1 year, Information for 90 days, Debug for 30 days.

---

## PII in Logs — Never

### Absolute Rules
- **Never log PII.** No emails, names, phone numbers, addresses, SSNs, credit card numbers, IP addresses, or session tokens in logs.
- **Never log secrets.** No API keys, passwords, connection strings, or tokens.
- **Never log request/response bodies** that may contain user data without explicit redaction.

### How to Handle
- **Mask sensitive data**: Log `user: ***@example.com` or `card: ****-****-****-1234`.
- **Use opaque identifiers**: Log user IDs, not usernames. Log order IDs, not order contents.
- **Implement a log scrubber** in the logging pipeline that detects and redacts PII patterns before storage.
- **Audit your logs** quarterly for PII leaks. Automated scanning tools exist — use them.
- Configure Application Insights to **strip query parameters** from URLs that might contain tokens.

---

## Observability Review Checklist

Before any service ships:

- [ ] Structured logging is configured (JSON, semantic parameters, no string interpolation).
- [ ] Log levels are used correctly (Info for happy path, Error for failures, Critical for service-level issues).
- [ ] Correlation IDs are generated and propagated across all service boundaries.
- [ ] OpenTelemetry is configured with traces, spans, and appropriate exporters.
- [ ] RED metrics are emitted (request rate, error rate, duration).
- [ ] Health checks are implemented (liveness, readiness, startup).
- [ ] SLIs and SLOs are defined and tracked on a dashboard.
- [ ] Alerts are configured, actionable, and include runbook links.
- [ ] Dashboard exists with golden signals, dependencies, and infrastructure views.
- [ ] No PII in logs — verified by automated scanning.
- [ ] Log retention policies are configured per environment.
- [ ] Sampling is configured appropriately for cost management.
