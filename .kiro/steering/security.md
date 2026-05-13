# Security Standards

Security is a core product principle in CasLearn OLMS, not a bolt-on. Every engineer — regardless of role — is a security engineer when writing CasLearn code.

## Identity & Access

- **Authentication:** OAuth 2.0 / JWT issued by `caslearn-auth-service`. Tokens ≤ 60 min, refresh token rotation on every refresh.
- **Password storage:** bcrypt with work factor ≥ 12. Never log passwords; never return them in any response.
- **Lockout:** 5 failed attempts in 10 min → 15-min lockout + `LOGIN_LOCKED` audit event.
- **Authorization enforced at two layers:**
  1. **Istio `AuthorizationPolicy`** at the mesh (service-to-service + role claim checks).
  2. **Spring Security** in each service (role + ownership checks).
- **Roles:** exactly three — `ADMIN`, `INSTRUCTOR`, `STUDENT`. No custom roles in MVP.
- **Role escalation guard:** reject any request (other than admin role-change) containing a `role` field in body/header/query — even if it matches the current role.

## Transport & Secrets

- **mTLS STRICT** between every backend service via Istio.
- **TLS 1.2+** only on the public ingress.
- **Secrets:** Kubernetes Secrets + Sealed Secrets (or Vault OSS). Never in Git, never in logs, never in container images.
- **Secret scan:** Gitleaks in CI blocks on any finding.
- **Key rotation:** JWT signing keys rotated every 90 days; overlap window of 24h.

## Data Protection

- **At rest:** MySQL encryption at rest (storage-level). Object storage (MinIO/S3) encrypted at rest.
- **In transit:** TLS everywhere.
- **PII minimization:** store only what's necessary. Never log PII (student IDs are okay; names, emails, grades are not).
- **FERPA:** student data accessible only to that student, instructors of their courses, and admins. Every access by anyone other than the student themselves is audit-logged.
- **Right to export / delete:** student data export in 72h; deletion anonymizes PII while preserving aggregate analytics.

## Audit Logging

- Every auth, authz denial, grade change, enrollment change, content deletion, role change, and admin action emits an `Audit_Log` entry.
- **Append-only.** No API path may mutate or delete past entries.
- **Tamper-evident.** Each entry's hash = `H(entry.payload || previous.hash)`.
- **Failure mode:** if audit write fails, the triggering request is rejected with HTTP 500.

## Input Validation

- **Every public endpoint** validates against the OpenAPI spec at the Gateway.
- **Every backend service** re-validates against its own Bean Validation constraints (defense in depth).
- **File uploads:** MIME-type allow-list + size limit per endpoint. Virus scan optional post-MVP.
- **Rate limiting:** 100 req/min per JWT subject at the Gateway by default. Per-endpoint overrides for expensive ops.
- **Output encoding:** React escapes by default; never use `dangerouslySetInnerHTML` without a reviewed sanitizer.

## OWASP Top 10 Coverage

| Risk                                | Control                                                                 |
|-------------------------------------|-------------------------------------------------------------------------|
| A01 Broken Access Control           | Mesh + app-layer RBAC; ownership checks; PBT property CP-1/CP-2/CP-3    |
| A02 Cryptographic Failures          | TLS everywhere; bcrypt; encrypted storage; Sealed Secrets               |
| A03 Injection                       | Parameterized JPA; Bean Validation; OpenAPI validation at Gateway       |
| A04 Insecure Design                 | Threat modeling per service in `caslearn-docs/threat-models/`              |
| A05 Security Misconfiguration       | Terraform + Helm in Git; no manual changes to prod                      |
| A06 Vulnerable Components           | Trivy + Dependabot; block on high/critical                              |
| A07 Authentication Failures         | Lockout; refresh rotation; MFA candidate post-MVP                       |
| A08 Software & Data Integrity       | Signed container images (cosign post-MVP); branch protection + reviews  |
| A09 Logging & Monitoring Failures   | Structured logs to Loki; alerts in Alertmanager                         |
| A10 SSRF                            | Allow-list outbound URLs; no user-controlled redirects                  |

## CI Security Gates

Every `*-service` and `*-ui` repo pipeline runs, and blocks on findings at severity `HIGH` or above:

1. **SAST** — SonarQube
2. **Dependency scan** — Trivy (fs) + Dependabot alerts
3. **Container scan** — Trivy (image)
4. **Secret scan** — Gitleaks
5. **License scan** — Trivy license report; block on denylisted licenses (GPL, AGPL in dependencies we'd ship)

## Threat Model Highlights

- **AI data leak:** AI assistant must only retrieve from materials the caller is authorized to see. Enforced by filtering the retrieval candidate set pre-prompt. See CP-6.
- **Mass enumeration:** 401 responses don't reveal whether a username exists. 404 preferred over 403 when existence itself is sensitive.
- **Grade tampering:** Grade amendments require justification; old + new values retained.
- **Supply chain:** pin base images by digest; scan on every build; review every new dependency.

## Security Review Checklist for a New Feature

Before merging a feature PR that touches auth, authz, PII, or file handling, confirm:

- [ ] Threat model entry added or updated
- [ ] RBAC at mesh layer updated if new service-to-service path
- [ ] RBAC at app layer updated with ownership check
- [ ] Audit event emitted for any sensitive action
- [ ] Input validation at Gateway + service
- [ ] Output encoding verified in UI
- [ ] Secrets referenced from Vault/Sealed Secrets
- [ ] Tests include negative authz cases
- [ ] Logs scrubbed of PII
