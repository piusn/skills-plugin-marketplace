---
description: "Performance engineering standards covering profiling, caching strategies, load testing, performance budgets, and optimization patterns across all stacks."
---

# Performance Engineering Standards

## Core Philosophy

Performance is a feature, not an afterthought. Define performance budgets before writing code. Measure before optimizing — intuition about bottlenecks is wrong more often than it's right. Every performance claim must be backed by profiling data.

## Performance Budgets

Define these budgets at project inception and enforce them in CI:

- **Page load (LCP):** < 3 seconds on 3G, < 1.5 seconds on broadband
- **API response time:** < 200ms P95, < 500ms P99 for standard CRUD; < 1s P95 for complex aggregations
- **Time to Interactive (TTI):** < 5 seconds on mobile
- **Bundle size:** < 200KB gzipped for initial JS bundle, < 50KB for critical CSS
- **First Contentful Paint:** < 1.8 seconds
- **Cumulative Layout Shift:** < 0.1
- **Memory usage:** Define per-service ceiling based on container limits (request 70% of limit)
- **Database query time:** < 50ms P95 for indexed queries, < 200ms for reports

Budgets are not aspirational — they are hard limits. Fail the build if a budget is exceeded.

## Profiling Before Optimizing

**Never optimize without profiling data.** The process is always:

1. **Measure** — Profile the current state with production-like data
2. **Identify** — Find the actual bottleneck (it's rarely where you think)
3. **Hypothesize** — Form a specific theory about why it's slow
4. **Fix** — Make a targeted change
5. **Verify** — Profile again to confirm improvement and check for regressions

### Profiling Tools by Stack

- **C# / .NET:** dotnet-trace, dotnet-counters, BenchmarkDotNet for micro-benchmarks, Application Insights Profiler
- **JavaScript / Node.js:** Chrome DevTools Performance tab, Node.js --prof, clinic.js, Lighthouse
- **Python:** cProfile, py-spy, memory_profiler
- **Database:** EXPLAIN ANALYZE (Postgres), Query Store (SQL Server), slow query logs
- **Distributed systems:** Jaeger/Zipkin traces, Application Insights end-to-end transaction diagnostics

### Anti-Pattern: Premature Optimization

Do NOT:
- Optimize code that runs once during startup
- Micro-optimize code that is not in a hot path
- Add complexity for theoretical performance gains without measurements
- Cache data that is cheap to compute and rarely accessed

## Caching Strategy

### Cache Hierarchy (check in order)

```
Browser Cache → CDN / Edge → Reverse Proxy → Application Cache → Database Cache → Origin
```

Every layer should be intentional. Do not add caching layers "just in case."

### Cache Invalidation Patterns

- **TTL-based:** Set explicit TTL for all cached data. Short TTL (1-5 min) for frequently changing data, longer (1-24 hours) for reference data
- **Event-driven invalidation:** Publish cache-bust events on data mutation. Prefer this over TTL for consistency-sensitive data
- **Cache-aside (lazy loading):** Application checks cache → on miss, loads from source → populates cache. Default pattern for most use cases
- **Write-through:** Write to cache and source simultaneously. Use when read-after-write consistency matters
- **Write-behind:** Write to cache, async write to source. Use only when you can tolerate potential data loss

### Cache Key Design

- Include all parameters that affect the response
- Include version/schema identifiers for format changes
- Use consistent hashing for distributed caches
- Prefix keys with service name to avoid collisions: `{service}:{entity}:{id}:{version}`

### What NOT to Cache

- User-specific sensitive data in shared caches
- Data that changes on every request
- Large objects that blow out cache memory
- Data where staleness causes correctness issues (financial transactions, inventory counts)

## Database Performance

### Indexing

- **Index every column used in WHERE, JOIN, and ORDER BY clauses**
- Use composite indexes matching your most common query patterns (leftmost prefix rule)
- Monitor unused indexes — they slow down writes for zero read benefit
- Use covering indexes for high-frequency queries to avoid table lookups
- Review query plans in CI for any full table scans on tables > 10K rows

### Query Patterns

- **Eliminate N+1 queries:** Use JOINs, batch loading, or DataLoader pattern. An ORM that issues N+1 queries is a bug, not a feature
- **Use read replicas** for reporting, analytics, and read-heavy workloads
- **Connection pooling is mandatory:** Never create a new connection per request. Use HikariCP (Java), Npgsql pooling (C#), pgBouncer (Postgres)
- **Parameterized queries always:** For security AND for query plan caching
- **Pagination is required:** Never return unbounded result sets. Use keyset pagination (WHERE id > @lastId) over OFFSET for large datasets

### ORM Guidelines

- Use projections — SELECT only the columns you need, not SELECT *
- Disable lazy loading by default; explicitly eager-load required relationships
- Log generated SQL in development to catch inefficient patterns
- Set query timeouts: 30s for user-facing, 5m for background jobs, never unlimited

## Async Over Sync

- **All I/O operations must be async:** Database calls, HTTP requests, file operations, message queue operations
- **Never block on async:** No `.Result`, `.Wait()`, or `Task.Run(() => asyncMethod().Result)` in C#. No `sync` wrappers around async in Python
- Use `async/await` end-to-end — one synchronous bottleneck negates the entire async chain
- **Fire-and-forget is a code smell:** If you don't await it, you can't handle errors. Use background job queues instead
- Use `IAsyncEnumerable` / async iterators for streaming large datasets

## Memory Management

### C# / .NET

- Use `using` statements / `IAsyncDisposable` for all disposable resources
- Pool expensive objects with `ObjectPool<T>`
- Use `Span<T>` and `Memory<T>` to avoid allocations in hot paths
- Be aware of closure allocations in LINQ and lambdas on hot paths
- Use `ArrayPool<T>.Shared` for temporary buffers
- Profile with dotnet-counters for GC pressure indicators

### JavaScript / TypeScript

- Remove event listeners when components unmount
- Clear intervals and timeouts
- Avoid closures that capture large scopes unnecessarily
- Use WeakMap/WeakRef for caches that should not prevent GC
- Watch for detached DOM nodes in SPAs
- Set `null` on references to large objects when done

### General

- Stream large files — never load entire files into memory
- Use generators/iterators for large collections
- Set memory limits on containers and test with those limits

## Lazy Loading

- **UI components:** Lazy-load below-the-fold components, route-level code splitting, dynamic imports for heavy libraries
- **Data:** Load data on demand, not on page load. Prefetch on hover/focus for perceived performance
- **Images:** Use native `loading="lazy"`, serve responsive images with `srcset`, use modern formats (WebP/AVIF)
- **Relationships:** Don't eager-load entity graphs. Load the minimum needed for the current view

## Compression and Transfer

- **Enable gzip/brotli** for all HTTP responses > 1KB (brotli preferred — 15-20% better compression)
- **Use CDN** for all static assets with long cache headers (1 year) and content-hash filenames
- **HTTP/2 or HTTP/3** for multiplexing — eliminates need for domain sharding and sprite sheets
- **Minimize payload size:** No unnecessary fields in API responses. Support `fields` parameter for sparse fieldsets
- Use protocol buffers or MessagePack for internal service-to-service communication when JSON overhead matters

## Batch Operations

- **Batch over individual calls:** One batch insert of 1000 rows beats 1000 individual inserts by 10-100x
- **Batch API calls:** Use bulk endpoints. If consuming an API without bulk support, parallelize with controlled concurrency
- **Database:** Use bulk insert, MERGE/upsert operations, table-valued parameters (SQL Server)
- **Message queues:** Send/receive in batches, not one-at-a-time

## Load Testing

### Requirements

- **Define baseline:** Establish current performance metrics before any optimization work
- **Test at 2x expected load** as standard, 5x for critical paths
- **Test with production-like data volumes** — performance on 100 rows tells you nothing about 10M rows
- **Include soak tests:** Run at sustained load for 4+ hours to detect memory leaks and connection exhaustion
- **Test failure scenarios:** What happens at 10x load? Graceful degradation > cascading failure

### Process

1. Define scenarios from production traffic patterns
2. Script realistic user journeys, not just individual endpoints
3. Run from multiple regions if globally distributed
4. Monitor application metrics, not just response times (CPU, memory, GC, connection pools, queue depth)
5. Compare results against performance budgets
6. Automate and run on every release candidate

### Tools

- k6 (preferred — scriptable, CI-friendly, cloud option)
- Azure Load Testing for cloud-native scenarios
- JMeter for complex scenarios requiring GUI design

## Performance in CI/CD

- **Bundle size checks:** Fail PR if bundle exceeds budget
- **Lighthouse CI:** Run on every PR for web applications, fail on regression
- **Query plan analysis:** Detect full table scans on new queries
- **Benchmark regression detection:** Run BenchmarkDotNet / k6 in CI, compare against baseline, alert on > 10% regression
- **Container image size:** Track and alert on growth > 20%

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Do This Instead |
|---|---|---|
| Premature optimization | Wastes time, adds complexity | Profile first, optimize bottlenecks |
| Over-caching | Stale data bugs, memory pressure | Cache what's expensive AND frequently read |
| Sync I/O in hot paths | Thread starvation under load | Async all I/O |
| Unbounded queries | Memory exhaustion, timeout | Always paginate |
| SELECT * | Wasted bandwidth, no covering indexes | Select only needed columns |
| N+1 queries | Database round-trip explosion | Batch load, JOINs |
| No connection pooling | Connection exhaustion | Pool is mandatory |
| Logging in hot loops | I/O bottleneck, disk pressure | Sample or aggregate |
| String concatenation in loops | GC pressure from allocations | StringBuilder / template literals |
| Ignoring cold start | Bad UX on first request | Warm up, lazy load strategically |
