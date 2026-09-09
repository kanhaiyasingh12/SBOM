# Tooling Evaluation & Recommendations for SBOM and VEX

---

## 1. SBOM Standard & Generation Tools

### Standard: CycloneDX vs. SPDX

| Feature | CycloneDX (1.5/1.6) | SPDX (2.2/2.3) |
| :--- | :---: | :---: |
| **Built for security & CVE tracking** |  Native |  Limited / Added later |
| **Native VEX support** |  Native |  Limited in 2.x |
| **Scanner compatibility (Trivy, Grype)** |  Full |  Partial |
| **Regulatory compliance (EO 14028 / NTIA)** |  Yes |  Yes |

> ** Recommendation: CycloneDX 1.5/1.6 (JSON)**  
> Built specifically for AppSec and DevSecOps workflows. If a client or regulator explicitly requires SPDX, our tooling can output both simultaneously.

---

### Generation Tool Options

| Tool | Outputs CycloneDX | Outputs SPDX | Dual Output (1-Run) | Azure DevOps | GitHub Actions |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Syft (Anchore)** | ✅ | ✅ | ✅ | ✅ Easy CLI | ✅ Official Action |
| **Trivy (Aqua)** | ✅ | ✅ | ⚠️ Separate runs | ✅ Extension | ✅ Official Action |
| **MS sbom-tool** | ❌ | ✅ (Only) | ❌ | ✅ Native Task | ✅ Official Action |
| **cdxgen (OWASP)** | ✅ | ❌ | ❌ | ⚠️ Needs Node.js | ⚠️ CLI |

> ** Recommendation: Syft**  
> Generates **both CycloneDX and SPDX in a single command**, offers best-in-class container inspection, and runs identically across Azure DevOps agents and GitHub Actions runners.

---

## 2. Vulnerability Scanning Tools

| Feature / Capability | Trivy (Aqua Security) | Grype (Anchore) | Snyk |
| :--- | :---: | :---: | :---: |
| **Scans SBOMs directly** | ✅ | ✅ | ✅ |
| **Native VEX filtering (`--vex`)** | ✅ | ✅ (OpenVEX) | ❌ Proprietary waivers |
| **CISA Known Exploited Vulns (KEV)** | ✅ Native | ❌ | ⚠️ Separate |
| **Pipeline gates (`--exit-code`)** | ✅ Native | ✅ Native | ✅ Native |
| **SARIF output (Azure Security tab / GHAS)** | ✅ | ✅ | ✅ |
| **Azure DevOps & GitHub Support** | ✅ Official | ⚠️ CLI only | ✅ Official |
| **Cost / Licensing** | ✅ Free (Apache 2.0) | ✅ Free (Apache 2.0) | ❌ Paid / Per-developer |

> ** Recommendation: Trivy**  
> Provides native `--vex` support to suppress non-exploitable findings before evaluating gate thresholds, emits native SARIF reports for pipeline security tabs, and has official extensions for both ADO and GitHub. Free and open source.

---

## 3. VEX (Vulnerability Exploitability eXchange) Support

| Format | Authoring Complexity | Trivy Integration | CI/CD Pipeline Gates | Central Tracking |
| :--- | :---: | :---: | :---: | :---: |
| **OpenVEX** | ✅ Simple (one CLI command) | ✅ Native `--vex` | ✅ Best | ⚠️ Lightweight |
| **CycloneDX VEX** | ⚠️ Medium (BOM schema) | ✅ Supported | ⚠️ Adequate | ✅ Best (Audit UI) |
| **CSAF 2.0** | ❌ Complex advisory model | ⚠️ Supported | ❌ Too heavy | ⚠️ Overkill |

> ** Recommendation: OpenVEX (for CI/CD gates)**  
> Manages false positives (e.g., unreachable code) via simple, version-controlled `.vex.json` files in Git under `.security/`. Trivy parses this during scan time to prevent pipeline build breaks on non-exploitable CVEs.

---

## 4. Azure DevOps Integration & Security Gates

### Recommended Severity Gates

| Environment | CRITICAL | HIGH | MEDIUM / LOW | Action on Failure |
| :--- | :---: | :---: | :---: | :--- |
| **Development** | ⚠️ Warn | ⚠️ Warn | ✅ Pass | Non-blocking alerts for visibility |
| **QA / Staging** | ❌ **Block** | ⚠️ Warn | ✅ Pass | Catches critical flaws prior to release |
| **Production** | ❌ **Block** | ❌ **Block** | ✅ Pass | Strict gate (unblockable only with approved VEX) |

### Pipeline Workflow

1. **Build:** Build container image or application artifact.
2. **Generate:** Syft creates CycloneDX + SPDX SBOMs (`syft <image> -o cyclonedx-json=... -o spdx-json=...`).
3. **Scan & Gate:** Trivy scans the SBOM using OpenVEX suppression:  
   `trivy sbom --vex .security/vex.json --severity HIGH,CRITICAL --exit-code 1`
4. **Publish:** SARIF report published to Azure DevOps Security tab; SBOM published as a build artifact or attached to ACR.

---

## 5. Centralized Reporting & Management Platforms

| Platform | SBOM Ingestion | Daily Re-scan (Zero-Days) | VEX Audit UI | ADO + GitHub | Cost |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **OWASP Dependency-Track** | ✅ Native (CycloneDX) | ✅ Automatic | ✅ Full triage UI | ✅ REST API | ✅ Free (Open Source) |
| **MS Defender for Cloud** | ⚠️ Indirect | ✅ Automatic | ❌ No VEX | ✅ Native Connectors | ⚠️ Azure license |
| **Anchore Enterprise** | ✅ Full | ✅ Automatic | ✅ Full UI | ✅ Extensions | ❌ Commercial |

> ** Recommendation: OWASP Dependency-Track (+ Defender for Cloud as overlay)**  
> Dependency-Track is purpose-built for CycloneDX SBOMs and continuously scans past deployments against new zero-day CVEs without pipeline re-runs. Defender for Cloud provides high-level organizational posture across ADO and GitHub.

---

## Summary of Recommended Toolchain

* **SBOM Standard:** CycloneDX 1.5/1.6 (JSON)
* **Generation Tool:** Syft (dual-format output)
* **Scanning & Gates:** Trivy (with `--exit-code 1` on HIGH/CRITICAL)
* **VEX Implementation:** OpenVEX (`vexctl` managed under `.security/`)
* **Central Dashboard:** OWASP Dependency-Track
* **Migration-Ready:** 100% runner-agnostic (identical commands run in Azure DevOps and GitHub Actions)

Let me know your thoughts on this toolchain, and we can set up a quick pilot pipeline to validate the flow.

— Dipak