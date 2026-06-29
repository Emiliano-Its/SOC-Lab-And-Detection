## ATT&CK ID: T1041

## Technique
Exfiltration Over C2 Channel

## Tactic
Exfiltration

### Command used

```powershell
Invoke-AtomicTest T1041 -TestNumbers 1
```

### Timestamp

Jun 29, 2026 - 09:05 PM

### Expected telemetry

- PowerShell process creation (`powershell.exe`)
- PowerShell spawning a child PowerShell instance
- Executable file dropped in a folder commonly used by malware
- Network connection attempts to an external C2 endpoint
- Wazuh alerts related to suspicious file drops and PowerShell activity
- Sysmon process creation and network connection events

### Screenshot
![WAZUH SCREENSHOT](img/T1041.png)
