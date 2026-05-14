# IDS Rule Strategy

## Suricata Deployment Model

Suricata runs as a package on pfSense in **inline IPS mode** on the WAN interface and
**IDS mode** (passive) on the LAN-side interfaces. This allows blocking at the WAN edge
while preserving full visibility into internal traffic.

| Interface | Mode | Purpose |
|-----------|------|---------|
| WAN | Inline IPS | Block known-bad inbound traffic |
| VLAN 60 (VULN-LAB) | IDS | Detect attack traffic from vuln targets |
| VLAN 20 (TRUSTED) | IDS | Detect lateral movement or compromised hosts |

## Ruleset Sources

| Ruleset | Update Frequency | Coverage |
|---------|-----------------|----------|
| Emerging Threats Open | Daily | Malware C2, exploits, scanners |
| Snort Community Rules | Daily | General signatures |
| Custom local.rules | Manual | Lab-specific detection scenarios |

## Detection Priorities

### High Priority (alert + block on WAN)

- Port scans (Nmap SYN/TCP/UDP)
- Exploit framework signatures (Metasploit shellcode patterns)
- Known malware C2 communication patterns
- SSH brute force (threshold: >10 attempts/60s from single source)

### Medium Priority (alert only, internal interfaces)

- ICMP sweeps between VLANs
- SMB enumeration traffic
- DNS queries to known-bad domains
- Outbound beaconing patterns (regular interval connections)

### Tuning / Suppression

Low-noise rules are suppressed to reduce alert fatigue. Suppression decisions are
documented in [rule-tuning.md](../configs/suricata/rule-tuning.md).

## Custom Rules Coverage

| Rule File | Detects |
|-----------|---------|
| `local.rules` | Lab-specific: VLAN 60 → VLAN 20 probe attempts, Nmap OS detection |

## Alert Forwarding

Suricata alerts are written to `eve.json` and forwarded to Wazuh via the Wazuh agent
on the pfSense host. This feeds Project 2's detection pipeline.
