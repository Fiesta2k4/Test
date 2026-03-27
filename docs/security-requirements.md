# Security Requirements (Plan Phase)

## 1. Scope
- System: Spring Petclinic
- Repo model: mono-repo
- Environments: dev / staging / prod
- In-scope components:
  - Web app
  - Database
  - CI/CD (Jenkins)
  - Container registry (when container image publishing is enabled)
- Out-of-scope:
  - Third-party SaaS services not operated by the project team
  - End-user device security

## 2. Data classification
| Data type | Examples | Sensitivity | Storage/Transit rules |
|---|---|---|---|
| Credentials/Secrets | API keys, DB password | Critical | Must be in secret manager / Jenkins credentials; never in repo/logs |
| PII | name, email, phone | High | Encrypt at rest & in transit; do not log raw PII |
| Session/Auth tokens | JWT, cookies | High | Secure cookie flags, rotation, short TTL |
| Operational logs | request logs | Medium | No secrets/PII; access controlled |
| Public data | docs | Low | No restrictions |

## 3. Security requirements (must)
### Authentication & Authorization
- SR-AUTH-01: All admin access must require MFA (where applicable).
- SR-AUTH-02: Enforce least privilege roles for application and CI.
- SR-AUTH-03: Service-to-service calls must be authenticated (if/when microservices are introduced).

### Secrets management
- SR-SEC-01: Secrets must not be committed to Git.
- SR-SEC-02: Secrets must be injected via Jenkins Credentials / secret store.
- SR-SEC-03: Rotate credentials on suspected leak.

### Transport & storage protection
- SR-ENC-01: TLS for all external traffic.
- SR-ENC-02: Sensitive data encrypted at rest (DB, backups).

### Logging & monitoring
- SR-LOG-01: Logs must not contain secrets or raw PII.
- SR-LOG-02: Security events must be logged: login failures, permission denied, config changes.

### Dependency & supply chain
- SR-SC-01: Dependencies must come from approved sources (Maven Central, npm registry, internal repo).
- SR-SC-02: SBOM must be generated for every release artifact (see [SBOM Strategy](./sbom-strategy.md)).

## 4. Security acceptance criteria (release gate)
A build is eligible for release only if:
- SAC-01: Plan Gate passes (required docs + ownership present; see [Release Criteria](./release-criteria.md)).
- SAC-02: Build passes (artifact produced).
- SAC-03: No critical secrets found in repo history for this change.
- SAC-04: (Optional later) SAST/Dependency scan has no Critical issues (Phase Code/Build).

## 5. Owners
- Product owner: TBD (must be assigned before production release)
- Tech owner: Spring Petclinic maintainers
- Security reviewer: Security Champion (AppSec) - TBD