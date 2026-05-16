# 04: Firewall Rule Configuration

## Objective

Implement zone-based firewall rules in pfSense that enforce least-privilege access between
all six VLANs. After this walkthrough, inter-VLAN traffic will be controlled by explicit
policy rather than pfSense's default allow-all LAN behavior.

## Prerequisites

- All VLAN interfaces created and DHCP configured ([Walkthrough 03](03-vlan-creation.md))
- VMs in each VLAN can reach their gateway and receive a DHCP lease
- Understand the policy defined in [firewall-policy.md](../design/firewall-policy.md)

## Steps

### 1. Remove the Default LAN Allow-All Rule

pfSense ships with a catch-all allow rule on the LAN interface. Remove it to enforce
default-deny before adding explicit rules:

1. **Firewall → Rules → LAN**
2. Locate the rule: *"Default allow LAN to any rule"*
3. Click the **Delete** (trash) icon → **Apply Changes**

> Do this for the MANAGEMENT interface tab as well if it inherited a default allow rule.
> The MANAGEMENT zone gets its own explicit allow-all rule in step 2.

### 2. MANAGEMENT (VLAN 10) Rules

MANAGEMENT is the admin zone — full access to all zones.

**Firewall → Rules → MANAGEMENT → Add**

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | MANAGEMENT net | Any | Any | Pass | Admin zone — full access |

> A single pass-all rule is appropriate here because MANAGEMENT access should already
> be restricted at the network level (only trusted hosts get VLAN 10 tags in Proxmox).

### 3. TRUSTED (VLAN 20) Rules

**Firewall → Rules → TRUSTED → Add** (create in order — pfSense evaluates top to bottom)

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | TRUSTED net | MANAGEMENT net | Any | Block | No access to management zone |
| 2 | Any | TRUSTED net | VULN-LAB net | Any | Block | No access to vuln lab |
| 3 | Any | TRUSTED net | SERVERS net | Any | Pass | Access to internal servers |
| 4 | Any | TRUSTED net | DMZ net | Any | Pass | Access to DMZ services |
| 5 | Any | TRUSTED net | !RFC1918 | Any | Pass | Internet access |

> `!RFC1918` is pfSense shorthand for "not any private address range." Use the alias
> **RFC1918** (built-in) as the destination and invert it with the **not** checkbox.

**Create the RFC1918 alias** if not already present:
- **Firewall → Aliases → Add**
- Name: `RFC1918`, Type: Network
- Networks: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`

### 4. SERVERS (VLAN 30) Rules

**Firewall → Rules → SERVERS → Add**

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | SERVERS net | VULN-LAB net | Any | Block | Wazuh cannot reach vuln targets |
| 2 | TCP | SERVERS net | TRUSTED net | 1514, 1515 | Pass | Wazuh agent communication |
| 3 | TCP | SERVERS net | Any | 443 | Pass | Package/update downloads |
| 4 | UDP | SERVERS net | Any | 53 | Pass | DNS resolution |
| 5 | Any | SERVERS net | Any | Any | Block | Default deny all other |

> To specify multiple ports in one rule: **Firewall → Aliases → Add** an alias of type
> Port named `WazuhPorts` with values `1514` and `1515`, then reference it in rule 2.

### 5. DMZ (VLAN 40) Rules

**Firewall → Rules → DMZ → Add**

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | DMZ net | RFC1918 | Any | Block | DMZ cannot reach any LAN zone |
| 2 | Any | DMZ net | Any | Any | Pass | Outbound internet access |

### 6. VULN-LAB (VLAN 60) Rules

This is the most restrictive zone — vulnerable targets must never reach internal networks.

**Firewall → Rules → VULN-LAB → Add**

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | VULN-LAB net | RFC1918 | Any | Block | Block all internal RFC1918 |
| 2 | TCP | VULN-LAB net | Any | 80, 443 | Pass | Tool downloads only |
| 3 | UDP | VULN-LAB net | Any | 53 | Pass | DNS resolution |
| 4 | Any | VULN-LAB net | Any | Any | Block | Default deny all other |

> Rule 1 must be first. If internet access (rules 2–3) is not needed during active
> testing, disable those rules to fully air-gap the zone.

### 7. GUEST (VLAN 99) Rules

**Firewall → Rules → GUEST → Add**

| # | Proto | Source | Destination | Port | Action | Description |
|---|-------|--------|-------------|------|--------|-------------|
| 1 | Any | GUEST net | RFC1918 | Any | Block | No LAN access |
| 2 | Any | GUEST net | Any | Any | Pass | Internet access |

### 8. Enable Logging on All Block Rules

For every Block rule created above:

1. Click the rule's **Edit** (pencil) icon
2. Expand **Extra Options**
3. Check **Log packets that are handled by this rule**
4. **Save → Apply Changes**

This feeds pfSense firewall logs to Wazuh (Project 2).

### 9. Configure Syslog Forwarding to Wazuh

1. **Status → System Logs → Settings**
2. Scroll to **Remote Logging**:

| Field | Value |
|-------|-------|
| Enable Remote Logging | checked |
| Source Address | SERVERS (or default) |
| IP Protocol | IPv4 |
| Remote log server 1 | `10.0.30.10:514` |
| Remote Syslog Contents | Firewall Events |

3. **Save**

> Wazuh must be running and its syslog input configured before logs arrive — covered in Project 2.

### 10. Apply All Changes

Click **Apply Changes** in the orange banner at the top of the Firewall Rules page.
pfSense reloads the ruleset — existing connections may briefly drop.

## Verification

Use a VM in each zone to confirm policy is enforced. From a VLAN 60 (VULN-LAB) VM:

```bash
# Should FAIL — blocked by rule 1
ping -c 3 10.0.20.1

# Should FAIL — blocked by rule 1
ping -c 3 10.0.30.10

# Should SUCCEED — rule 2 allows TCP 80/443
curl -I https://example.com
```

From a VLAN 20 (TRUSTED) VM:

```bash
# Should FAIL — blocked by rule 1
ping -c 3 10.0.10.1

# Should SUCCEED — rule 3 allows SERVERS access
ping -c 3 10.0.30.10

# Should SUCCEED — rule 5 allows internet
curl -I https://example.com
```

In pfSense, confirm blocked traffic is logged:
- **Status → System Logs → Firewall**
- Filter by interface VULN-LAB — blocked pings from 10.0.60.x should appear

## Troubleshooting

**Legitimate traffic being blocked unexpectedly:**
- pfSense evaluates rules top-to-bottom, first match wins — check rule order
- Use **Diagnostics → Packet Capture** on the affected interface to see if traffic arrives
- Use **Firewall → Logs** and filter by source IP to find which rule is matching

**Block rules not appearing in logs:**
- Confirm logging is checked on the specific block rule (step 8)
- **Status → System Logs → Settings** — ensure **Firewall Events** is selected

**Syslog not reaching Wazuh:**
- Confirm Wazuh VM is up and on `10.0.30.10`
- Test with: **Diagnostics → Test Port** → host `10.0.30.10`, port `514`, UDP
- Check Wazuh's `/var/ossec/logs/ossec.log` for incoming syslog entries

**After applying rules, can no longer reach pfSense web UI:**
- If locked out from MANAGEMENT, connect via Proxmox console → pfSense menu option
  **8) Shell** → `pfctl -d` to temporarily disable the firewall → fix the rule

## References

- [pfSense Firewall Rules](https://docs.netgate.com/pfsense/en/latest/firewall/index.html)
- [pfSense Aliases](https://docs.netgate.com/pfsense/en/latest/firewall/aliases.html)
- [pfSense Remote Syslog](https://docs.netgate.com/pfsense/en/latest/monitoring/logs/remote.html)
