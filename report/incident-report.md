# Incident Report: RDP Brute-Force Attack Detection

## Summary
On 2026-10-28, a simulated RDP brute-force attack was conducted against a Windows 11
endpoint (win11-victim, 192.168.128.4) from an isolated attacker host (kali-attacker,
192.168.128.5). The attack was detected by a custom Wazuh correlation rule, mapped to
MITRE ATT&CK technique T1110 (Brute Force).

## Environment
- Wazuh server: wazuh-server (192.168.128.3), Wazuh v4.14.8, Ubuntu 24.04 ARM64
- Victim: win11-victim (192.168.128.4), Windows 11, Sysmon (SwiftOnSecurity config) + Wazuh agent
- Attacker: kali-attacker (192.168.128.5), Kali Linux
- Network: isolated UTM Host Only network, no internet/home network route (verified:
  ping to 8.8.8.8 returns "Network is unreachable"; RDP port 3389 on the victim
  confirmed reachable from the attacker host)

## Timeline
| Time (UTC) | Event |
|---|---|
| 19:13:49 | First failed RDP authentication attempt (Event ID 4625) recorded on win11-victim |
| 19:13:50–19:13:51 | 6 additional failed authentication attempts in rapid succession |
| 19:13:51 | Custom Wazuh rule 100100 fires — "Multiple Windows RDP logon failures against the same account - possible brute force attack" (Level 10) |

## Attack Method
A scripted loop using `xfreerdp` (FreeRDP client) attempted authentication against the
`victim` account using a password list of 8 candidate passwords. Hydra's RDP module
(thc-hydra) was initially tried but proved unreliable against this target configuration
("account not active for remote desktop" errors regardless of correct credentials); the
attack was conducted with a scripted xfreerdp loop instead, forcing legacy RDP security
(/sec:rdp) to bypass an NLA/Kerberos negotiation failure encountered during testing.

## Evidence
- Windows Security Event Log: 34 Event ID 4625 entries, Logon Type 3, target account
  `victim`, failure status 0xC000006D / substatus 0xC000006A (bad username or password).
  See screenshot: 05-failed-logons.png
- Note: 34 total Event ID 4625 failures were recorded across multiple attack test runs
  during the testing session; the custom rule 100100 correctly fired once per
  qualifying burst (5+ failures within 120 seconds), rather than once per individual
  event, which is the intended frequency-correlation behavior.
- Source Network Address field was not populated in these events — consistent with
  NTLM pre-authentication-stage failures (Logon Type 3) rather than full interactive
  RDP session failures (Logon Type 10), which do populate source IP.
- Wazuh Threat Hunting: events ingested and correctly classified by built-in rule 60122
  ("Logon Failure - Unknown user or bad password"). See screenshot: 05-wazuh-raw-events.png
- Custom rule 100100 fired correctly at 19:13:51, Level 10, MITRE ATT&CK T1110.
  See screenshot: 06-custom-rule-fired.png

## Detection Rule
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

**Design note:** the rule originally included `<same_source_ip />` to correlate failures
by attacker IP. Testing showed these specific events (NTLM pre-authentication failures,
Logon Type 3) do not populate a source IP field in Windows' Security log — expected
behavior at this authentication stage, distinct from full interactive RDP logons (Type
10). The rule was adapted to correlate on repeated failures against the same target
account instead, accurate for this single-attacker lab but would need source-IP or
source-workstation correlation in a multi-user production environment to avoid false
positives from legitimate users mistyping passwords.

## MITRE ATT&CK Mapping
- **T1110 — Brute Force**: repeated authentication attempts against a single account
  using a list of common/weak passwords.

## Containment Recommendations
1. Enforce account lockout policy (disabled during lab testing to allow repeated runs;
   should be re-enabled with a sensible threshold, e.g. 5 attempts / 15 minute lockout,
   in production).
2. Require Network Level Authentication (NLA) for RDP — this lab disabled NLA to work
   around a client-side negotiation issue; NLA requires pre-authentication before a
   full session is established and significantly raises the bar for brute-force tools.
3. Restrict RDP exposure to a VPN or jump host rather than direct network access.
4. Deploy a strong password policy / MFA for remote access accounts.
5. Alert on rule 100100 (or equivalent) to SOC on-call for investigation within SLA.

## Limitations
- Single attacker source, single target account — rule does not yet correlate by
  source IP (not available in this telemetry) or distinguish distributed attacks from
  multiple sources.
- Lab environment: NLA and security layer were deliberately weakened to enable testing;
  production systems should not run with these settings.
