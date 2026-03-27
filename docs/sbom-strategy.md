# SBOM Strategy (Plan Phase)

## 1. Purpose
SBOM (Software Bill of Materials) is used to:
- Know exactly which components are included in an artifact
- Support vulnerability response (which release is affected)
- Support compliance & supply-chain visibility

## 2. Scope
- Repo model: mono-repo
- SBOM unit (choose one):
  - [ ] Per repository
  - [x] Per build artifact (recommended)
  - [ ] Per container image (later)
- For this project:
  - Artifact: target/*.jar
  - SBOM generated on each successful main-branch build and each release build

## 3. Standard & format
- Standard (choose one):
  - [x] CycloneDX
  - [ ] SPDX
- Output formats:
  - JSON (preferred for tools)
  - XML (optional)

## 4. Storage & retention
- Store SBOM as Jenkins artifact attached to the build
- Retain SBOM artifacts for at least as long as release artifacts are retained
- (Optional later) publish to an SBOM repository or artifact registry

## 5. Ownership
- SBOM owner: DevOps / Build Engineering (TBD assignee)
- Approval required when:
  - Adding new critical dependency group
  - Changing build toolchain or repository sources

## 6. Link to provenance (future)
- Plan to add build provenance (SLSA) to ensure artifact traceability.

## 7. References
- [Security Requirements](./security-requirements.md)
- [Release Criteria](./release-criteria.md)