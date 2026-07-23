# Incident Response Report — IR-002

## Summary

| Field | Value |
|---|---|
| ATT&CK Technique | T1046 — Network Service Discovery |
| Tactic | Discovery |
| Affected host | `soc-lab-UbuntuServer` (10.0.0.5) — target of the scan |
| Source | `10.0.0.4` — internal network segment |
| Detection source | Zeek `conn.log` (network telemetry) via Wazuh agent on `soc-lab-UbuntuServer` |
| Triggering rules | `100903` (level 7, per-connection) → `100904` (level 10, aggregate) |
| Alert timestamp | 2026-06-27 07:17:31 UTC |
| Analyst | Jose Emiliano Cortez Rivera (Tier 1/2 SOC Analyst) |
| Severity | Medium |
| Status | Closed — confirmed true positive (controlled test) |

---

## 1. Alert

At **07:17:31.3 UTC on 2026-06-27**, Wazuh rule `100904` fired on `soc-lab-UbuntuServer`:

> T1046: Zeek - Multiple rejected connections from 10.0.0.4 (5+ in 20s - possible port scan activity)

This is a frequency rule (`if_matched_sid`, 5 hits / 20s) sitting on top of the base rule
`100903`, which logs each individual rejected connection. In the 20-second window leading
up to the aggregate alert, at least 12 individual `100903` events fired, each for a
different destination port on `10.0.0.5`:

```
10.0.0.4 → 10.0.0.5:3986   rejected
10.0.0.4 → 10.0.0.5:26     rejected
10.0.0.4 → 10.0.0.5:6001   rejected
10.0.0.4 → 10.0.0.5:1086   rejected
10.0.0.4 → 10.0.0.5:1070   rejected
10.0.0.4 → 10.0.0.5:79     rejected
10.0.0.4 → 10.0.0.5:8088   rejected
10.0.0.4 → 10.0.0.5:1001   rejected
10.0.0.4 → 10.0.0.5:1067   rejected
10.0.0.4 → 10.0.0.5:1309   rejected
10.0.0.4 → 10.0.0.5:4111   rejected
10.0.0.4 → 10.0.0.5:1984   rejected
```

Unlike the host-based detections in this project, this alert is driven entirely by network
telemetry (Zeek) — it does not depend on any process or file event on the target host, so it
would catch a scan even if the target had no endpoint agent at all.

## 2. Triage

1. **Pull the raw Zeek `conn.log` entries** for `id.orig_h == 10.0.0.4` in the alert window.
   All flagged connections carried `connection_state: REJ` (Wazuh's decoded name for Zeek's
   `conn_state` field — see repo note on the decoder field-renaming gotcha), meaning the
   destination actively refused the connection (TCP RST) rather than timing out.
2. **Check the destination port list for a pattern.** Ports are non-sequential and span a
   wide range (26, 79, 1001–8088) — consistent with a randomized port list rather than a
   simple sequential sweep, and consistent with the specific Atomic Red Team test used
   (`Invoke-AtomicTest T1046 -TestNumbers 3`).
3. **Check for any successful connection.** No `conn_state: SF` (successful handshake)
   events were present for `10.0.0.4` in this window — the scan did not find an open port on
   `soc-lab-UbuntuServer` during this pass.
4. **Attribute the source.** `10.0.0.4` is the designated internal scanner/attacker host for
   this environment. In a production setting this step would instead involve checking DHCP
   leases, asset inventory, and whether `10.0.0.4` has any legitimate scanning function
   (e.g., an authorized vulnerability scanner), per the rule's documented false-positive
   cases.

## 3. Indicators of Compromise (IOCs)

| Type | Value |
|---|---|
| Source IP | `10.0.0.4` |
| Target IP | `10.0.0.5` (`soc-lab-UbuntuServer`) |
| Connection state | `REJ` (rejected) |
| Wazuh rule IDs | 100903 (individual reject), 100904 (aggregate: 5+ in 20s) |
| Pattern | ≥12 distinct destination ports probed from one source within 20 seconds |

## 4. Scope of Impact

- **Blast radius:** reconnaissance only. No connection reached `conn_state: SF`, so no
  service on `soc-lab-UbuntuServer` was actually reached or exploited during this scan.
- **Information exposed to the attacker:** effectively none — a scan returning only `REJ`
  tells the scanning party that the host is up and actively refusing connections on the
  probed ports, but reveals no open service to target next.
- **Tactic context:** Discovery-stage activity typically precedes exploitation attempts
  against any port found open; because none were found open here, this scan alone does not
  indicate a foothold was gained.

## 5. Containment, Eradication & Recovery

**Containment**
- Because `10.0.0.4` is inside the same trust boundary, containment in this lab means
  restricting it at the host firewall / NSG level rather than a perimeter block. In a
  production network, the equivalent step would be blocking the source IP at the nearest
  enforcement point (host firewall, segment ACL, or perimeter firewall if external).

**Eradication**
- No artifact was left behind on `soc-lab-UbuntuServer` — a rejected-connection scan leaves
  nothing to remove from the target host itself.

**Recovery**
- No recovery action needed on `soc-lab-UbuntuServer` since no service was reached.
- Verified Zeek continued shipping `conn.log` normally after the event (confirming the
  sensor itself wasn't a target/casualty of the scan).

## 6. Lessons Learned

- **Detection window is tunable, and that's a tradeoff.** Rule `100904` requires 5+ rejected
  connections within 20 seconds. A slower scan (e.g., one port every 5 seconds) would stay
  under this threshold and evade the aggregate rule entirely, while still tripping individual
  `100903` alerts that an analyst would have to notice manually. A secondary, longer-window
  aggregate rule (e.g., 10+ in 5 minutes) would catch low-and-slow scans without raising the
  false-positive rate of the fast-window rule.
- **Network telemetry closes a blind spot host telemetry can't.** This is the only detection
  in the current rule set that doesn't depend on Sysmon/auditd on the target — it would have
  caught the same activity even against an unmanaged or unmonitored host, which is a strong
  argument for keeping Zeek coverage on every network segment, not just the ones with
  endpoint agents installed.
- **Single point of failure.** This detection depends entirely on the Zeek sensor on
  `soc-lab-UbuntuServer` continuing to ship logs and the manager-side decoder continuing to
  rename `conn_state` → `connection_state` correctly (a real regression was caught here
  previously — see `wazuh/README.md` notes). A health check / heartbeat alert for "Zeek logs
  stopped arriving" would prevent this detection from silently going dark.
