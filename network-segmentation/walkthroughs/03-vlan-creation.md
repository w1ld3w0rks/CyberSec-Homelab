# 03: VLAN Creation and Interface Assignment

## Objective

Create all six lab VLANs inside pfSense, assign each a dedicated interface, and configure
DHCP scopes per zone so VMs get correct IPs based on their VLAN tag.

## Prerequisites

- pfSense installed and reachable at `https://10.0.10.1` ([Walkthrough 02](02-pfsense-install.md))
- Proxmox `vmbr0` bridge is VLAN-aware
- pfSense LAN interface (`vtnet0`) connected to `vmbr0` as a trunk (no VLAN tag)

## Steps

### 1. Create VLANs on the LAN Interface

1. Go to **Interfaces → Assignments → VLANs → Add**
2. Create one entry per VLAN:

| Parent Interface | VLAN Tag | Description |
|-----------------|----------|-------------|
| vtnet0 | 10 | MANAGEMENT |
| vtnet0 | 20 | TRUSTED |
| vtnet0 | 30 | SERVERS |
| vtnet0 | 40 | DMZ |
| vtnet0 | 60 | VULN-LAB |
| vtnet0 | 99 | GUEST |

For each: click **Add**, select `vtnet0` as parent, enter the VLAN tag and description, **Save**.

### 2. Assign VLAN Interfaces

1. Go to **Interfaces → Assignments**
2. For each VLAN created above, click **Add** next to the corresponding `vtnet0.XX` entry
3. After all six are added, they appear as `OPT1` through `OPT6` — rename them next

### 3. Rename and Enable Each Interface

For each interface (**Interfaces → OPT1**, etc.):

1. Check **Enable interface**
2. Set **Description** to the zone name (e.g., `MANAGEMENT`, `TRUSTED`)
3. Set **IPv4 Configuration Type** to **Static IPv4**
4. Enter the gateway IP and subnet:

| Interface | Description | IPv4 Address | Subnet |
|-----------|-------------|--------------|--------|
| OPT1 (vtnet0.10) | MANAGEMENT | 10.0.10.1 | /24 |
| OPT2 (vtnet0.20) | TRUSTED | 10.0.20.1 | /24 |
| OPT3 (vtnet0.30) | SERVERS | 10.0.30.1 | /24 |
| OPT4 (vtnet0.40) | DMZ | 10.0.40.1 | /24 |
| OPT5 (vtnet0.60) | VULN-LAB | 10.0.60.1 | /24 |
| OPT6 (vtnet0.99) | GUEST | 10.0.99.1 | /24 |

> Leave **IPv4 Upstream Gateway** blank — these are LAN-side interfaces, not WAN.

5. **Save → Apply Changes** after each interface

### 4. Configure DHCP per VLAN

Go to **Services → DHCP Server** — each enabled interface appears as a tab.

Configure each tab:

**MANAGEMENT (vtnet0.10)**
| Field | Value |
|-------|-------|
| Enable | checked |
| Range | 10.0.10.100 – 10.0.10.200 |
| DNS servers | 10.0.10.1 |

**TRUSTED (vtnet0.20)**
| Field | Value |
|-------|-------|
| Enable | checked |
| Range | 10.0.20.100 – 10.0.20.200 |
| DNS servers | 10.0.20.1 |

**SERVERS (vtnet0.30)**
| Field | Value |
|-------|-------|
| Enable | unchecked | 
| *(Static IPs only — no DHCP)* | |

**DMZ (vtnet0.40)**
| Field | Value |
|-------|-------|
| Enable | checked |
| Range | 10.0.40.100 – 10.0.40.150 |
| DNS servers | 10.0.40.1 |

**VULN-LAB (vtnet0.60)**
| Field | Value |
|-------|-------|
| Enable | checked |
| Range | 10.0.60.100 – 10.0.60.200 |
| DNS servers | 10.0.60.1 |

**GUEST (vtnet0.99)**
| Field | Value |
|-------|-------|
| Enable | checked |
| Range | 10.0.99.100 – 10.0.99.200 |
| DNS servers | 1.1.1.1 |

> GUEST uses an external DNS server — it has no access to internal pfSense-served DNS.

Click **Save** on each tab.

### 5. Add Firewall Allow Rules for DHCP

pfSense's default-deny will block DHCP discovery. Add a rule on each VLAN interface
to allow DHCP before applying zone isolation rules (covered in Walkthrough 04):

1. **Firewall → Rules → [each VLAN interface tab] → Add**
2. Create this rule on every VLAN interface:

| Field | Value |
|-------|-------|
| Action | Pass |
| Interface | (current VLAN) |
| Protocol | UDP |
| Source | any |
| Destination | (this interface address) |
| Destination Port | 67 (BOOTPS) |
| Description | Allow DHCP |

> pfSense actually handles DHCP internally and this rule is not strictly required, but it
> documents intent and prevents confusion when auditing rules later.

### 6. Assign VLAN Tags to VMs in Proxmox

For each VM in Proxmox, set its network device VLAN tag to match its intended zone:

1. **VM → Hardware → Network Device → Edit**
2. Set **VLAN Tag** to the appropriate ID:

| VM | VLAN Tag |
|----|----------|
| Wazuh | 30 |
| Kali (attack box) | 60 |
| Metasploitable 2 | 60 |
| Windows Server 2019 | 60 |
| Any trusted workstation VM | 20 |

3. Click **OK** — changes take effect on next VM boot (no reboot of Proxmox required)

## Verification

For each VLAN, boot a test VM tagged to that VLAN and confirm:

1. VM receives a DHCP lease in the correct range:
```bash
ip addr show eth0
# Expected: 10.0.XX.1XX/24
```

2. VM can ping its gateway:
```bash
ping -c 3 10.0.XX.1
# Expected: 3 packets transmitted, 3 received
```

3. VM cannot yet reach other VLANs (default-deny is in effect until Walkthrough 04):
```bash
ping -c 3 10.0.20.1   # from a VLAN 60 VM — should fail
```

4. In pfSense, **Status → DHCP Leases** shows active leases per interface.

## Troubleshooting

**VM gets a 169.254.x.x address (APIPA):**
- Confirm the VM's Proxmox network device has the correct VLAN tag set
- Confirm DHCP is enabled on the matching pfSense interface tab
- Check **Status → System Logs → DHCP** for lease activity

**pfSense interface shows as DOWN:**
- Confirm the VLAN sub-interface was created correctly under **Interfaces → VLANs**
- Ensure the parent interface (`vtnet0`) has link — **Status → Interfaces** should show it UP

**DHCP leases not appearing in Status → DHCP Leases:**
- Restart the DHCP service: **Status → Services → dhcpd → Restart**
- Check **System Logs → DHCP** for errors (common cause: overlapping subnet ranges)

**All VMs on the same VLAN getting each other's traffic:**
- This is expected within a VLAN — isolation only applies between VLANs via firewall rules (Walkthrough 04)

## References

- [pfSense VLAN Configuration](https://docs.netgate.com/pfsense/en/latest/vlan/index.html)
- [pfSense DHCP Server](https://docs.netgate.com/pfsense/en/latest/services/dhcp/index.html)
- [Proxmox VLAN-Aware Bridge](https://pve.proxmox.com/wiki/Network_Configuration#_vlan_802_1q)
