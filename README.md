# Home SOC Lab & Detection Engineering

> Practical capstone project for SOC Analyst (Tier 1/2) — Detection Engineering

---

## Project Overview

This project simulates building a minimum viable SOC from scratch for a small fintech startup with no existing monitoring infrastructure. The lab covers the full detection engineering lifecycle: deploy, ingest, simulate, detect, and respond.

---

## Lab Architecture

| VM | Role | Size | OS |
|---|---|---|---|
| `soc-lab-windows` | Victim endpoint | Standard D2s v3 | Windows 11 Pro |
| `soc-lab-UbuntuServer` | Linux server + Zeek | Standard D2s v3 | Ubuntu Server 24.04 LTS |
| `soc-lab-wazuh` | SIEM | Standard D4s v3 | Ubuntu Server 24.04 LTS |

All VMs are deployed on **Microsoft Azure** (East US region) inside an isolated Virtual Network (`soc-lab-vnet`, `10.0.0.0/16`).

---

## Technology Stack

### Infrastructure
- **Cloud:** Microsoft Azure (Azure for Students)
- **Virtualization:** Azure Virtual Machines
- **Network:** Azure Virtual Network (`soc-lab-vnet`, isolated)

### Telemetry
- **Sysmon** (SwiftOnSecurity config) — deep Windows endpoint visibility
- **auditd** (Neo23x0 ruleset) — Linux command and file activity
- **Zeek 8.2.0** — network traffic analysis (JSON output)

### SIEM
- **Wazuh 4.14.5** — log ingestion, alerting, MITRE ATT&CK mapping
- **Wazuh Agent** — installed on Windows and Ubuntu VMs

### Attack Simulation
- **Atomic Red Team** (Invoke-AtomicTest) — controlled attack technique execution

### Detection
- **Sigma** (YAML) — universal detection rule format
- **pySigma** — converts Sigma rules to Wazuh native format

### Version Control
- **GitHub** — stores all rules, reports, diagrams, and documentation

---

## How Logs Flow

```
Windows VM (Sysmon + Wazuh Agent)
        │
        ├──────────────────────────┐
        │                         ▼
Ubuntu VM (auditd + Zeek          Wazuh SIEM
         + Wazuh Agent) ─────────► (alerts + dashboards)
```

---

## Project Phases

### Phase 1 — Lab Build & Telemetry *(current)*
- [x] Azure resource group created (`soc-lab`)
- [x] Isolated virtual network deployed (`soc-lab-vnet`)
- [x] Windows 11 VM deployed (`soc-lab-windows`)
- [x] Ubuntu Server VM deployed (`soc-lab-UbuntuServer`)
- [x] Wazuh SIEM VM deployed (`soc-lab-wazuh`)
- [x] Wazuh 4.14.5 installed and dashboard accessible
- [x] Wazuh agent installed on Windows (`windows-endpoint`)
- [x] Wazuh agent installed on Ubuntu (`ubuntu-server`)
- [x] Sysmon installed on Windows (SwiftOnSecurity config)
- [x] Sysmon events flowing into Wazuh (verified via Threat Hunting)
- [x] auditd configured on Ubuntu (Neo23x0 ruleset)
- [x] auditd events flowing into Wazuh
- [x] Zeek 8.2.0 installed and capturing network traffic
- [ ] Zeek JSON logs shipping to Wazuh
- [ ] Verify all logs searchable in Wazuh

### Phase 2 — Attack Simulation
- [ ] ≥ 6 ATT&CK techniques selected (≥ 3 tactics)
- [ ] Atomic Red Team installed on Windows VM
- [ ] Each technique executed and logged
- [ ] Evidence captured per technique

### Phase 3 — Detection Engineering
- [ ] ≥ 6 Sigma rules written (one per technique)
- [ ] Each rule tested and alert screenshot captured
- [ ] At least 1 rule tuned (before/after documented)
- [ ] At least 1 correlation rule across 2 data sources
- [ ] pySigma conversion to Wazuh format

### Phase 4 — Incident Response Writeups
- [ ] 3 IR reports written
- [ ] 1 report written for non-technical leadership

---

## Repository Structure

```
/
├── README.md
├── LESSONS_LEARNED.md
├── /architecture/
│   └── lab-topology.png
├── /rules/
│   └── *.yml (Sigma rules)
├── /attacks/
│   └── *.md (one per ATT&CK technique)
├── /ir-reports/
│   └── *.md (3 incident response reports)
└── /tuning/
    └── *.md (before/after rule tuning)
```

---

## How to Reproduce the Lab

> Full setup guide coming as phases are completed.

### Prerequisites
- Microsoft Azure account (student or PAYG)
- Basic Linux CLI knowledge
- Git installed
- SSH key pair generated

### Quick Start
```bash
# Clone this repo
git clone https://github.com/Emiliano-Its/SOC-Lab-And-Detection.git
cd SOC-Lab-And-Detection

# Deploy lab (Terraform template coming soon)
# For now, follow manual steps in /architecture/
```

### Key Lessons Learned
- Wazuh agent requires read permissions on log files — add `wazuh` user to `adm` group for audit.log access
- Zeek must be configured to output JSON format for Wazuh compatibility
- Azure NSG must have port 443 open for Wazuh dashboard external access
- Always stop VMs from Azure Portal (not from inside the OS) to get "Stopped (deallocated)" state

---

## MITRE ATT&CK Coverage

> To be updated as attack simulations are completed.

| Technique ID | Name | Tactic | Detected |
|---|---|---|---|
| TBD | TBD | TBD | ⏳ |

---

## Resources

- [MITRE ATT&CK](https://attack.mitre.org)
- [Sigma HQ](https://github.com/SigmaHQ/sigma)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Neo23x0 auditd Rules](https://github.com/Neo23x0/auditd)
- [Zeek Documentation](https://docs.zeek.org)
- [Wazuh Documentation](https://documentation.wazuh.com)
- [The DFIR Report](https://thedfirreport.com)

---

## Author

Jose Emiliano Cortez Rivera  
Cybersecurity Student — Universidad Autónoma de Nuevo León  
GitHub: [Emiliano-Its](https://github.com/Emiliano-Its)
