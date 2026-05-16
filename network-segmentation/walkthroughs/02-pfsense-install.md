# 02: pfSense VM Installation and Initial Configuration

## Objective

Create a pfSense VM in Proxmox, configure it with a VLAN trunk interface, and complete
initial setup so it acts as the default gateway and DHCP server for all lab VLANs.

## Prerequisites

- Proxmox VE installed with a VLAN-aware `vmbr0` bridge ([Walkthrough 01](01-proxmox-vm-setup.md))
- pfSense CE ISO downloaded
- At least 2 GB RAM and 16 GB disk allocated for the pfSense VM

## Steps

### 1. Download pfSense CE ISO

1. Go to [pfsense.org/download](https://www.pfsense.org/download/)
2. Select:
   - **Architecture:** AMD64
   - **Installer:** DVD Image (ISO) Installer
   - **Mirror:** closest to you
3. Download and verify the SHA256 checksum

### 2. Upload ISO to Proxmox

1. In the Proxmox web UI, go to **local (proxmox) → ISO Images → Upload**
2. Select the pfSense ISO and upload
3. Wait for the upload to complete — it appears in the ISO list when done

### 3. Create the pfSense VM

1. Click **Create VM** in the top-right
2. Fill in each tab:

**General**
| Field | Value |
|-------|-------|
| Node | proxmox |
| VM ID | 100 |
| Name | pfsense |

**OS**
| Field | Value |
|-------|-------|
| ISO image | pfSense-CE-2.7.x-RELEASE-amd64.iso |
| Type | Other |

**System**
- Leave defaults (SeaBIOS, IDE SCSI controller)

**Disks**
| Field | Value |
|-------|-------|
| Bus | VirtIO Block |
| Size | 16 GB |

**CPU**
| Field | Value |
|-------|-------|
| Sockets | 1 |
| Cores | 2 |
| Type | host |

**Memory**
| Field | Value |
|-------|-------|
| Memory | 2048 MB |

**Network**
| Field | Value |
|-------|-------|
| Bridge | vmbr0 |
| VLAN Tag | *leave blank* (trunk — pfSense sees all VLANs) |
| Model | VirtIO |

3. Click **Finish** — do not start yet

### 4. Add a Second Network Interface (Optional but Recommended)

A dedicated WAN interface keeps WAN traffic separate from the VLAN trunk. If your host has
only one physical NIC, skip this and use a VLAN for WAN simulation instead.

If you have a second NIC (`enp4s0`), create a second bridge in Proxmox:

1. **Datacenter → proxmox → Network → Create → Linux Bridge**
   - Name: `vmbr1`
   - Bridge port: `enp4s0`
   - Leave VLAN aware unchecked (WAN is untagged)
2. Add a second network device to the pfSense VM: **VM 100 → Hardware → Add → Network Device**
   - Bridge: `vmbr1`, Model: VirtIO

> If using a single NIC, WAN can be a separate VLAN (e.g., VLAN 1) on `vmbr0`. Adjust the
> pfSense WAN interface assignment in step 7 accordingly.

### 5. Install pfSense

1. Start the VM and open the **Console** tab
2. Boot from ISO — select **Install pfSense**
3. Accept copyright notice
4. Select **Install** at the welcome screen
5. Keymap: **Continue with default keymap**
6. Partitioning: **Auto (ZFS)** → **Stripe** → select the VirtIO disk → **Install**
7. Reboot when prompted — remove ISO after reboot:
   - **VM 100 → Hardware → CD/DVD Drive → Edit → Do not use any media**

### 6. Assign Interfaces

After reboot, pfSense shows the interface assignment prompt:

```
Valid interfaces are:
vtnet0  xx:xx:xx:xx:xx:xx  (up)
vtnet1  xx:xx:xx:xx:xx:xx  (up)   ← only if second NIC added

Do VLANs need to be set up first? [y|n]  n
```

> Answer **n** here — you'll configure VLANs through the web UI in Walkthrough 03.

```
Enter the WAN interface name or 'a' for auto-detection:  vtnet1
Enter the LAN interface name or 'a' for auto-detection:  vtnet0
```

> If single NIC: assign `vtnet0` as WAN, leave LAN blank for now. VLANs will serve as LAN zones.

Confirm the assignment — pfSense assigns LAN a default IP of `192.168.1.1`.

### 7. Set the Management IP

At the pfSense console menu, select **2) Set interface(s) IP address**:

1. Select **2 - LAN (vtnet0)**
2. Enter IPv4 address: `10.0.10.1`
3. Subnet bit count: `24`
4. IPv4 upstream gateway: *(leave blank for LAN)*
5. IPv6: **n**
6. Enable DHCP on LAN: **n** (you'll configure per-VLAN DHCP via the web UI)
7. Revert to HTTP: **n** (keep HTTPS)

pfSense confirms the web UI is available at `https://10.0.10.1`.

### 8. Access the Web UI

From a VM on VLAN 10 (or temporarily from the Proxmox host):

```
https://10.0.10.1
```

Default credentials:
- Username: `admin`
- Password: `pfsense`

> **Change the password immediately** — go to **System → User Manager → admin → Edit**.

### 9. Run the Setup Wizard

The setup wizard launches on first login:

| Step | Setting |
|------|---------|
| Hostname | `pfsense` |
| Domain | `lab.local` |
| Primary DNS | `1.1.1.1` |
| Secondary DNS | `8.8.8.8` |
| NTP Server | `pool.ntp.org` |
| WAN type | DHCP (if getting IP from home router) or Static |
| LAN IP | `10.0.10.1 / 24` (already set) |
| Admin password | *(set a strong password)* |

Click **Reload** to apply. pfSense restarts services.

### 10. Disable Default Block Rules for RFC1918 on WAN (Lab Only)

By default pfSense blocks private IPs on WAN. If your WAN is your home LAN (another
private subnet), this will block management traffic:

1. **Interfaces → WAN**
2. Uncheck **Block private networks and loopback addresses**
3. Uncheck **Block bogon networks** *(optional for lab)*
4. **Save → Apply Changes**

## Verification

- Web UI reachable at `https://10.0.10.1`
- **Status → Dashboard** shows WAN with a valid IP and LAN at `10.0.10.1`
- **Status → Interfaces** shows both interfaces UP
- Ping from pfSense console to `1.1.1.1`: **Diagnostics → Ping** → Host: `1.1.1.1` → **Ping**

## Troubleshooting

**Web UI unreachable at `https://10.0.10.1`:**
- Confirm the VM running your browser is tagged to VLAN 10 in Proxmox
- From the pfSense console, run option **7) Ping host** → ping `10.0.10.1` — if it fails, re-check the LAN IP assignment in step 7

**WAN shows no IP (DHCP):**
- Confirm `vtnet1` is connected to `vmbr1` and the physical NIC has link
- Console option **11) Restart webConfigurator** then check **Status → Interfaces**

**"GUI redirect loop" after wizard:**
- Clear browser cache or try a different browser — this is a known cosmetic issue after the setup wizard completes

**Can't reach internet from pfSense:**
- **Diagnostics → Routes** — confirm a default route exists via the WAN gateway
- **Diagnostics → DNS Lookup** → lookup `cloudflare.com` — if it fails, check DNS server settings in the wizard

## References

- [pfSense CE Download](https://www.pfsense.org/download/)
- [pfSense Installation Guide](https://docs.netgate.com/pfsense/en/latest/install/index.html)
- [pfSense Interface Configuration](https://docs.netgate.com/pfsense/en/latest/interfaces/index.html)
