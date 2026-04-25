---
description: "Cross-cutting security best practices covering secrets management, authentication, authorization, input validation, dependency security, and OWASP compliance."
---

# Security Best Practices

## Secrets Management

- ⛔ NEVER commit secrets, API keys, connection strings, or credentials to source control
- ⛔ NEVER log secrets — not even partially
- ⛔ NEVER pass secrets as command-line arguments (visible in process lists)
- ✅ Use environment variables or secret managers (Azure Key Vault, GitHub Secrets)
- ✅ Use `.gitignore` to exclude config files with secrets
- ✅ Use `git-secrets` or `gitleaks` as pre-commit hooks
- ✅ Rotate secrets on a regular schedule and after any suspected compromise
- ✅ Use managed identities to avoid secrets entirely where possible

## Authentication

- Use established auth libraries/frameworks — never roll your own
- Use OAuth 2.0 / OpenID Connect for web applications
- Use JWT with short expiration times (15 min access, longer refresh)
- Store tokens securely (HTTP-only cookies, not localStorage)
- Implement proper logout (invalidate tokens, clear sessions)
- Enforce multi-factor authentication for privileged operations
- Rate-limit authentication endpoints to prevent brute force

## Authorization

- Implement authorization at EVERY layer (API, service, data)
- Use role-based access control (RBAC) at minimum
- Check authorization on every request — don't trust client-side checks
- Use attribute-based access control (ABAC) for fine-grained permissions
- Log all authorization failures
- Default to deny — explicitly grant access, never implicitly

## Input Validation

- Validate ALL input on the SERVER side — client validation is UX only
- Use allowlists over denylists
- Validate type, length, format, and range
- Sanitize output based on context (HTML encoding, SQL parameterization, URL encoding)
- Use parameterized queries — NEVER string concatenation for SQL
- Validate file uploads: type, size, content (not just extension)
- Reject unexpected fields in API requests

## Dependency Security

- Keep dependencies up to date — security patches are critical
- Use `npm audit`, `dotnet list package --vulnerable`, `pip audit` regularly
- Pin dependency versions in lock files
- Review new dependencies before adding — check maintenance, known issues
- Use GitHub Dependabot or Renovate for automated updates
- Minimize dependency count — fewer dependencies = smaller attack surface
- Never use packages with known critical vulnerabilities

## OWASP Top 10 Awareness

1. **Broken Access Control** — enforce auth on every endpoint
2. **Cryptographic Failures** — encrypt sensitive data, use strong algorithms
3. **Injection** — parameterize queries, validate input
4. **Insecure Design** — threat model early, defense in depth
5. **Security Misconfiguration** — remove defaults, harden configs
6. **Vulnerable Components** — keep dependencies updated
7. **Identification & Auth Failures** — strong passwords, MFA, session management
8. **Software & Data Integrity Failures** — verify signatures, secure CI/CD
9. **Security Logging & Monitoring** — log security events, alert on anomalies
10. **Server-Side Request Forgery (SSRF)** — validate/sanitize URLs, restrict outbound

## API Security

- Use HTTPS everywhere — no exceptions
- Implement rate limiting and throttling
- Validate content types (`Content-Type` header)
- Return minimal error information to clients (no stack traces)
- Use CORS with specific origins — never `*` in production
- Implement request size limits
- Version your APIs for backward compatibility

## Secure Coding Practices

- Use prepared statements / parameterized queries for all database access
- Escape output based on context (HTML, JS, URL, CSS, SQL)
- Use Content Security Policy (CSP) headers
- Set security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`
- Use `SameSite` attribute on cookies
- Implement proper error handling — don't expose internal details
- Use constant-time comparison for secret values (prevent timing attacks)

## Incident Response

- Have a documented incident response plan
- Log security events with sufficient detail for forensics
- Monitor for anomalies in authentication patterns
- Alert on repeated authentication failures
- Have a process for emergency secret rotation
- Document and learn from security incidents (blameless postmortems)
