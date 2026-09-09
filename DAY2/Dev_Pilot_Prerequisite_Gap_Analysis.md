# Dev Pilot: Prerequisite, Access, and Gap Analysis (Syft, Trivy, OpenVEX)

---


## 1. Tooling, Extensions & Infrastructure

| Area | Azure DevOps (Current Pilot) | GitHub Actions (Migration Parity) | Requirements & Notes |
| :--- | :--- | :--- | :--- |
| **Pipeline Agents** | Microsoft-hosted (`ubuntu-latest`) or Self-hosted / ARC runners | GitHub-hosted (`ubuntu-latest`) or ARC runners (`arc-runner-set`) | Agents require outbound HTTPS (port 443) to download tool binaries and vulnerability database feeds. |
| **Marketplace Extensions** | *Optional:* `AquaSecurity.trivy-official` | Native Marketplace Actions (`anchore/sbom-action`, `aquasecurity/trivy-action`) | **Recommendation: Use direct CLI downloads** rather than marketplace extensions. Portable scripts run identically on both ADO and GitHub with zero vendor lock-in. |
| **Tool Installation** | Download standalone binaries to `$(Agent.TempDirectory)/bin` | Download to `$RUNNER_TEMP/bin` or use composite actions | **Non-root requirement:** In PR #291, non-root ARC runners failed when writing to `/usr/local/bin`. Installation must target agent temp paths without `sudo`. |
| **Tool Footprint** | • Syft CLI (~50MB)<br>• Trivy CLI (~80MB)<br>• vexctl CLI (~30MB) | Identical | Lightweight single-binary tools with zero runtime language dependencies (no Node.js or Python runtime required). |

---

## 2. Service Connections & Permissions

| Resource / Action | Azure DevOps Requirement | GitHub Actions Parity | Mapping to Maurice's Work Items |
| :--- | :--- | :--- | :--- |
| **ACR Access (Image Inspection)** | Service Connection with `AcrPull` (or existing `AcrPush`) on `heliosdevaksregistry`. | Workload Identity Federation (OIDC) with `AcrPull`. | **AB#27252** (Already provisioned: `helios-sbom-reconciler-github-actions` with federated OIDC credentials). |
| **Vulnerability DB Updates** | Outbound HTTPS access to `ghcr.io` (Trivy DB mirror) and `github.com`. | Outbound HTTPS access to `ghcr.io` / `github.com`. | **AB#27254** (Workstream C). If strict network isolation exists, Trivy can cache the database internally in ACR. |
| **SARIF Report Ingestion** | Native ADO Task: `PublishSecurityAnalysisResult@1` (renders in Security / Scans tab). | Native action: `github/codeql-action/upload-sarif@v3` (renders in Code Scanning). | Native capability on both platforms without commercial add-on licensing. |
| **Artifact Storage** | Native `PublishBuildArtifacts@1` stores CycloneDX, SPDX, and VEX files in the build drop. | `actions/upload-artifact@v4`. | **ADR 0003 §2 / §6**. For the pilot, standard build drop storage is sufficient. (ACR OCI referrer attachment is tracked separately). |

---

## 3. OpenVEX Integration & Prerequisites

| Item | Requirement | Details |
| :--- | :--- | :--- |
| **Authoring Tool** | `vexctl` CLI (OpenSSF) | Single standalone binary downloaded during build or committed to repo. |
| **File Location** | `.security/vex.json` | Committed directly to the repository (GitOps-friendly, tracks audit history in git commits). |
| **Pilot Implementation** | Single mock/test statement | Marks one known finding as `not_affected` with justification `code_not_reachable`. |
| **Trivy Ingestion** | `--vex .security/vex.json` | Trivy natively evaluates the VEX file and suppresses matching CVEs from pipeline output. |
| **Governance Gap** | CTO Approval ([AB#27251](https://dev.azure.com/qcellsces/Helios/_workitems/edit/27251)) | Naming an authoritative approver for customer-facing statements remains open. **Does not block the Dev pilot**, as the pilot only validates the technical filtering mechanism. |

---

## 4. Technical Gaps, Gotchas & Mitigations

These are proven findings documented in ADR 0003 and ADR 0004 that must be observed during the pilot:

1. **Tag vs. Digest Resolution (Silent Failure Trap):**
   * *Gap:* If the pipeline scans a mutable tag (`:latest` or `:sha-xxxx`), tag movement can result in evidence describing an image that was never actually deployed.
   * *Mitigation:* The pilot must capture the **immutable digest** directly from the build/push step output (`repo@sha256:...`) and pass the digest to Syft/Trivy.
2. **Platform Architecture Invariant:**
   * *Gap:* Syft defaults to the runner's host architecture. If an `amd64` image is inspected on an `arm64` agent, Syft silently fails or scans the wrong platform.
   * *Mitigation:* Always specify `--platform linux/amd64` explicitly in the Syft invocation (ADR 0003 §1).
3. **File Cataloger Discrepancy:**
   * *Gap:* By default, Syft includes individual file catalogers, causing CycloneDX to output ~3,000 components compared to SPDX's ~230 packages.
   * *Mitigation:* Always pass `--select-catalogers "-file"` to ensure uniform, package-level parity across formats (ADR 0003 §2a).
4. **Enforcement Scope (Report-Only):**
   * *Gap:* Coupling this pilot with deployment blocking would trigger dependencies on digest pinning (AB#24116) and promotion gates (AB#27485).
   * *Mitigation:* Per ADR 0005, the pilot must run **strictly report-only** (`--exit-code 0` or warn-only on findings).

---

## 5. Work Item Mapping (Maurice's ADO Backlog)

| Pilot Requirement | Mapped ADO Work Item | Status in Backlog |
| :--- | :--- | :--- |
| **Federated Identities & ACR Access** | **AB#27252** (Provision federated identities, signing, ACR permissions) | Provisioned for dev/qa/prod |
| **Shared Tooling & Invocation** | **AB#27254** (Build shared digest-bound SBOM, scan, and attachment tooling) | Merged in `Helios-release-orchestrator` (PR #277) |
| **OpenVEX Rendering & Filtering** | **AB#27259** (Render, validate, attach OpenVEX from approved records) | Reassigned to Sam Chai |
| **VEX Governance & Approver Authority** | **AB#27251** (Approve architecture decisions & VEX authority) | With CTO (blocks publication, not pilot) |
| **Pipeline Promotion Enforcement** | **AB#27485** / **AB#27261** (Enforce supply-chain evidence at promotion) | Deferred under ADR 0005 |

---

## 6. Pilot Readiness Checklist

- [x] **Tools:** Syft, Trivy, and `vexctl` can be downloaded dynamically without admin/root rights.
- [x] **Extensions:** Zero third-party extensions required; direct CLI ensures 100% parity between ADO and GitHub Actions.
- [x] **Permissions:** Standard `AcrPull` on `heliosdevaksregistry` is sufficient.
- [x] **Artifacts:** Native ADO pipeline storage handles CycloneDX, SPDX, SARIF, and VEX files.
- [x] **Portability:** The exact same script logic will run in GitHub Actions with no changes.
- [ ] **Pending:** Candidate repo selection from the team (`digital-twin-api` or `helios-audit-mcp-server` recommended).

---

