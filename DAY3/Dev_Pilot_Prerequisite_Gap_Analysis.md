# SBOM & VEX Dev Pilot: Prerequisite, Access & Gap Analysis


---

## 1. Executive Summary

This document establishes the implementation baseline for running a single-workload **Dev pilot** using **Syft**, **Trivy**, and **OpenVEX** in our existing **Azure DevOps** pipeline, while designing for seamless migration to **GitHub Actions**.

All core runtime prerequisites—including outbound internet access and container registry pull access—have been validated without requiring changes to shared deployment workflows.

---

## 2. Tooling, Extensions & Installation Strategy

| Area | Azure DevOps (Pilot Implementation) | GitHub Actions (Migration Target) | Notes & Guidelines |
| :--- | :--- | :--- | :--- |
| **Pipeline Agents** | Microsoft-hosted (`ubuntu-latest`) or Self-hosted / ARC runners | GitHub-hosted (`ubuntu-latest`) or ARC runners (`arc-runner-set`) | Requires outbound HTTPS (port 443) for binary downloads and vulnerability DB sync. |
| **Marketplace Extensions** | None required (portable CLI approach) | None required (portable CLI or official actions) | Direct CLI execution avoids platform lock-in and dependency on organization-level extension approvals. |
| **Installation Path** | `$(Agent.TempDirectory)/bin` | `$RUNNER_TEMP/bin` | **Non-root requirement:** Tools install to user-writable temp storage, preventing `/usr/local/bin` permission failures observed on non-root runners. |
| **Tool Footprint** | • Syft CLI (~50MB)<br>• Trivy CLI (~80MB)<br>• vexctl CLI (~30MB) | Identical | Lightweight single-binary Go tools with zero external runtime dependencies. |

---

## 3. Service Connections, Permissions & Artifact Publishing

| Component | Azure DevOps (Current Pilot) | GitHub Actions Parity | Status / Work Item Mapping |
| :--- | :--- | :--- | :--- |
| **ACR Pull Access** | Azure Service Connection with `AcrPull` on `heliosdevaksregistry`. | Workload Identity Federation (OIDC) with `AcrPull`. | **AB#27252** (Validated live: `heliosdevaksregistry` is reachable and queryable). |
| **Vulnerability DB Egress** | Outbound HTTPS (port 443) to `ghcr.io` / `github.com`. | Outbound HTTPS (port 443). | **AB#27254** (Validated live: tool downloads and DB fetches succeed). |
| **SARIF Report Ingestion** | **Publish to Build Drop (`CodeAnalysisLogs`):**<br>Published using native `PublishBuildArtifacts@1`. Compatible with the Azure DevOps SARIF Viewer extension.<br>*(Note: `PublishSecurityAnalysisResult@1` is dependent on the optional Microsoft Security DevOps extension, so standard artifact publishing serves as our guaranteed baseline).* | `github/codeql-action/upload-sarif@v3` (renders in GitHub Code Scanning if licensed, or retained as build drop). | Native build artifact drops serve as the reliable baseline across both platforms. |
| **SBOM & Evidence Storage** | Native `PublishBuildArtifacts@1` stores CycloneDX, SPDX, and OpenVEX JSON files in the pipeline drop. | `actions/upload-artifact@v4`. | **ADR 0003 §2 / §6** (Build drop storage for pilot; OCI referrer registry attachment evaluated separately). |

---

## 4. OpenVEX Integration Specifications

| Parameter | Specification | Details |
| :--- | :--- | :--- |
| **Authoring CLI** | `vexctl` (OpenSSF) | Single binary utility used to create, validate, and inspect OpenVEX JSON-LD documents. |
| **File Location** | `.security/vex.json` | Committed to repository root; changes follow GitOps review cycles. |
| **Pilot Test Statement** | `status: "not_affected"` | Uses the standard OpenVEX specification justification:<br>`"justification": "vulnerable_code_not_in_execute_path"`<br>*(Conforms strictly to OpenVEX v0.2.0 schema validation).* |
| **Trivy Ingestion** | `--vex .security/vex.json` | Trivy natively parses the statement and suppresses matching CVEs from pipeline gate logic. |
| **Governance Scope** | Technical validation only | Validates filtering behavior. Official approval authority ([AB#27251](https://dev.azure.com/qcellsces/Helios/_workitems/edit/27251)) remains with leadership and does not block the pilot. |

---

## 5. Technical Rules & Proven Guardrails

These guardrails originate from ADR 0003 and ADR 0004 to eliminate known silent-failure modes:

1. **Digest-Bound Inspection (Never Scan Tags):**
   * The pipeline captures the immutable image digest from build output (`<image>@sha256:...`) and scans by digest. Scanning mutable tags (`:latest` or `:sha-xxxx`) can produce evidence for code that was never deployed.
2. **Platform Architecture Invariant:**
   * Syft resolves architecture based on host runners by default. The command explicitly includes `--platform linux/amd64` to prevent silent misclassification on multi-platform runners (ADR 0003 §1).
3. **Disable File Catalogers:**
   * Syft must be invoked with `--select-catalogers "-file"` to prevent component bloat (~3,000 file entries vs. ~230 packages), ensuring clean format parity between CycloneDX and SPDX (ADR 0003 §2a).
4. **Report-Only Gate Policy:**
   * Per ADR 0005, the pilot runs strictly non-blocking (`--exit-code 0` or warn-only). Deployment gating is decoupled and owned under promotion gate initiatives (AB#27485).

---

## 6. Pilot Readiness & Parity Status

- [x] **Tools:** Syft, Trivy, and `vexctl` run as standalone binaries without root privileges.
- [x] **Extensions:** Zero third-party Azure DevOps marketplace extensions required.
- [x] **Permissions:** Confirmed read/pull access to `heliosdevaksregistry`.
- [x] **Network:** Confirmed outbound HTTPS to `ghcr.io` and GitHub releases.
- [x] **Reporting:** Native `PublishBuildArtifacts@1` configured for SARIF, SBOM, and VEX drops.
- [ ] **GitHub Actions Parity:** **Designed for portability / validation pending** (CLI scripts are runner-agnostic; live execution will be verified during the pilot).
- [ ] **Workload Confirmation:** Awaiting final selection of the pilot candidate service (`digital-twin-api` or `helios-audit-mcp-server` recommended).

---

## 7. Next Action

Upon confirmation of the pilot candidate service, the 11 pilot steps will be executed in report-only mode to generate baseline evidence for decisions `SBOM-D01` through `SBOM-D14`.
