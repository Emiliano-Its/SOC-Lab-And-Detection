# Wazuh Rule Translations

Manual translations of the Sigma rules in [`/rules/`](../rules/) into Wazuh's
native XML rule format.

## Why manual?

There is **no official pySigma backend for Wazuh** (`sigma plugin list` shows
backends for Splunk, Elasticsearch, Sentinel, QRadar, etc. — but not Wazuh).
Wazuh uses a custom XML rule/decoder engine that differs enough from standard
query languages that hand-translation is the accepted approach. Doing it by hand
also documents the field-level mapping explicitly.

## Field mapping (Sigma → Wazuh)

Wazuh decodes Sysmon events and exposes their fields under `win.eventdata.*`:

| Sigma field           | Wazuh field                       |
|-----------------------|-----------------------------------|
| `Image`               | `win.eventdata.image`             |
| `ParentImage`         | `win.eventdata.parentImage`       |
| `CommandLine`         | `win.eventdata.commandLine`       |
| `Initiated`           | `win.eventdata.initiated`         |
| `DestinationPort`     | `win.eventdata.destinationPort`   |
| `DestinationHostname` | `win.eventdata.destinationHostname` |

Sysmon Event ID → Wazuh parent group:

| Sysmon Event | Meaning            | Wazuh group     |
|--------------|--------------------|-----------------|
| 1            | Process Creation   | `sysmon_event1` |
| 3            | Network Connection | `sysmon_event3` |

`endswith` in Sigma is rendered with a `pcre2` regex anchored with `$` and made
case-insensitive with `(?i)`.

## Rule inventory

| Wazuh ID | Sigma rule                  | ATT&CK    | Level | Log source     |
|----------|-----------------------------|-----------|-------|----------------|
| 100001   | PowerShell child spawn      | T1059.001 | 12    | Sysmon EID 1   |
| 100002   | Credential dumping tools    | T1003     | 14    | Sysmon EID 1   |
| 100003   | Network discovery tools     | T1016     | 10    | Sysmon EID 1   |
| 100004   | tasklist process discovery  | T1057     | 10    | Sysmon EID 1   |
| 100005   | PowerShell C2 connection    | T1041     | 14    | Sysmon EID 3   |
| 100010/11| Port scan (correlation)     | T1046     | 12    | Zeek conn.log  |

## Deploy on the Wazuh manager

```bash
sudo cp local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager

# confirm the rules loaded without syntax errors:
sudo tail -n 50 /var/ossec/logs/ossec.log
```

## Known translation caveats

- **T1041 (100005):** Sysmon omits `DestinationHostname` when a connection has no
  resolved DNS name (a raw IP). Wazuh cannot assert the *absence* of a field in a
  single rule, so the "raw IP / no hostname" condition is approximated by the
  remaining strong indicators (PowerShell + outbound + C2 port). Chain a
  suppression rule if you need the exact Sigma logic.

- **T1046 (100010/100011):** This is a correlation rule and depends on Zeek
  `conn.log` being ingested and decoded by Wazuh. The field names (`conn_state`,
  `id.orig_h`) assume a JSON/Zeek decoder is producing them — adjust to match your
  actual Zeek → Wazuh pipeline.
