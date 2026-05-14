# ADR-002: Firewall/Router Selection

**Status:** Accepted

## Context

The lab needs a software firewall/router VM to handle inter-VLAN routing, DHCP per zone,
stateful firewall rules, and serve as the host for an IDS/IPS. It must be free and support
VLAN sub-interfaces on a single trunk NIC.

## Options Considered

### Option A: pfSense CE (Community Edition)

- **Pros:** Free, mature, widely used in production environments, excellent VLAN support, Suricata and Snort available as packages, large documentation base, relevant to job market (Netgate/pfSense certs exist)
- **Cons:** Netgate has been shifting focus to pfSense Plus (commercial), CE updates are slower; UI can be complex for beginners

### Option B: OPNsense

- **Pros:** Free, active open source community, modern UI, frequent updates, built-in Suricata support via Sensei/IDS plugin
- **Cons:** Less market recognition than pfSense; slightly less documentation for home lab setups

### Option C: VyOS

- **Pros:** Free, CLI-driven (good for CCNA practice), powerful routing features
- **Cons:** No GUI, no built-in IDS/IPS package, steeper learning curve for firewall rule management, overkill for this lab's firewall needs

### Option D: Linux iptables/nftables VM

- **Pros:** Full control, deeply educational
- **Cons:** Significant manual configuration, no GUI for firewall rule visualization, harder to document and demonstrate to interviewers

## Decision

**pfSense CE 2.7.x**

pfSense has the best balance of industry relevance, documentation, and integrated Suricata
support. The ability to add Suricata as a package eliminates the need to manage a separate
IDS VM, simplifying the architecture.

## Consequences

- Suricata runs on pfSense rather than a dedicated VM — reduces VM count and resource usage
- pfSense CE may lag behind pfSense Plus in features, but CE is sufficient for this lab's scope
- The pfSense skill is directly transferable to production environments and CCNA-adjacent certification paths
