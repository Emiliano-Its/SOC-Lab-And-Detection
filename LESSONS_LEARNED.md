# Lessons Learned

## What was hard

**Wazuh has no official Sigma backend.** Tools like Splunk, Elasticsearch, and Sentinel have
a pySigma backend that converts a Sigma YAML rule straight into a native query. Wazuh
doesn't. Every rule in `rules/` had to be hand-translated into `wazuh/local_rules.xml` /
`wazuh/zeek_rules.xml`, which meant learning Wazuh's field-naming conventions
(`win.eventdata.image`, `sysmon_event1` groups, `pcre2` field matching) well enough to
translate accurately rather than just copy-pasting a converter's output. It was slower, but
it forced an understanding of the underlying event structure that a one-command conversion
would have let me skip past.

**Decoders silently rewrite field names, and that's easy to get backwards.** The Wazuh
manager runs a custom decoder that renames raw Zeek JSON keys before rules ever see them —
Zeek's `conn_state` becomes Wazuh's `connection_state`, `query` becomes `dnsquery`, and so
on. This isn't visible from the Sigma rule or from Zeek's own docs, only from the decoder
config on the manager. I broke rule `100903` once by "fixing" `connection_state` back to
`conn_state` because that matched the raw Zeek field name and looked more "correct" — it
actually broke the rule, since the decoder never emits a field called `conn_state`. The fix
was confirmed only by checking a live alert's actual field name, not by reading any
documentation. Lesson: for a decoded pipeline, the decoder's output is the source of truth,
not the upstream tool's native schema.

**Real attack tooling doesn't always match my mental model of it.** The T1057 process
discovery rule assumed `tasklist.exe` would be spawned directly by `powershell.exe`. Testing
with `Invoke-AtomicTest` showed Atomic Red Team actually runs it through an intermediary
shell (`powershell.exe → cmd.exe → tasklist.exe`), so the original rule had a false negative
— it simply never fired, silently. This is written up in
[`tuning/T1057-tasklist-parent-broadening.md`](tuning/T1057-tasklist-parent-broadening.md).
The broader lesson: a rule that looks logically correct on paper still has to be proven
against the actual telemetry the real tool produces, not the telemetry I assumed it would
produce.

**Documentation drifts from reality fast, and I have to actively guard against trusting it.**
Both `README.md` and `wazuh/README.md` at points listed rule IDs (`100001`–`100011`) that
didn't match what was actually deployed (`100101`–`100107`, `100900`–`100907`). The
`<rule_dir>`/`<decoder_dir>` auto-load config means "is this rule live" reduces to "does the
file exist in the directory," not "is it registered in some index" — but the README's tables
implied otherwise. Every time I needed to know a real rule ID, I had to read the actual XML,
not the summary table above it.

## What I would do differently

- **Write the tuning writeup at the same time as the tuning fix, not after.** The T1057
  before/after screenshots and the commit that fixed it happened on 2026-07-01; the writeup
  explaining them wasn't produced until this session. Documenting immediately after a fix,
  while the root cause is still fresh, would have been more accurate and faster than
  reconstructing it from git history and screenshots later.
- **Add a behavioral LSASS-access rule (Sysmon Event ID 10) alongside the T1003 filename
  match.** The current credential-dumping rule only matches known tool filenames
  (`gsecdump.exe`, `mimikatz.exe`, etc.), which is trivial to bypass by renaming the binary.
  This gap is called out explicitly in
  [`ir-reports/IR-001-T1003-CredentialDumping.md`](ir-reports/IR-001-T1003-CredentialDumping.md)
  and should have been built alongside the original rule rather than left as a follow-up.
- **Add a longer-window aggregate rule for slow port scans.** The T1046 detection
  (`100903`/`100904`) requires 5+ rejected connections within 20 seconds. A scan spaced out
  below that rate would evade the aggregate rule entirely while still generating individual
  alerts nobody is likely to notice in isolation — see
  [`ir-reports/IR-002-T1046-PortScan.md`](ir-reports/IR-002-T1046-PortScan.md).
- **Scope the project explicitly, in writing, earlier.** I made a deliberate call partway
  through to stop at 6 detection rules (T1003, T1016, T1041, T1046, T1057, T1059.001) rather
  than keep expanding coverage — which means two of the capstone brief's stretch-style
  coverage requirements (≥2 network-telemetry techniques beyond T1046, and a
  cross-source correlation rule combining Sysmon + Zeek) are knowingly left unmet. That was
  the right call for the time I had, but I should have written the scope decision down when
  I made it instead of it living only in conversation history.

## What I want to learn next

- **Cross-source correlation rules.** Every rule here fires on a single data source (Sysmon
  *or* Zeek, never both together). A rule that ties a Sysmon process-creation event to a
  Zeek connection event for the same host within a time window — e.g., confirming that a
  PowerShell process seen dropping a payload (`92213`) is the same process that later
  produced the outbound connection Zeek observed — would catch more of what T1041 was
  designed to detect and close the confirmation gap noted in
  [`ir-reports/IR-003-T1041-Exfiltration-Leadership.md`](ir-reports/IR-003-T1041-Exfiltration-Leadership.md).
- **Threat intel enrichment.** None of the current rules check an indicator against any feed
  — everything is behavior-only. Standing up a lightweight feed (even just a static IOC
  list ingested into Wazuh's CDB list feature) and matching it against Zeek's `dnsquery` or
  `resolved_by` fields would be a natural next step.
- **Sysmon Event ID 10 (ProcessAccess).** Directly relevant to closing the T1003 gap above —
  worth learning what a well-tuned LSASS-access rule looks like without generating noise
  from legitimate AV/EDR access to `lsass.exe`, which is a known noisy source for this event
  type.
