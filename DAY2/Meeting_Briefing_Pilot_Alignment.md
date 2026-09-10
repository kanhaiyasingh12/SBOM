# Meeting Briefing: SBOM & VEX Pilot & Implementation Alignment
 
---

## 1. Openings
>I Worked on the Daily QRE Report channel alerts for the Helious Release Orchestrator in the Main and Dev, QA environments in promotion alerts.

>I Added the Fabric Knowledge Graph Discovery Report to the ADO Wiki for the Production environment.

>I Completed the initial tooling evaluation, and the recommendations were added to Sam's Pilot Plan page on the ADO Wiki.

> And Evaluated the tools (Syft, Trivy, OpenVEX, and Dependency-Track) and shared the document to samual. 

>Helped Sam prepare for the pilot by checking all the prerequisites in Azure DevOps and making sure the same setup would also work in GitHub Actions.

>And Tested access in the live Dev environment by verifying access to helios-dev-aks-registry.

>Also confirmed that the build runners can download the required tools and the vulnerability database from GitHub.

---



## 2. The Recommended Toolchain

* **SBOM Format — CycloneDX 1.5/1.6 (JSON):**  
  * *"CycloneDX is security-first with first-class vulnerability mapping and native VEX support. However, by using Syft, we will generate both CycloneDX and SPDX in one single command, giving us a compliance safety net with zero extra cost."*

* **Generation — Syft:**  
  *  *"Syft provides deep container layer inspection in 3–5 seconds, supports multi-language repos, and outputs dual formats from one command."*

* **Scanning — Trivy:**  
  * *"Trivy scans the generated SBOM directly, provides native `--vex` filtering to suppress non-exploitable noise, and outputs SARIF for the ADO/GitHub security tabs."*

* **False Positive Suppression — OpenVEX:**  
  * *"We store `.security/vex.json` in git. It allows developers to mark unreachable or mitigated CVEs as not-affected so pipelines don't get blocked by irrelevant alerts."*

* **Central Dashboard — OWASP Dependency-Track:**  
  * *"Pipelines only scan during build time. Dependency-Track continuously re-scans stored SBOMs every 24 hours against newly discovered zero-days without recompiling code."*

---
