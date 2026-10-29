# SIEM Detection Lab — RDP Brute-Force Detection (Wazuh, Sysmon, Windows 11, Kali Linux)

An isolated SOC lab built to simulate, detect, and investigate an RDP brute-force
attack using a self-hosted Wazuh SIEM, Sysmon, and a custom detection rule mapped to
MITRE ATT&CK.

## Overview

Three virtual machines on a fully isolated network (no internet or host network
access) simulate a minimal SOC environment:

- A Wazuh server collecting Windows Security and Sysmon logs from an endpoint
- A Windows 11 endpoint running Sysmon (SwiftOnSecurity config) and the Wazuh agent
- A Kali Linux attacker host used to simulate an RDP brute-force attack

The attack was detected via a custom Wazuh correlation rule (mapped to MITRE ATT&CK
T1110 — Brute Force), investigated, and written up as a full incident report.

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for the full architecture breakdown.

- **wazuh-server** (Ubuntu 24.04 ARM64) — Wazuh manager, indexer, dashboard
- **win11-victim** (Windows 11) — Sysmon + Wazuh agent
- **kali-attacker** (Kali Linux) — attack tooling

All three VMs run on an isolated UTM Host Only network. No traffic reaches the
internet or the host's home network — verified by failed pings to 8.8.8.8 and
confirmed reachability only between the lab VMs themselves.

## Setup

1. Hypervisor: UTM (Apple Silicon) or VirtualBox (Intel)
2. Wazuh server: Ubuntu 24.04 ARM64 + Wazuh 4.14.8 all-in-one install
3. Victim: Windows 11, Sysmon installed with the SwiftOnSecurity config
   (`configs/sysmonconfig-export.xml`), Wazuh agent installed and connected
4. Attacker: Kali Linux with FreeRDP (`xfreerdp`) and Hydra
5. All VMs switched from NAT/Shared networking to an isolated Host Only network
   before any attack traffic was generated

Full config files are in `configs/`.

## Attack Simulated

An RDP brute-force attack against the `victim` account on the Windows endpoint,
using a password list of 8 candidates. Hydra's RDP module proved unreliable against
this configuration (`account not active for remote desktop`, independent of
credential correctness), so the attack was run as a scripted loop using `xfreerdp`
instead, forcing legacy RDP security to work around an NLA/Kerberos negotiation issue
encountered in testing.

## Custom Detection Rule

```xml
<rule id="100100" level="10" frequency="5" timeframe="120">
  <if_matched_sid>60122</if_matched_sid>
  <description>Multiple Windows RDP logon failures against the same account - possible brute force attack</description>
  <mitre>
    <id>T1110</id>
  </mitre>
  <group>authentication_failures,</group>
</rule>
```

Fires when 5+ failed Windows logons (Event ID 4625) occur against the same account
within 120 seconds. Mapped to MITRE ATT&CK **T1110 — Brute Force**.

Full rule file: `rules/local_rules.xml`

## MITRE ATT&CK Mapping

- **T1110 — Brute Force**

## Incident Report Summary

34 failed logon attempts (Event ID 4625) were generated across testing; the custom
rule correctly correlated qualifying bursts (5+ failures within 120 seconds) into a
single high-severity alert rather than alerting per individual failure. Full
incident report: `report/incident-report.md`

## Limitations and Next Steps

- Single-source attack lab — detection correlates on target account, not source IP,
  since these particular Windows events (NTLM pre-authentication failures) don't
  populate a source IP field. A production rule would also correlate by source IP
  where available.
- Only one attack technique (T1110) and two endpoints in scope.
- **Next steps:** add Linux log collection, threat intel enrichment (e.g. VirusTotal
  integration), Wazuh active response to automatically block the attacking IP,
  and/or a parallel Microsoft Sentinel version of the same detection for comparison.

## Teardown

See `docs/architecture.md` and the Teardown section of the incident report for VM
snapshot/deletion steps once the lab is no longer needed.

## Interview Talking Points

1. **ARM64 driver architecture mismatch:** Sysmon's x64 binary failed to load its
   kernel driver on Windows 11 ARM64 — diagnosed via Windows service/driver state
   checks and resolved using the ARM64-native Sysmon binary.
2. **Silent pipeline failure:** after switching the lab to an isolated network, the
   Wazuh agent kept trying to reach its old pre-isolation IP and silently
   disconnected — found via the dashboard's Agents view, confirmed via agent logs,
   fixed by updating `ossec.conf` and restarting the service.
3. **Adapting detection logic to real telemetry:** the initial rule design
   correlated failures by source IP, but the actual Windows events (NTLM
   pre-authentication failures) don't populate that field — the rule was redesigned
   to correlate by target account instead, with the trade-off documented as a known
   limitation.
