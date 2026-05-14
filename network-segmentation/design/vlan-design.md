# VLAN Design

## Overview

Six VLANs provide zone-based traffic segmentation. All inter-VLAN routing goes through
pfSense, which enforces firewall policy at each zone boundary.

## VLAN Table

| VLAN ID | Name | Subnet | Gateway | DHCP Range | Purpose |
|---------|------|--------|---------|------------|---------|
| 10 | MANAGEMENT | 10.0.10.0/24 | 10.0.10.1 | 10.0.10.100–200 | Proxmox host, pfSense mgmt |
| 20 | TRUSTED | 10.0.20.0/24 | 10.0.20.1 | 10.0.20.100–200 | Daily-use VMs, workstation |
| 30 | SERVERS | 10.0.30.0/24 | 10.0.30.1 | Static only | Wazuh SIEM, internal services |
| 40 | DMZ | 10.0.40.0/24 | 10.0.40.1 | 10.0.40.100–150 | Public-facing services |
| 60 | VULN-LAB | 10.0.60.0/24 | 10.0.60.1 | 10.0.60.100–200 | Vulnerable targets (isolated) |
| 99 | GUEST | 10.0.99.0/24 | 10.0.99.1 | 10.0.99.100–200 | Internet-only, no LAN |

## Static IP Assignments

| Host | VLAN | IP | Role |
|------|------|----|------|
| Proxmox host | 10 | 10.0.10.10 | Hypervisor management |
| pfSense (mgmt) | 10 | 10.0.10.1 | Firewall/router gateway |
| Wazuh manager | 30 | 10.0.30.10 | SIEM server |
| Metasploitable 2 | 60 | 10.0.60.10 | Vulnerable Linux target |
| Windows Server 2019 | 60 | 10.0.60.20 | Vulnerable Windows target |

> All IPs above are internal lab addresses. Real values are in `CLAUDE.local.md` (gitignored).

## Proxmox Bridge Configuration

Each VLAN maps to a VLAN-aware Linux bridge on the Proxmox host:

```
vmbr0  — Physical NIC uplink (VLAN trunk to pfSense VM)
```

pfSense receives a trunk interface and creates sub-interfaces (VLANs) internally.
Each VM is assigned to a specific VLAN tag on `vmbr0`.

## Zone Isolation Rules (Summary)

| From Zone | To Zone | Policy |
|-----------|---------|--------|
| MANAGEMENT | Any | Allow (admin access) |
| TRUSTED | SERVERS | Allow |
| TRUSTED | MANAGEMENT | Deny |
| TRUSTED | VULN-LAB | Deny |
| SERVERS | TRUSTED | Allow (syslog, alerts) |
| DMZ | LAN | Deny all |
| VULN-LAB | Any internal | Deny all |
| GUEST | LAN | Deny all |
| Any | Internet | Allow (via WAN) |

Full rule logic is in [firewall-policy.md](firewall-policy.md).
