# Security Policy

> Security is not a feature — it is a system property built from day one. This document
> defines the mandatory practices.
> Any deviation must be explicitly approved by the Tech Lead.

---

## Security principles

1. **Defense in Depth:** Multiple security layers. If one fails, the others contain the damage.
2. **Least Privilege:** Each component has only the minimum necessary permissions.
3. **Fail Secure:** In case of error, the system denies access, does not allow it.
4. **Security by Design:** Security controls are designed from the start, not added at the end.
5. **Zero Trust:** Always verify, never implicitly trust, even within the internal network.

---

## Authentication

### JWT (JSON Web Tokens)

| Property | Required value |
|----------|---------------|
| Signing algorithm | RS256 (asymmetric). Private/public keys configured via paths. |
| Access token expiration | 15 minutes (`exp`) (as implemented) |
| Refresh token expiration | 7 days (rotation on use as implemented) |
| Required claims | `sub` (userId), `iat`, `exp`, `jti` (unique token ID), `iss`, `scope`/roles as applicable |
| Client storage | Secure storage (httpOnly cookie for web where applicable) or secure mobile storage (Keychain/Keystore) |

**Prohibited in the payload:**
- Passwords
- Card data
- Full PII (only the user ID)

### Refresh Token

- Stored in the database as a hash (SHA-256), never in plain text
- Mandatory rotation on each use (one refresh token = one use)
- Invalidated on logout and on password change
- ALL active tokens invalidated if use of a revoked token is detected

---

## Authorization

### RBAC (Role-Based Access Control)

The system implements RBAC with system roles and permissions. Global roles are assigned via `UserSystemRole`. Home-scoped roles (`OWNER`, `MEMBER`) exist for home membership and are separate from global system roles.

**Permission model:**

```
Permission: [resource]:[action]
```

**Validation:**
- JWT signature and expiration are validated on each request (resource server)
- Each bounded context validates permissions for protected operations as per use cases
- Access control is enforced at the application/use case layer (not only controllers)

---

## Secure communication

### Transmission

- **HTTPS mandatory** in all environments except local
- TLS 1.2 minimum; TLS 1.3 recommended
- Certificates: Let's Encrypt (staging) / Corporate CA (production)
- HSTS enabled in production

### Internal service-to-service communication

As implemented (24-Oct-2026):
- Inside `energy-monitor-backend` (modular monolith), modules communicate in process through application events and calls across module boundaries allowed by Spring Modulith.
- Between services, internal REST under `/internal/**` is accepted only from loopback or the Docker network **and** with the shared `X-Internal-Token` header (`INTERNAL_API_TOKEN`), compared in constant time; the API gateway never forwards `/internal/**`. Mosquitto's `/internal/mqtt/auth/**` calls are restricted by network only.
- Events between services go through RabbitMQ (credentials `RABBITMQ_*`), reachable only inside Docker.
- There is no mTLS; traffic inside the Docker network is plain HTTP/AMQP, and HTTPS is terminated at the host's Nginx.

---

## Secret management

```
✗ NEVER in source code
✗ NEVER in committed .env
✗ NEVER in logs
✗ NEVER in client error messages
✓ Environment variables (injected by the orchestrator)
✓ Vault (HashiCorp Vault, AWS Secrets Manager, etc.)
✓ Kubernetes Secrets (encrypted with KMS)
```

**Secret rotation:**
- API keys: every 90 days
- TLS certificates: 60 days before expiration
- DB passwords: every 6 months or immediately if compromise is suspected
- JWT private keys: rotate per key management policy
- Password reset pepper: rotate with careful handling (invalidates existing codes)

---

## Input validation and sanitization

### General rules

1. **Never trust user input.** Validate at the edge (controller) before processing.
2. **Whitelist, not blacklist.** Define what is allowed, not only what is prohibited.
3. **Reject early.** If input is invalid, respond 400 and do not process further.

### SQL Injection — Prevention

```typescript
// ✗ VULNERABLE
const result = await db.query(`SELECT * FROM users WHERE email = '${userInput}'`);

// ✓ SAFE — always use prepared parameters
const result = await db.query('SELECT * FROM users WHERE email = $1', [userInput]);
```

### XSS — Prevention

```typescript
// ✗ VULNERABLE — rendering HTML without escaping
element.innerHTML = userProvidedContent;

// ✓ SAFE — use textContent or sanitize
element.textContent = userProvidedContent;
// or with library: DOMPurify.sanitize(userProvidedContent)
```

### Validation with Zod / Joi

```typescript
// Explicit validation schema in the controller
const CreateOrderSchema = z.object({
  clientId: z.string().uuid(),
  items: z.array(z.object({
    productId: z.string().uuid(),
    quantity: z.number().int().positive().max(1000),
    price: z.object({
      amount: z.number().positive(),
      currency: z.enum(['COP', 'USD']),
    }),
  })).min(1).max(50),
});
```

---

## OWASP Top 10 — Review checklist

| Vulnerability | Implemented control |
|---------------|-------------------|
| A01: Broken Access Control | RBAC + permission validation in each service |
| A02: Cryptographic Failures | TLS 1.2+, bcrypt for passwords, secrets in vault |
| A03: Injection | Prepared parameters in SQL, schema validation |
| A04: Insecure Design | Threat modeling in design, Security review |
| A05: Security Misconfiguration | IaC for configuration, review of defaults |
| A06: Vulnerable Components | Dependabot / Snyk for automatic updates |
| A07: Authentication Failures | JWT with rotation, brute-force protection |
| A08: Software Integrity Failures | Verify dependency checksums, SBOM |
| A09: Logging Failures | Logs without PII, centralized, with alerts |
| A10: SSRF | Whitelist of external URLs, do not follow redirects automatically |

---

## Audit and security logs

### Events that are ALWAYS recorded

```typescript
// Security events — store in a separate log, with retention > 1 year
const SECURITY_EVENTS = [
  'auth.login.success',
  'auth.login.failure',
  'auth.login.brute_force_detected',
  'auth.password.changed',
  'auth.token.revoked',
  'auth.unauthorized_access_attempt',
  'data.pii.accessed',
  'admin.role.changed',
  'admin.user.deleted',
];
```

**Required fields in security logs:**
- `userId` (or `ANONYMOUS` if not authenticated)
- `sourceIp`
- `action`
- `resource`
- `result` (SUCCESS / FAILURE)
- `timestamp`

---

## Vulnerability process

### What to do if you find a vulnerability

1. **Do not commit it to the public repo** or discuss it in open channels
2. Immediately notify the Tech Lead via a private channel
3. Create a private issue or a restricted repository issue
4. Severity is assigned (CVSS score or internal classification)
5. Remediated in the current sprint if critical, in the next sprint if high

### Remediation SLAs

| Severity | Remediation time |
|----------|----------------|
| Critical (CVSS 9-10) | 24 hours |
| High (CVSS 7-8.9) | 1 week |
| Medium (CVSS 4-6.9) | 1 month |
| Low (CVSS < 4) | Next security review |

---

## Correlations

- Security non-functional requirements → `04-requirements/non-functional.md`
- ADR on authentication → `05-architecture/decisions/`
- Security event observability → `13-operations/observability.md`
- RBAC implemented in → `09-microservices/services/02-security-service/` (`en-monitor-security-service`)
