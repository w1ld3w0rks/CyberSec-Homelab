# Homelab Cybersecurity Portfolio

**Author:** Cory Fernandez | **Role Target:** Network/Security Admin | **Status:** In Progress

## Overview
Three integrated homelab projects built on a single PC using Proxmox virtualization.
Logs from Project 1 feed into Project 2. Project 3 targets live inside the segmented
network built in Project 1.

## Projects

| # | Project | Status | Key Skills |
|---|---------|--------|------------|
| 1 | [Network Segmentation + IDS/IPS](project1-network-segmentation/) | 🔄 In Progress | VLANs, pfSense, Suricata |
| 2 | [SIEM with Wazuh](project2-siem/) | 📋 Planned | Log aggregation, detection rules |
| 3 | [Vulnerability Management](project3-vuln-management/) | 📋 Planned | Nessus, CVSS, remediation |

## Tools & Technologies

| Category | Tools |
|----------|-------|
| Hypervisor | Proxmox VE 8.x |
| Firewall | pfSense 2.7.x |
| IDS/IPS | Suricata 7.x |
| SIEM | Wazuh 4.x |
| Vuln Scanner | Nessus Essentials |
| OS Targets | Ubuntu Server 22.04, Metasploitable 2, Windows Server 2019 Eval |