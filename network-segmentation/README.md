# Project 1: Network Segmentation + Firewall + IDS

## Objective

Build a segmented virtual network using pfSense as the perimeter firewall and Suricata as an
inline IDS/IPS. Demonstrate VLAN isolation, inter-zone firewall policy enforcement, and
intrusion detection tuning — all running on Proxmox VE.

## Architecture

![Network Topology](../docs/architecture/network-topology.png)
*Diagram in [docs/architecture/](../docs/architecture/)*

## Components

| Component | Tool | Role |
|-----------|------|------|
| Hypervisor | Proxmox VE 8.x | Hosts all VMs, provides VLAN-aware virtual networking |
| Firewall/Router | pfSense 2.7.x | Inter-VLAN routing, firewall rules, DHCP per zone |
| IDS/IPS | Suricata 7.x | Inline traffic inspection on WAN and LAN interfaces |

## VLAN Zones

| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | MANAGEMENT | 10.0.10.0/24 | Proxmox host, pfSense management interface |
| 20 | TRUSTED | 10.0.20.0/24 | Daily-use VMs, workstation |
| 30 | SERVERS | 10.0.30.0/24 | Wazuh SIEM, internal services |
| 40 | DMZ | 10.0.40.0/24 | Exposed/public-facing services |
| 60 | VULN-LAB | 10.0.60.0/24 | Vulnerable targets — fully isolated from all other VLANs |
| 99 | GUEST | 10.0.99.0/24 | Internet-only, no LAN access |

## What I Built

- Proxmox VE installed on bare metal with VLAN-aware Linux bridges
- pfSense VM with one interface per VLAN zone, acting as the default gateway for each
- Suricata deployed on pfSense via the package manager, running in inline IDS mode on WAN
- Custom Suricata rules for port scan and SSH brute force detection
- Inter-VLAN firewall rules enforcing least-privilege access between zones
- Verified isolation: VULN-LAB (60) blocked from all internal VLANs

## Key Outcomes

- Full inter-VLAN traffic segmentation confirmed via ICMP ping tests
- Suricata detecting Nmap scans and Metasploit probes from VULN-LAB (60)
- pfSense firewall logs forwarded to Wazuh (Project 2) via syslog

## Challenges and How I Solved Them

*Populated as the build progresses.*

## Walkthroughs

1. [Proxmox VM Setup](walkthroughs/01-proxmox-vm-setup.md)
2. [pfSense Install](walkthroughs/02-pfsense-install.md)
3. [VLAN Creation](walkthroughs/03-vlan-creation.md)
4. [Firewall Rules](walkthroughs/04-firewall-rules.md)
5. [Suricata Install](walkthroughs/05-suricata-install.md)
6. [Suricata Tuning](walkthroughs/06-suricata-tuning.md)
7. [Verification Testing](walkthroughs/07-verification-testing.md)

## Skills Demonstrated

VLANs, inter-VLAN routing, stateful firewall rule management, IDS deployment and tuning,
Proxmox networking, pfSense administration, Suricata rule writing

## References

- [pfSense Documentation](https://docs.netgate.com/pfsense/en/latest/)
- [Suricata User Guide](https://suricata.readthedocs.io/en/latest/)
- [Proxmox VE Admin Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html)
