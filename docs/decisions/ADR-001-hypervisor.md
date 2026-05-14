# ADR-001: Hypervisor Selection

**Status:** Accepted

## Context

The lab requires running 5–7 VMs simultaneously (pfSense, Wazuh, Kali, Metasploitable,
Windows Server, plus optionally more targets). The hypervisor must support VLAN-aware
networking, run on commodity hardware, and be free to use indefinitely.

## Options Considered

### Option A: Proxmox VE

- **Pros:** Free and open source, KVM-based (near-native performance), excellent VLAN support via Linux bridges, active community, built-in web UI, supports both VM and LXC containers, snapshot support
- **Cons:** Steeper initial learning curve than VMware Workstation, no official free support

### Option B: VMware Workstation Pro

- **Pros:** Familiar UI, widely documented, good NAT/host-only networking
- **Cons:** No longer free as of 2024 for personal use (now requires Broadcom account), limited VLAN support without ESXi, not suitable for bare-metal server deployment

### Option C: VirtualBox

- **Pros:** Free, cross-platform, easy to use
- **Cons:** Poor performance for multiple concurrent VMs, no VLAN tagging on virtual NICs without workarounds, not suitable for a realistic network lab

### Option D: VMware ESXi (free tier)

- **Pros:** Enterprise-grade, strong VLAN support, widely used in industry
- **Cons:** Free tier removed by Broadcom in 2024, licensing cost prohibitive for a home lab

## Decision

**Proxmox VE 8.x**

Proxmox is the only option that is free, production-grade, supports proper VLAN-aware
networking, and runs well on commodity hardware. KVM performance is sufficient for all
planned workloads.

## Consequences

- Must learn Proxmox-specific networking (Linux bridges, VLAN-aware mode) — addressed in walkthrough 01
- pfSense requires a VM with a trunk interface rather than multiple physical NICs — handled via VLAN sub-interfaces inside pfSense
- Snapshot support means lab state can be saved before destructive testing (positive)
