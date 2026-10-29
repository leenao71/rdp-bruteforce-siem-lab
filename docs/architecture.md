# Lab Architecture

Three VMs on an isolated UTM Host Only network (no internet or home network access):

- **wazuh-server** (192.168.128.3) — Ubuntu 24.04 ARM64, Wazuh 4.14.8 (manager, indexer, dashboard)
- **win11-victim** (192.168.128.4) — Windows 11, Sysmon (SwiftOnSecurity config) + Wazuh agent
- **kali-attacker** (192.168.128.5) — Kali Linux, attack tooling (Hydra, xfreerdp)

Attack traffic flows: kali-attacker → win11-victim (RDP, port 3389)
Telemetry flows: win11-victim (Sysmon + Windows Security log) → Wazuh agent → wazuh-server (manager → indexer → dashboard)

