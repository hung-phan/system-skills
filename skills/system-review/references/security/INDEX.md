---
name: security
description: Index of security skills — threat modeling (STRIDE/DREAD/PASTA), authentication (OAuth2/OIDC/SAML/JWT/WebAuthn), authorization (RBAC/ABAC/ReBAC), encryption at rest and in transit, secrets management, OWASP Top 10, CSRF/XSS/SQLi/SSRF/IDOR, zero-trust, mTLS, defense in depth, audit logging, compliance (SOC 2/HIPAA/GDPR/PCI). Use when designing authn/authz, threat-modeling a feature, hardening an endpoint, storing secrets, or chasing a security finding.
---

# Security

How a system **resists abuse**. Most breaches trace to a missing pattern in this folder, not a novel attack.

## Skills

### Foundations

| Skill | Description |
|-------|-------------|
| [Threat Modeling](threat-modeling/) | STRIDE, DREAD, PASTA, attack trees. Build the threat model **before** the feature. |
| [Defense in Depth](defense-in-depth/) | Stack independent controls. Each layer assumes the previous one failed. |
| [Zero Trust](zero-trust/) | Never trust the network. Authenticate every call. BeyondCorp model. |

### Authentication

| Skill | Description |
|-------|-------------|
| [Authentication (Authn)](authn/) | Sessions vs tokens, password hashing (Argon2/bcrypt), MFA, WebAuthn / passkeys. |
| [OAuth 2.0 / OIDC](oauth-oidc/) | Authorization code flow + PKCE, refresh tokens, OIDC ID tokens, common misuse. |
| [SAML](saml/) | Enterprise SSO. When you must care; how it differs from OIDC. |
| [JWT](jwt/) | Claims, signing, verification. Why "alg: none" still bites. JWT-vs-session decision. |

### Authorization

| Skill | Description |
|-------|-------------|
| [Authorization (Authz)](authz/) | RBAC, ABAC, ReBAC. Policy engines (OPA, Cedar, Zanzibar). Decision tree. |
| [Multi-Tenancy](multi-tenancy/) | Per-tenant isolation. Pool-vs-silo trade-offs. Avoiding cross-tenant data leaks. |

### Crypto

| Skill | Description |
|-------|-------------|
| [Encryption in Transit](encryption-in-transit/) | TLS 1.3, certificate management, HSTS, mTLS. |
| [Encryption at Rest](encryption-at-rest/) | Disk-level vs application-level. Envelope encryption. KMS. |
| [Secrets Management](secrets-management/) | Vault, AWS Secrets Manager, GCP Secret Manager, K8s sealed secrets. Rotation, revocation. |
| [Hashing & Password Storage](password-storage/) | Argon2id, scrypt, bcrypt. PBKDF2 only when forced. Pepper vs salt. |

### Application security

| Skill | Description |
|-------|-------------|
| [OWASP Top 10](owasp-top-10/) | The 10 web app risks every reviewer should hold in their head. Mitigation per item. |
| [SQL Injection](sqli/) | Parameterized queries, ORM safe layers, what input is "safe". |
| [XSS](xss/) | Stored, reflected, DOM-based. CSP. Output encoding by context. |
| [CSRF](csrf/) | SameSite cookies, double-submit tokens, anti-CSRF headers. |
| [SSRF](ssrf/) | Server-side request forgery. IMDS metadata, internal IP egress, allowlists. |
| [IDOR](idor/) | Insecure direct object reference. Authz checks at the data layer. |
| [Deserialization](deserialization/) | Untrusted input → executable code. Why Pickle / Java Serialization / unsafe YAML are landmines. |

### Operations

| Skill | Description |
|-------|-------------|
| [Audit Logging](audit-logging/) | What to log, what NOT to log, immutability, retention. PII risk. |
| [Compliance](compliance/) | SOC 2, ISO 27001, HIPAA, GDPR, PCI-DSS, FedRAMP. What each demands; how to map controls. |
| [Vulnerability Management](vulnerability-management/) | CVE triage, dependency scanning (SCA), secret scanning, SBOM. |
| [Incident Response (Security)](security-incident-response/) | Containment, eradication, recovery, lessons. Differs from operational IR. |
| [mTLS](mtls/) | Mutual TLS between services. Certificate rotation. Service mesh integration. |

## Decision Trees

### Pick an authz model

| Need | Use |
|------|-----|
| A few roles (admin/editor/viewer) | RBAC |
| Decisions depend on attributes (department, location, time of day) | ABAC |
| Decisions depend on **relationships** (sharing a doc with X) | ReBAC (Zanzibar / OpenFGA) |
| Cross-system policy with code-as-policy | OPA (Rego) or Cedar |
| Per-resource ACLs at scale | ReBAC |

### "What kind of token?"

| Need | Use |
|------|-----|
| Server-rendered web app, single domain | Server-side session cookie (HttpOnly + Secure + SameSite) |
| SPA + API on same domain | Same; SameSite=Lax/Strict mitigates CSRF |
| SPA + API on different domains | OAuth 2.0 + PKCE → access token (short) + refresh (rotated) |
| Mobile native app | Same as SPA; secure storage (Keychain / Keystore) |
| Service-to-service inside trust boundary | mTLS or short-lived signed tokens |
| Federated SSO into multiple apps | OIDC (preferred) or SAML (legacy enterprise) |

### "How do I store secrets?"

| Where it runs | Use |
|---------------|-----|
| AWS | Secrets Manager (rotated) or Parameter Store (cheap) |
| GCP | Secret Manager |
| Azure | Key Vault |
| Kubernetes | External secrets operator → Vault / cloud secret manager |
| Local dev | `.env` (gitignored) + `direnv` / `1Password CLI` |
| Anywhere | HashiCorp Vault (cross-cloud) |

## Rules of Thumb

- **Authentication ≠ authorization.** Authn = "who are you?"; Authz = "are you allowed?"
- **Authorize at the data layer**, not just the route. The route is bypassable; the data layer is the truth.
- **Trust no input. Encode by output context.** XSS is encoding-specific (HTML vs JS vs URL vs CSS).
- **Parameterize every query.** String concat into SQL is malpractice in 2026.
- **Rotate keys on a schedule.** Plan rotation before you need it.
- **No secrets in env vars committed to images.** Use a secret manager.
- **Log security events, not security secrets.** Hash IDs, redact PII, but capture *what happened*.
- **Default deny.** If a permission isn't explicitly granted, the answer is no.
- **The vulnerability you don't see is the one that breaks you.** Run SAST + SCA + secret scanning in CI.

## See Also

- `../communication/api-gateway/` — auth at the edge
- `../communication/idempotency/` — preventing replay attacks
- `../reliability/observability/` — overlap with audit logging
- `../data-systems/replication/` — encryption-at-rest implications
- `../architecture-patterns/service-mesh/` — mTLS at scale

## References

- OWASP — https://owasp.org/Top10/
- OWASP ASVS — https://owasp.org/www-project-application-security-verification-standard/
- OWASP Cheat Sheets — https://cheatsheetseries.owasp.org/
- *Building Secure and Reliable Systems* (free) — https://sre.google/books/building-secure-reliable-systems/
- NIST SP 800-63 (digital identity) — https://pages.nist.gov/800-63-3/
- IETF OAuth 2.0 — https://datatracker.ietf.org/doc/html/rfc6749
- OIDC — https://openid.net/specs/openid-connect-core-1_0.html
- Google Zanzibar (ReBAC) paper — https://research.google/pubs/pub48190/
- BeyondCorp papers — https://research.google/pubs/pub43231/
