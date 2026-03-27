# Threat Model (Plan Phase)

## 1. System overview
- Summary: Spring Petclinic is a web application for managing pet owners, pets, and veterinary visits.
- Primary users: clinic staff (administrative users), system maintainers, and CI/CD operators.
- Trust boundaries:
  - Internet -> Web app
  - Web app -> Database
  - CI/CD -> Build artifact storage / registry

## 2. Assets (what we protect)
- A1: User accounts / admin accounts
- A2: PII data
- A3: Secrets (DB password, tokens)
- A4: Build artifacts & pipeline integrity

## 3. Entry points
- E1: Web endpoints (HTTP)
- E2: Admin endpoints
- E3: CI webhook trigger (GitHub -> Jenkins)
- E4: Dependency downloads (Maven)

## 4. Threat assessment method
- Method: STRIDE (Spoofing, Tampering, Repudiation, Information disclosure, DoS, Elevation of privilege)
- Risk rating: Likelihood (1-5) x Impact (1-5)

## 5. Key threats & mitigations (minimum list)
| ID | Threat | STRIDE | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---:|---:|---|---|
| T-01 | Credential leakage in repo/logs | I | 3 | 5 | Use Jenkins Credentials; secret scanning; block commits | DevOps |
| T-02 | Tampering build artifact | T | 2 | 5 | Immutable artifacts; signed tags (future); provenance (future) | DevOps |
| T-03 | Dependency with known CVE | T/I | 4 | 4 | Dependency policy; SBOM per release; scan in Build | Dev |
| T-04 | Unauthorized admin access | S/E | 3 | 5 | MFA; RBAC; audit logs | App team |
| T-05 | DoS on public endpoints | D | 3 | 3 | Rate limit; WAF (future) | Platform |

## 6. Residual risk & decision
- Accepted risks:
  - T-05 (DoS) residual risk accepted for Plan phase until operational controls are implemented.
- Deferred controls (to Code/Build phases):
  - Automated secret scanning in CI pull request checks
  - Dependency and SAST gating for release pipelines
  - Artifact signing/provenance verification

## 7. Review cadence
- Re-review this threat model at least once per release cycle or when architecture changes significantly.