# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this repository.

## What This Is

A three-project cybersecurity homelab portfolio built on a single PC using Proxmox virtualization.
Projects are integrated — logs from Project 1 feed into Project 2, and Project 3 targets live in
the same segmented network built in Project 1.

## Repository Layout

| Folder | Project |
|--------|---------|
| `network-segmentation/` | Project 1 — VLAN segmentation + pfSense firewall + Suricata IDS |
| `siem-wazuh/` | Project 2 — Wazuh SIEM deployment and detection rules |
| `vulnerability-management/` | Project 3 — Nessus vulnerability scanning and remediation |
| `docs/architecture/` | Network topology diagrams, VLAN scheme, VM inventory |
| `docs/decisions/` | Architecture Decision Records (ADRs) |
| `docs/lessons-learned/` | Post-project retrospectives |
| `scripts/` | Automation scripts (backup-configs, reset-lab, test-traffic) |

## File Roles

| File | Purpose |
|------|---------|
| `README.md` | Master portfolio overview — tools matrix, project status table, architecture embed |
| `CLAUDE.local.md` | Local-only private notes — VM IPs, Proxmox access, lab credentials — gitignored, never publish |
| `.gitignore` | Excludes secrets, live IPs, pcaps, nessus reports, raw logs |

## Sensitive Data Rules

Private lab details (VM IPs, Proxmox host IP, credentials, MAC addresses) belong exclusively
in `CLAUDE.local.md`, which is gitignored. All docs use sanitized placeholders (e.g., `10.0.10.X`).
See `.claude/rules/sensitive-data.md` for the full list of what must never be committed.

## Documentation Conventions

- **Walkthrough files** follow a strict structure: Objective → Prerequisites → Steps → Verification → Troubleshooting → References
- **ADRs** cover: Status → Context → Options considered → Decision → Consequences
- **Use case files** (Project 2) cover: Threat scenario → Log source → Rule IDs → Alert sample → Response action → False positive notes
- **Remediation files** (Project 3) cover: Finding + CVE → CVSS score → Affected host → Root cause → Fix steps → Verification → Post-scan result
- Commit messages use Conventional Commits: `type(scope): description` — types: `feat`, `fix`, `docs`, `chore`

## Branch Strategy

One branch per walkthrough or major section:
- `network-segmentation/pfsense-install`
- `network-segmentation/vlan-config`
- `network-segmentation/suricata-setup`
- `siem/wazuh-deploy`
- `siem/custom-rules`
- `vuln-mgmt/nessus-baseline`
- `vuln-mgmt/remediation-round1`
