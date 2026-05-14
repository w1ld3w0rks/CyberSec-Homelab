# Firewall Policy

## Design Principles

- **Default deny** — all inter-VLAN traffic is blocked unless explicitly permitted
- **Least privilege** — each zone gets only the access it needs to function
- **VULN-LAB isolation** — VLAN 60 has zero access to any internal zone; internet access only for downloading tools
- **Admin access** — MANAGEMENT (10) can reach all zones for administration
- **Logging** — all deny rules log to syslog (forwarded to Wazuh)

## Firewall Rules by Interface

### WAN (Internet → LAN)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | Any | Any | Any | Block | Default deny inbound; no exposed services |

### MANAGEMENT (VLAN 10)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 10 | Any | Any | Allow | Admin zone has full access |

### TRUSTED (VLAN 20)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 20 | VLAN 10 | Any | Block | No access to management zone |
| 2 | VLAN 20 | VLAN 60 | Any | Block | No access to vuln lab |
| 3 | VLAN 20 | VLAN 30 | Any | Allow | Access to internal servers |
| 4 | VLAN 20 | Internet | Any | Allow | Normal internet access |

### SERVERS (VLAN 30)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 30 | VLAN 60 | Any | Block | Wazuh cannot reach vuln targets directly |
| 2 | VLAN 30 | VLAN 20 | TCP 1514,1515 | Allow | Wazuh agent comms from trusted VMs |
| 3 | VLAN 30 | Internet | TCP 443 | Allow | Package updates only |
| 4 | VLAN 30 | Any | Any | Block | Default deny all other traffic |

### DMZ (VLAN 40)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 40 | Any LAN | Any | Block | DMZ cannot initiate connections to LAN |
| 2 | VLAN 40 | Internet | Any | Allow | Outbound internet access |

### VULN-LAB (VLAN 60)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 60 | 10.0.0.0/8 | Any | Block | Block all RFC1918 — no internal access |
| 2 | VLAN 60 | Internet | TCP 80,443 | Allow | Tool downloads only |
| 3 | VLAN 60 | Any | Any | Block | Default deny all other |

### GUEST (VLAN 99)

| # | Source | Destination | Port/Proto | Action | Notes |
|---|--------|-------------|------------|--------|-------|
| 1 | VLAN 99 | 10.0.0.0/8 | Any | Block | No LAN access |
| 2 | VLAN 99 | Internet | Any | Allow | Internet only |

## Logging Configuration

All block rules have logging enabled. pfSense sends logs to Wazuh via syslog (UDP 514):
- Source: pfSense management IP (10.0.10.1)
- Destination: Wazuh manager (10.0.30.10)
- Format: RFC 5424
