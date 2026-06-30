# Home SOC Lab & Detection Engineering

A hands-on detection engineering lab built from scratch on Microsoft Azure. The goal is to simulate realistic attacker behavior across a multi-host environment, build custom detections in Sigma, and produce incident response documentation — covering the full blue team lifecycle from telemetry ingestion to alert triage.

---

## Architecture

Three isolated VMs deployed on Azure inside a private virtual network (`soc-lab-vnet`, `10.0.0.0/16`):

| Host | Role | OS |
|---|---|---|
| `soc-lab-windows` | Victim endpoint | Windows 11 Pro |
| `soc-lab-UbuntuServer` | Linux server + network sensor | Ubuntu Server 24.04 LTS |
| `soc-lab-wazuh` | SIEM | Ubuntu Server 24.04 LTS |

```
Windows (Sysmon + Wazuh Agent)
        │
        ├─────────────────────────────┐
        │                             ▼
Ubuntu (auditd + Zeek + Wazuh Agent)──► Wazuh SIEM
```

---

## Stack

| Layer | Tool |
|---|---|
| SIEM | Wazuh 4.14.5 |
| Endpoint telemetry (Windows) | Sysmon — SwiftOnSecurity config |
| Endpoint telemetry (Linux) | auditd — Neo23x0 ruleset |
| Network telemetry | Zeek 8.2.0 (JSON output) |
| Attack simulation | Atomic Red Team (Invoke-AtomicTest) |
| Detection format | Sigma (YAML) |
| Rule conversion | pySigma |

---

## ATT&CK Coverage

| Technique | Name | Tactic | Sigma Rule |
|---|---|---|---|
| T1003 | OS Credential Dumping | Credential Access | ✓ |
| T1016 | System Network Configuration Discovery | Discovery | ✓ |
| T1041 | Exfiltration Over C2 Channel | Exfiltration | ✓ |
| T1046 | Network Service Discovery | Discovery | ✓ |
| T1057 | Process Discovery | Discovery | ✓ |
| T1059.001 | PowerShell | Execution | ✓ |
| T1059.001 | PowerShell Encoded Commands | Execution | ✓ |

---

## Repository Structure

```
/
├── README.md
├── LESSONS_LEARNED.md
├── /architecture/
│   └── lab-topology.png
├── /rules/
│   └── *.yml              — one Sigma rule per technique
├── /attacks/
│   └── *.md               — one doc per simulated technique
├── /ir-reports/
│   └── *.md               — incident response writeups
└── /tuning/
    └── *.md               — before/after rule tuning documentation
```

---

## Detection Engineering Approach

Each technique follows the same workflow:

1. **Simulate** — execute the technique using Atomic Red Team
2. **Observe** — identify what telemetry Wazuh captured and from which source
3. **Write** — author a Sigma rule targeting the specific behavior, not the generic alert
4. **Tune** — introduce a benign edge case, refine the rule logic until false positives disappear
5. **Document** — produce an IR writeup treating the detection as a real incident

---

## Key Technical Notes

- Wazuh agent requires read access to log files — add `wazuh` user to `adm` group for `audit.log` and `zeek` group for Zeek logs
- Zeek must output JSON format: load `policy/tuning/json-logs` in `local.zeek`
- Zeek log path is `/opt/zeek/spool/zeek/` — not `/opt/zeek/logs/current/`
- Enable `logall_json` in Wazuh to archive all events, not just alerts
- Azure NSG requires port 443 open for external Wazuh dashboard access

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
