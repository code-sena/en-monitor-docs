# Technical Security Rules

> Mandatory technical controls that apply to all project code.
> These rules complement the security policy (`security-policy.md`) with
> concrete implementation practices.

---

## OWASP Top 10 — Controls per category

### A01 — Broken Access Control

```
❌ BAD — trusting frontend data: const userId = req.body.userId
✅ GOOD — extract from verified JWT token: const userId = req.user.sub
```

**Rules:**
- Every protected endpoint MUST have the authentication middleware applied
- Permissions are verified in the Use Case, not in the Controller
- A resource is only returned if the user has `[resource]:read` permission
- Write actions require `[resource]:write` or `[resource]:delete` permission

### A02 — Cryptographic Failures

**Rules:**
- Passwords: use **bcrypt** with cost factor ≥ 12. Never MD5 or SHA-1 for passwords
- JWTs: sign with RS256 (asymmetric) using configured private/public keys. Never HS256 in production with a weak secret
- Sensitive data in transit: HTTPS mandatory in all environments except local
- Sensitive data at rest: encrypt fields marked as PII as required (AES-256-GCM)
- Never log passwords, tokens, or sensitive PII (emails in certain contexts should be handled carefully)
- Password reset codes are hashed with HMAC-SHA256 using a server-side pepper

### A03 — Injection

**SQL/JPA:**
- Use parameterized queries/JPA repository methods with parameters. Never concatenate user input into SQL strings.
- Validate all inputs with a validation library (Bean Validation for Java backend; Zod for frontend validation where applicable)
- For REST APIs, enforce input validation at controller/adapter boundaries

### A04 — Insecure Design

- Every HU that exposes user data must undergo privacy review
- Bulk query endpoints have mandatory pagination (maximum [100] records per page)
- Internal IDs are business identifiers (not sequential auto-increment exposed as is in all contexts) - use non-guessable identifiers as designed; avoid exposing internal implementation details

### A05 — Security Misconfiguration

```
# Verification checklist per environment
□ Stack traces NOT visible in production
□ Security headers configured (e.g. Spring Security headers / reverse proxy):
  - X-Content-Type-Options: nosniff
  - X-Frame-Options: DENY
  - Content-Security-Policy defined
  - Strict-Transport-Security in production (HSTS)
□ Unnecessary ports closed
□ Development credentials NOT in production
□ CORS configured with explicit allowed origins (no wildcard with credentials)
```

### A06 — Vulnerable Components

**Rules:**
- Run dependency audits (npm audit for frontends; Maven dependency checks for backend) before each release
- **Critical/High** vulnerabilities block the deploy
- Renew dependencies regularly; review security advisories
- Pin dependencies appropriately; avoid unverified `latest` versions

### A07 — Identification and Authentication Failures

- JWT access token expiration: **15 minutes** (as implemented)
- Refresh tokens expiration: **7 days** with rotation on each use
- Rate limiting on authentication endpoints (password reset attempts: max 10 per window as configured)
- Account lockout after consecutive failed login attempts (as implemented)

### A08 — Software and Data Integrity Failures

- Verify Docker image checksums before using in production
- Third-party integrations must verify cryptographic signatures when applicable
- Validate message schemas for event-driven communication (when migrating to microservices with message broker)

### A09 — Security Logging and Monitoring Failures

- Every failed authentication must be logged (IP, timestamp, user-agent as applicable)
- Log sensitive operations (deletions, role changes) with who, when, and what was affected
- Security logs are retained according to policy (minimum 90 days)
- Configure alerts for suspicious activity (e.g., brute force attempts, elevated 401/403 rates)

### A10 — Server-Side Request Forgery (SSRF)

- URLs constructed from user input MUST be validated against an allowlist of permitted domains
- Do not fetch from private IPs (192.168.x.x, 10.x.x.x, 127.x.x.x) from the server

---

## User input handling

```java
// Backend (Java) example with Bean Validation in controller/adapter layer
@Valid @RequestBody CreateUserRequest request
```

```typescript
// Frontend (TypeScript) example with Zod for client-side validation
const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).trim(),
});
const parsed = CreateUserSchema.parse(data);
```

**Rule:** All external inputs (HTTP body, query params, path params, broker messages)
pass through a validation schema before reaching the domain.

---

## Secure error handling

```
Backend (Spring): return generic error responses; do not expose stack traces in production.
Use @ControllerAdvice for centralized error handling with trace IDs for correlation.
```

---

## Correlations

- Security policy (management, access, vault) → `00-governance/security-policy.md`
- System threat model → `05-architecture/security-threat-model.md`
- Authentication and JWT → `07-api/authentication.md`
- Observability and security logs → `13-operations/observability.md`
