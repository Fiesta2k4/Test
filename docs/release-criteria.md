# Release Criteria (Plan Phase)

## Required (must pass)
- RC-01: Plan Gate passes (required docs exist and ownership is assigned)
- RC-02: Build produces artifact (jar)
- RC-03: No secrets committed (policy in place; automated enforcement enabled in Code/Build phases)
- RC-04: Dependency policy acknowledged (SBOM strategy exists)

## Optional (later phases)
- RC-05: Unit tests pass and test reports published
- RC-06: SAST has no High/Critical findings
- RC-07: Dependency scanning has no Critical vulnerabilities
- RC-08: Container image scan (if packaging containers)

## Evidence checklist
- [ ] [Security Requirements](./security-requirement.md) reviewed and approved
- [ ] [Threat Model](./threat-model.md) reviewed and approved
- [ ] [SBOM Strategy](./sbom-strategy.md) reviewed and approved
- [ ] Build log/artifact link attached in release record