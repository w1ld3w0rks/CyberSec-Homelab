# 01: Proxmox VE Installation and Network Setup

## Objective

Install Proxmox VE 8.x on bare metal and configure a VLAN-aware Linux bridge so all
VMs can be assigned to specific VLAN segments via a single physical NIC.

## Prerequisites

- Bare metal PC with VT-x/AMD-V enabled in BIOS
- At least 16 GB RAM, 500 GB storage
- USB drive (8 GB+) for Proxmox installer
- Network access for the host during install

## Steps

### 1. Download Proxmox VE ISO

1. Go to [proxmox.com/downloads](https://www.proxmox.com/en/downloads)
2. Download the latest **Proxmox VE ISO Installer** (e.g., `proxmox-ve_8.x-1.iso`)
3. Flash to USB with Balena Etcher or `dd`:

```bash
dd if=proxmox-ve_8.x-1.iso of=/dev/sdX bs=4M status=progress
```

### 2. Install Proxmox VE

1. Boot from USB, select **Install Proxmox VE (Graphical)**
2. Accept EULA
3. Select target disk — Proxmox will wipe it
4. Set country/timezone
5. Set root password and admin email
6. Network config:
   - Management IP: `10.0.10.10/24`
   - Gateway: *(your home router IP — temporary during setup)*
   - Hostname: `proxmox.lab`
7. Complete install and reboot

### 3. Access the Web UI

```
https://10.0.10.10:8006
```

Log in as `root` with the password set during install. Accept the self-signed cert warning.

> Dismiss the "no valid subscription" popup — CE is fully functional without a subscription.

### 4. Configure VLAN-Aware Bridge

By default Proxmox creates `vmbr0` linked to your physical NIC. Enable VLAN-aware mode:

1. Go to **Datacenter → proxmox → Network**
2. Click `vmbr0` → **Edit**
3. Check **VLAN aware** → **Apply**
4. Click **Apply Configuration** at the top

This allows VMs to be assigned VLAN tags directly on `vmbr0` rather than needing separate bridges per VLAN.

### 5. Verify Bridge Configuration

SSH into Proxmox and confirm:

```bash
cat /etc/network/interfaces
```

Expected output includes:

```
auto vmbr0
iface vmbr0 inet static
    address 10.0.10.10/24
    bridge-ports enp3s0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
```

> `enp3s0` will be your actual NIC name — verify with `ip link show`.

## Verification

- Web UI accessible at `https://10.0.10.10:8006`
- `vmbr0` shows **VLAN aware: yes** in the Network view
- `ip link show vmbr0` shows the bridge is UP

## Troubleshooting

**Web UI not reachable after install:**
- Confirm management IP was set correctly during install
- Check `ip addr show vmbr0` — if IP is missing, re-edit `/etc/network/interfaces` and run `ifreload -a`

**"No valid subscription" blocks features:**
- This popup is cosmetic — all features work on CE. Dismiss it each login or remove it via a browser console script (optional, cosmetic only)

**VMs can't communicate after enabling VLAN-aware:**
- Ensure each VM's network device has the correct VLAN tag set in its hardware config
- VLAN 0 or blank tag = untagged (avoid this in lab — always assign explicit VLAN IDs)

## References

- [Proxmox VE Admin Guide — Network Configuration](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_network_configuration)
- [Proxmox VLAN-Aware Bridge](https://pve.proxmox.com/wiki/Network_Configuration#_vlan_802_1q)
