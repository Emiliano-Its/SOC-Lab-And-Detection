# Incident Response Report — IR-001

## Summary

| Field | Value |
|---|---|
| ATT&CK Technique | T1003 — OS Credential Dumping (T1003.001, LSASS Memory) |
| Tactic | Credential Access |
| Affected host | `soc-lab-windows` (Windows 11 Pro endpoint) |
| Detection source | Sysmon Event ID 1 (process creation) via Wazuh agent |
| Triggering rule | `100102` — "T1003: Known credential dumping tool executed" (level 14) |
| Alert timestamp | 2026-07-01 21:59:41 UTC |
| Analyst | Jose Emiliano Cortez Rivera (Tier 1/2 SOC Analyst) |
| Severity | High |
| Status | Closed — confirmed true positive (controlled test) |

---

## 1. Alert

At **21:59:41 UTC on 2026-07-01**, Wazuh rule `100102` fired on `soc-lab-windows`:

> T1003: Known credential dumping tool executed

The rule matched a Sysmon Event ID 1 process-creation record where `win.eventdata.image`
matched `gsecdump.exe`. The same second produced three correlated alerts on the same
host, giving a full picture of the process chain:

| Time (UTC) | Rule ID | Level | Description |
|---|---|---|---|
| 21:59:40.578 | 100103 | 10 | T1016: Built-in network discovery tool spawned by PowerShell |
| 21:59:40.596 | 92052 | 4 | Windows command prompt started by an abnormal process |
| 21:59:40.611 | 100102 | 14 | T1003: Known credential dumping tool executed |
| 21:59:40.617 | 92213 | 15 | Executable file dropped in folder commonly used by malware |

The clustering of a discovery alert, an abnormal `cmd.exe` parent, a known-credential-dumper
execution, and a malware-path file drop within a single second is consistent with an
automated post-exploitation tool chain rather than a manual, interactive session.

## 2. Triage

1. **Confirm the process chain.** Pulled the Sysmon Event ID 1 record for the `gsecdump.exe`
   launch and walked the parent chain: `powershell.exe → cmd.exe → gsecdump.exe`. The
   `cmd.exe` parent was flagged by rule `92052` as an abnormal parent for a command shell,
   consistent with a script (not a user) driving the shell.
2. **Confirm the binary is not a known-good tool.** `gsecdump.exe` has no legitimate business
   use on this endpoint — it is not part of the approved software baseline, ruling out the
   "sysadmin debugging tool" false-positive case noted in the Sigma rule.
3. **Check for LSASS access.** Rule `100102` fires on process name only; it does not confirm
   memory access to `lsass.exe`. Sysmon Event ID 10 (ProcessAccess) is **not currently
   covered by any local rule**, so this step could not be automated — flagged as a detection
   gap (see Lessons Learned).
4. **Check for the dropped artifact.** Rule `92213` fired in the same second for an
   executable dropped in a folder commonly abused by malware, indicating the tool was staged
   to disk immediately before execution rather than run from a pre-existing location.
5. **Check for follow-on activity.** Reviewed the 60 seconds following the alert for outbound
   network connections from `powershell.exe` (rule `100105`, C2 port list) or further
   credential-dumping executions — none were observed in this window, so credential access
   was not immediately followed by confirmed exfiltration.

## 3. Indicators of Compromise (IOCs)

| Type | Value |
|---|---|
| Process name | `gsecdump.exe` |
| Parent process chain | `powershell.exe` → `cmd.exe` → `gsecdump.exe` |
| Host | `soc-lab-windows` |
| Wazuh rule IDs | 100102 (credential dumping), 92213 (malware-path file drop), 92052 (abnormal cmd parent), 100103 (network discovery) |
| Behavior pattern | Discovery → file drop → credential-dumping tool execution within 1 second |

## 4. Scope of Impact

- **Blast radius:** single endpoint (`soc-lab-windows`). No evidence of lateral movement,
  remote authentication, or credential reuse elsewhere in the environment was found in the
  telemetry reviewed.
- **Data at risk:** any credentials cached in memory or the local SAM database on this host
  at the time of execution — including the interactively logged-on user's session secrets.
  Since this environment has no domain controller, domain credential exposure is not
  applicable; in a domain-joined environment this same technique could expose cached domain
  credentials and enable lateral movement.
- **Confirmed compromise vs. attempt:** the tool executed successfully (process creation
  confirmed); whether it successfully extracted usable credentials from LSASS could not be
  confirmed or ruled out with current telemetry (see detection gap below).

## 5. Containment, Eradication & Recovery

**Containment**
- Isolated `soc-lab-windows` from the rest of the lab network segment to prevent any
  extracted credentials from being used for lateral movement.
- Killed the `gsecdump.exe` process and its parent `cmd.exe`/`powershell.exe` chain.

**Eradication**
- Removed the dropped executable and its staging folder (flagged by rule `92213`) from disk.
- Reviewed scheduled tasks, registry Run keys, and startup folders on the host for
  persistence artifacts tied to the same process chain — none found.

**Recovery**
- Rotated credentials for any account with an active session on the host at the time of the
  alert (local Administrator and the interactive test user).
- Re-enabled network connectivity for the host only after confirming a clean state via a
  full Sysmon-logged reboot with no recurrence of rules `100101`–`100103`.
- Re-baselined the host against the Sysmon/Wazuh alert history to confirm no other
  outstanding alerts.

## 6. Lessons Learned

- **Detection gap — LSASS memory access.** Rule `100102` only matches on known tool
  filenames (`gsecdump.exe`, `mimikatz.exe`, `procdump.exe`, etc.), which is trivially
  bypassed by renaming the binary. A behavioral rule on Sysmon Event ID 10
  (`ProcessAccess`, target image `lsass.exe`, from a non-system-signed source process) would
  catch renamed or custom tooling and should be added as a compensating control.
- **Correlation value confirmed.** Rules `100102`, `92213`, `92052`, and `100103` firing
  together within one second turned a single medium-confidence alert into a
  high-confidence incident — this correlation pattern (discovery → staging → execution)
  is worth encoding as an explicit composite/correlation rule rather than relying on an
  analyst noticing the cluster manually.
- **Response time.** Because the rule fires on process creation (not on completion of the
  dump), triage and containment can begin before the tool finishes running — this should be
  reflected in on-call runbooks as a "kill first, investigate after" alert class.
