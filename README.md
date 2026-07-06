[![Darreon Phillips Homepage](https://img.shields.io/badge/Darreon%20Phillips-Homepage-blue?style=for-the-badge&logo=github&logoColor=white)](https://github.com/DaPhilll)

# Security Operations Homelab: Build Guide

A high-level guide for building a security operations homelab similar to the environment behind my portfolio projects. This document covers the overall structure, the tools involved, and where to find current setup instructions for each. It intentionally does not reproduce full command sequences, because installation steps and package versions change often. Each section links to the official documentation so you follow the current process rather than a snapshot that ages out.

## How to Use This Guide
Work through the phases in order. Each phase lists its purpose, the tool it uses, whether that tool is open source or a commercial product, and a link to the current official documentation. The related portfolio repository is referenced where one exists, so you can see the specific artifacts (scripts, configs, rules) that came out of each phase.

A note on tooling: most of this lab runs on free and open source software. A few components reference commercial platforms (Microsoft Sentinel, Rapid7 InsightIDR). Those were used in a professional enterprise environment, not in this homelab. They are included here only to document useful work, commands, and processes. Where a commercial tool has a viable trial, that is noted so you can approximate the workflow without a paid license.

## Reference Hardware and Resource Requirements
This lab is not designed to run continuously or with all virtual machines active at once. Most workflows use a subset of the machines. The requirements below reflect intermittent, subset-based use with headroom for the host OS and basic tasks such as web browsing.

**Virtual machine RAM allocations:**

| VM | Role | RAM |
| :--- | :--- | :--- |
| SRV-DC01 | Windows Server domain controller | 4 GB |
| WKSTN-01 | Windows 10/11 endpoint | 4 GB |
| WIN7-LEGACY01 | Legacy scan target | 2 GB |
| SRV-SOC01 | Ubuntu (Wazuh, Shuffle, OpenVAS) | 8 GB |
| SURICATA-01 | Ubuntu (Suricata sensor) | 4 GB |

Running all five machines at once requires 22 GB of VM RAM. In practice, a working subset of three to four machines covers most tasks.

**Recommended host specifications:**
- **CPU:** 8 cores / 16 threads minimum. Suricata packet processing and OpenVAS scans both benefit from additional cores.
- **RAM:** 32 GB comfortably runs a working subset plus the host OS. 64 GB allows all five machines to run at once and leaves room to split the SOC stack across separate VMs later.
- **Storage:** A 1 TB SSD (NVMe preferred). Budget roughly 350 to 400 GB for the base VMs, with additional space for Wazuh indices, OpenVAS feed data, and VM snapshots.

**Reference build (host used for this lab):**
- **CPU:** AMD Ryzen 9 7900X (12 cores / 24 threads)
- **RAM:** 64 GB DDR5

This configuration runs all five virtual machines simultaneously with headroom for the host OS and standard tasks. The 12-core CPU covers concurrent Suricata monitoring and OpenVAS scanning without contention, and 64 GB of RAM supports running the full lab at once or separating the SOC stack (Wazuh, Shuffle, OpenVAS) onto dedicated virtual machines.

---

## Phase 1: Virtualization and Network Foundation

**Purpose:** Provide the isolated virtual network all other components run on.

**Tool:** VMware Workstation Pro (commercial, now free for personal use). VirtualBox is a fully open source alternative.

**Steps:**
1. Install the hypervisor.
2. Create an internal or host-only network segment (this lab uses `10.10.0.0/24`).
3. Provision the base virtual machines: a Windows Server domain controller, one or more Windows client endpoints, a legacy Windows target for vulnerability testing, and two Ubuntu Server hosts (one for the security stack, one for network monitoring).

**Documentation:**
- VMware Workstation Pro: https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro.html
- VirtualBox: https://www.virtualbox.org/wiki/Documentation
- Ubuntu Server: https://ubuntu.com/server/docs
- Windows Server evaluation (180-day free): https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server

---

## Phase 2: SIEM and Endpoint Telemetry

**Purpose:** Aggregate endpoint telemetry into a central platform and build detection rules.

**Tool:** Wazuh (open source).

**Steps:**
1. Install the Wazuh central components (server, indexer, dashboard) on the security stack host using the quickstart installation assistant.
2. Deploy the Wazuh agent to the Windows endpoints and point them at the manager.
3. Confirm agents report in, then tune noisy default rules with custom local rules.

**Documentation:** https://documentation.wazuh.com/current/quickstart.html

**Related repository:** [Wazuh-Deployment](https://github.com/DaPhilll/Wazuh-Deployment)

---

## Phase 3: Network Intrusion Detection

**Purpose:** Add packet-level visibility into traffic on the lab network.

**Tool:** Suricata (open source).

**Steps:**
1. Install Suricata on the dedicated monitoring host.
2. Enable promiscuous mode on the capture interface at both the hypervisor and OS level.
3. Configure the home network variable, pull the Emerging Threats open ruleset, and add custom signatures.

**Documentation:** https://docs.suricata.io/en/latest/

**Related repository:** [Suricata-Network-Intrusion-Detection-System](https://github.com/DaPhilll/Suricata-Network-Intrusion-Detection-System)

---

## Phase 4: Vulnerability Assessment

**Purpose:** Run authenticated vulnerability scans against lab hosts.

**Tool:** Greenbone Vulnerability Manager / OpenVAS (open source).

**Steps:**
1. Deploy the Greenbone Community Edition container stack on the security host using Docker Compose.
2. Prepare the legacy Windows target for credentialed scanning (enable SMB, Remote Registry, and the local account token policy).
3. Create a scan target and run a Full and Fast scan, then document any accepted risk with overrides.

**Documentation:** https://greenbone.github.io/docs/latest/

**Related repositories:** [Vulnerability-Scanning-and-Reporting-with-OpenVAS](https://github.com/DaPhilll/Vulnerability-Scanning-and-Reporting-with-OpenVAS), [Enterprise-Governance-Risk-and-Compliance-GRC-Vulnerability-Management-Framework](https://github.com/DaPhilll/Enterprise-Governance-Risk-and-Compliance-GRC-Vulnerability-Management-Framework)

---

## Phase 5: Governance, Risk, and Compliance

**Purpose:** Wrap the vulnerability scanning in a risk-based process with compliance mapping.

**Tool:** Process and documentation driven; no dedicated platform required. GLPI (open source) is one option for ITSM ticketing if you want to add it.

**Steps:**
1. Define severity tiers and remediation SLA windows.
2. Build an asset classification and tagging scheme.
3. Map controls to NIST SP 800-53 and SOC 2 Type II.
4. Establish a risk-exception process for findings that cannot be immediately remediated.

**Documentation:**
- NIST SP 800-53: https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final
- GLPI: https://glpi-project.org/documentation/

**Related repository:** [Enterprise-Governance-Risk-and-Compliance-GRC-Vulnerability-Management-Framework](https://github.com/DaPhilll/Enterprise-Governance-Risk-and-Compliance-GRC-Vulnerability-Management-Framework)

---

## Phase 6: Security Orchestration and Automation (SOAR)

**Purpose:** Automate incident triage and response actions across the tools already deployed.

**Tools:** Shuffle (open source SOAR). VirusTotal and ANY.RUN offer free API tiers for enrichment.

**Steps:**
1. Deploy Shuffle via Docker Compose on the security host.
2. Store API credentials in Shuffle's encrypted vault rather than in workflow nodes.
3. Build a workflow that ingests an alert, enriches indicators through VirusTotal, detonates flagged items in ANY.RUN, and isolates the endpoint through the Wazuh API.

**Documentation:**
- Shuffle: https://shuffler.io/docs/about
- VirusTotal API (free tier available): https://docs.virustotal.com/docs/api-overview
- ANY.RUN: https://any.run/

**Related repository:** [SOAR-Playbook-Engineering-Incident-Response-Automation](https://github.com/DaPhilll/SOAR-Playbook-Engineering-Incident-Response-Automation)

---

## Phase 7: Generative AI Incident Triage

**Purpose:** Use a large language model to draft structured incident reports from raw logs.

**Tool:** Google Gemini API (commercial, with a free tier).

**Steps:**
1. Install the official Google GenAI SDK for Python.
2. Set an API key as an environment variable.
3. Build a pipeline that parses log output, sends it to the model with a constrained system prompt, and writes a structured report for analyst review.

**Documentation:** https://ai.google.dev/gemini-api/docs

**Related repository:** [Generative-AI-Incident-Triage-Investigative-Reporting](https://github.com/DaPhilll/Generative-AI-Incident-Triage-Investigative-Reporting)

---

## Phase 8: Detection-as-Code

**Purpose:** Manage detection logic as version-controlled queries mapped to MITRE ATT&CK.

**Tools:** Microsoft Sentinel and Rapid7 InsightIDR (both commercial). These are enterprise platforms I used in a professional environment, not in this homelab. The related repository stores the query logic, tuning patterns, and ATT&CK mapping as reference work.

**Trial access:** Microsoft Sentinel can be explored through an Azure free account and a Log Analytics workspace. Rapid7 InsightIDR does not offer a broadly available self-service trial, so its queries in the related repository are reference material rather than something a homelab can easily reproduce.

**Steps (conceptual, for a Sentinel trial):**
1. Create an Azure free account and a Log Analytics workspace.
2. Enable Microsoft Sentinel on the workspace.
3. Connect a data source (for example, Azure Activity or Entra ID sign-in logs).
4. Author analytics rules in KQL and store them under version control.

**Documentation:**
- Microsoft Sentinel: https://learn.microsoft.com/en-us/azure/sentinel/
- Azure free account: https://azure.microsoft.com/en-us/free/
- Rapid7 InsightIDR: https://docs.rapid7.com/insightidr/

**Related repository:** [Enterprise-Detection-as-Code-Log-Correlation-Suite](https://github.com/DaPhilll/Enterprise-Detection-as-Code-Log-Correlation-Suite)

---

## Phase 9: Hybrid Cloud Identity

**Purpose:** Extend the on-premises domain controller into a cloud identity tenant and apply risk-based access controls.

**Tool:** Microsoft Entra ID (commercial, available through the Microsoft 365 Developer Program).

**Trial access:** The Microsoft 365 Developer Program provides a renewable E5 sandbox that includes Entra ID P2. Note that eligibility tightened in 2024: a personal Microsoft account is generally no longer sufficient on its own, and access typically requires a Visual Studio Enterprise or Professional subscription or qualifying program membership. Confirm current eligibility on the program page before planning around it.

**Steps:**
1. Obtain a Microsoft 365 Developer Program tenant (E5 sandbox with Entra ID P2).
2. Install the Entra provisioning agent on a domain-joined host and configure Entra Cloud Sync.
3. Scope the initial sync to a pilot organizational unit.
4. Build Conditional Access policies (MFA for admin roles, block legacy authentication) and risk-based policies in report-only mode first.

**Documentation:**
- Microsoft 365 Developer Program: https://developer.microsoft.com/en-us/microsoft-365/dev-program
- Entra Cloud Sync: https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync
- Conditional Access: https://learn.microsoft.com/en-us/entra/identity/conditional-access/

**Related repository:** [Hybrid-Identity-Bridge-Entra-Cloud-Sync](https://github.com/DaPhilll/Hybrid-Identity-Bridge-Entra-Cloud-Sync)

---

## Phase 10: Host Hardening (Ongoing)

**Purpose:** Audit the Linux hosts in the lab against a recognized benchmark.

**Tool:** Bash and the CIS Ubuntu Linux Benchmark (the benchmark PDF is free after registration with the Center for Internet Security).

**Steps:**
1. Run an audit script against each Ubuntu host to check SSH, firewall, file permission, and audit logging settings.
2. Remediate failed checks.
3. Re-run to confirm the score improves.

**Documentation:** https://www.cisecurity.org/cis-benchmarks

**Related repository:** [Linux-Security-Hardening-Compliance-Audit](https://github.com/DaPhilll/Linux-Security-Hardening-Compliance-Audit)

---

## Tool Summary

| Phase | Tool | Type |
| :--- | :--- | :--- |
| Virtualization | VMware Workstation Pro / VirtualBox | Free (personal) / Open source |
| SIEM | Wazuh | Open source |
| Network IDS | Suricata | Open source |
| Vulnerability Scanning | Greenbone / OpenVAS | Open source |
| GRC / ITSM | Process-driven / GLPI | Open source |
| SOAR | Shuffle | Open source |
| Enrichment | VirusTotal, ANY.RUN | Commercial (free tier) |
| AI Triage | Google Gemini API | Commercial (free tier) |
| Detection-as-Code | Microsoft Sentinel, Rapid7 InsightIDR | Commercial (enterprise, used professionally) |
| Cloud Identity | Microsoft Entra ID | Commercial (Developer Program) |
| Host Hardening | Bash / CIS Benchmark | Open source / free registration |

## License
MIT — see [LICENSE](./LICENSE).

<br><br><br>
[![Darreon Phillips Homepage](https://img.shields.io/badge/Darreon%20Phillips-Homepage-blue?style=for-the-badge&logo=github&logoColor=white)](https://github.com/DaPhilll)
