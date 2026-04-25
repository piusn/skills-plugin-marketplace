---
description: "REST and GraphQL API design standards covering versioning, pagination, error responses, naming conventions, OpenAPI specs, and contract-first development."
applyTo: "**/controllers/**,**/api/**,**/endpoints/**,**/*Controller*.cs,**/*Router*.ts,**/swagger/**,**/*.openapi.*"
---

# API Design Standards

These standards apply to all REST and GraphQL APIs across teams. APIs are contracts — treat them with the same rigor as database schemas.

---

## Contract-First Development

- **Always start with the OpenAPI spec** before writing any implementation code.
- The spec IS the source of truth. Generate server stubs and client SDKs from it.
- Every endpoint MUST be documented in OpenAPI 3.0+ with descriptions, examples, and schema definitions.
- Store specs in `docs/api/` or alongside the controller in a `*.openapi.yaml` file.
- Use `$ref` for shared schemas — never duplicate type definitions across endpoints.
- Run spec linting (Spectral or equivalent) in CI. Broken specs block merge.

---

## Resource Naming

- Use **plural nouns** for resource collections: `/users`, `/orders`, `/deployments`.
- Use **kebab-case** for multi-word resources: `/build-queues`, `/service-connections`.
- Resources represent nouns, never verbs: `/users/{id}/activate` → `POST /users/{id}/activation`.
- Nest resources to express ownership, but limit to **2 levels max**: `/teams/{teamId}/members/{memberId}` is fine; deeper nesting should be flattened with query filters.
- Use consistent casing: **camelCase** for JSON property names in request/response bodies.

---

## HTTP Methods — Use Them Correctly

| Method  | Semantics                        | Idempotent | Safe |
|---------|----------------------------------|------------|------|
| GET     | Read a resource or collection    | Yes        | Yes  |
| POST    | Create a resource or trigger action | No      | No   |
| PUT     | Full replacement of a resource   | Yes        | No   |
| PATCH   | Partial update of a resource     | No*        | No   |
| DELETE  | Remove a resource                | Yes        | No   |

- **GET** must never mutate state. Ever.
- **POST** is for creation and for RPC-style actions when REST semantics don't fit cleanly.
- **PUT** replaces the entire resource — omitted fields are set to defaults/null.
- **PATCH** uses JSON Merge Patch (RFC 7396) or JSON Patch (RFC 6902). Prefer Merge Patch for simplicity.
- **DELETE** should be idempotent: deleting an already-deleted resource returns 204, not 404.

---

## Status Codes — Be Precise

### Success Codes
| Code | When to Use |
|------|-------------|
| **200 OK** | Successful GET, PUT, PATCH that returns a body |
| **201 Created** | Successful POST that creates a resource. Include `Location` header with new resource URI |
| **202 Accepted** | Request accepted for async processing. Return a status monitor URI |
| **204 No Content** | Successful DELETE or PUT/PATCH that returns no body |

### Client Error Codes
| Code | When to Use |
|------|-------------|
| **400 Bad Request** | Malformed syntax, invalid JSON, missing required fields |
| **401 Unauthorized** | Missing or invalid authentication credentials |
| **403 Forbidden** | Authenticated but lacking permission for this resource/action |
| **404 Not Found** | Resource does not exist (also use to hide existence from unauthorized users) |
| **409 Conflict** | State conflict — e.g., duplicate creation, optimistic concurrency violation |
| **422 Unprocessable Entity** | Syntactically valid but semantically invalid — business rule violations |
| **429 Too Many Requests** | Rate limit exceeded. Include `Retry-After` header |

### Server Error Codes
| Code | When to Use |
|------|-------------|
| **500 Internal Server Error** | Unhandled exception. Log it, alert on it, never expose internals |
| **502 Bad Gateway** | Upstream dependency failure |
| **503 Service Unavailable** | Planned maintenance or overload. Include `Retry-After` |
| **504 Gateway Timeout** | Upstream dependency timeout |

- **Never return 200 with an error in the body.** If something failed, use an error status code.
- **Never return 500 for client mistakes.** Validate input and return 4xx.

---

## Error Response Format (RFC 9457 — Problem Details)

All error responses MUST follow RFC 9457:

```json
{
  "type": "https://api.example.com/problems/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 422,
  "detail": "Account balance is $30.00 but the transfer requires $50.00.",
  "instance": "/transfers/txn-12345",
  "traceId": "00-abcdef1234567890-abcdef12-01",
  "errors": {
    "amount": ["Transfer amount exceeds available balance"]
  }
}
```

- `type` — URI identifying the error type (can be a documentation link).
- `title` — Short human-readable summary (same for all instances of this type).
- `status` — HTTP status code (duplicated for convenience).
- `detail` — Human-readable explanation specific to this occurrence.
- `instance` — URI identifying the specific occurrence.
- `traceId` — Correlation ID for support and debugging.
- `errors` — Optional validation errors keyed by field name.

---

## Pagination — Cursor-Based Preferred

### Cursor-Based (Preferred)
```
GET /users?limit=25&after=eyJpZCI6MTAwfQ==
```
Response:
```json
{
  "data": [...],
  "pagination": {
    "hasNextPage": true,
    "endCursor": "eyJpZCI6MTI1fQ==",
    "hasPreviousPage": true,
    "startCursor": "eyJpZCI6MTAxfQ=="
  }
}
```

- Use opaque cursors (base64-encoded) — never expose raw IDs or offsets.
- Cursor-based pagination is stable under concurrent writes. Offset-based is not.

### Offset-Based (Only When Necessary)
```
GET /reports?page=3&pageSize=25
```
- Acceptable for admin dashboards, static datasets, or when "jump to page N" is required.
- Always return `totalCount`, `page`, `pageSize`, and `totalPages` in response metadata.
- Cap `pageSize` at a server-defined maximum (e.g., 100). Reject requests exceeding it.

---

## Filtering, Sorting, and Field Selection

### Filtering
```
GET /users?status=active&role=admin&createdAfter=2024-01-01
```
- Use flat query parameters for simple filters.
- For complex filters, accept a `filter` parameter with a defined query language (OData subset or custom).
- Always validate and whitelist filterable fields — never pass raw input to queries.

### Sorting
```
GET /users?sort=createdAt:desc,name:asc
```
- Format: `field:direction` with comma separation for multi-sort.
- Default sort must be deterministic (include a tiebreaker like `id`).
- Document which fields are sortable.

### Field Selection (Sparse Fieldsets)
```
GET /users?fields=id,name,email
```
- Optional optimization — reduce payload size for bandwidth-sensitive clients.
- Default to returning all fields if `fields` is omitted.

---

## Versioning

- Use **URL path versioning**: `/v1/users`, `/v2/users`.
- The version represents the major contract version. Breaking changes = new version.
- Maintain at most **2 concurrent versions**. Deprecate older versions with a sunset timeline.
- Non-breaking changes (new optional fields, new endpoints) do NOT require a version bump.
- Include `api-version` in response headers for clarity.
- Sunset header: `Sunset: Sat, 01 Mar 2025 00:00:00 GMT` on deprecated versions.

---

## Idempotency

- All **PUT** and **DELETE** operations are idempotent by nature.
- For **POST** operations that create resources, support an `Idempotency-Key` header:
  ```
  POST /payments
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  ```
- Store the idempotency key → response mapping for at least 24 hours.
- If a duplicate key is received, return the original response (same status code and body).
- Return `422` if the same key is reused with a different request body.

---

## Rate Limiting

Include these headers on every response:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1620000000
Retry-After: 30
```

- `Retry-After` is REQUIRED on 429 responses (seconds or HTTP-date).
- Rate limits should be per-client (API key or token), not per-IP.
- Document rate limits in the OpenAPI spec and developer portal.

---

## Content Negotiation

- Default to `application/json` for all endpoints.
- Support `Accept` header for content negotiation when multiple formats are available.
- Return `406 Not Acceptable` if the requested format is not supported.
- Use `Content-Type` header on all responses.
- For file downloads, use appropriate MIME types and `Content-Disposition` headers.

---

## HATEOAS — When Appropriate

Include hypermedia links for discoverable APIs:
```json
{
  "id": "user-123",
  "name": "Jane Doe",
  "_links": {
    "self": { "href": "/v1/users/user-123" },
    "orders": { "href": "/v1/users/user-123/orders" },
    "deactivate": { "href": "/v1/users/user-123/activation", "method": "DELETE" }
  }
}
```

- Use HATEOAS for public APIs and APIs consumed by multiple teams.
- Skip it for internal microservice-to-microservice calls where clients are tightly coupled.

---

## Long-Running Operations

For operations that take more than a few seconds:

1. **Accept the request**: Return `202 Accepted` with a status monitor URI.
   ```json
   {
     "operationId": "op-abc-123",
     "status": "Running",
     "statusMonitor": "/v1/operations/op-abc-123"
   }
   ```
2. **Poll for status**: Client polls the status monitor URI.
   ```json
   {
     "operationId": "op-abc-123",
     "status": "Succeeded",
     "result": { "href": "/v1/deployments/deploy-456" },
     "percentComplete": 100
   }
   ```
3. **Status values**: `NotStarted`, `Running`, `Succeeded`, `Failed`, `Cancelled`.
4. Include `Retry-After` header on the 202 response to guide polling interval.

---

## Bulk Operations

- Use `POST /users/bulk` for batch creates with an array body.
- Return `200` with per-item results (not `201`) since individual items may fail:
  ```json
  {
    "results": [
      { "index": 0, "status": 201, "id": "user-1" },
      { "index": 1, "status": 409, "error": { "title": "Duplicate email" } }
    ],
    "succeeded": 1,
    "failed": 1
  }
  ```
- Cap batch size (e.g., 100 items max per request).
- For very large batches, use the long-running operation pattern.

---

## Request/Response Envelope

### Collection Response
```json
{
  "data": [...],
  "pagination": { ... },
  "meta": {
    "requestId": "req-abc-123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Single Resource Response
Return the resource directly (no envelope) for single-item GETs:
```json
{
  "id": "user-123",
  "name": "Jane Doe",
  "email": "jane@example.com"
}
```

- Envelopes on collections. Direct objects on single resources.
- Always include `requestId` for traceability in collection responses.

---

## API Review Checklist

Before any API ships, verify:

- [ ] OpenAPI spec is complete, linted, and reviewed.
- [ ] All endpoints return appropriate status codes (not just 200 and 500).
- [ ] Error responses follow RFC 9457 Problem Details format.
- [ ] Pagination is implemented for all collection endpoints.
- [ ] Rate limiting is configured and headers are returned.
- [ ] Authentication and authorization are enforced on every endpoint.
- [ ] Input validation returns 400/422 with descriptive errors.
- [ ] No PII is logged in request/response logs.
- [ ] Idempotency keys are supported for POST operations that create resources.
- [ ] Breaking changes have been versioned appropriately.
- [ ] Performance: response times are acceptable under expected load.
- [ ] Security: CORS, CSP, and security headers are configured.
- [ ] Documentation: developer portal / README is updated.
