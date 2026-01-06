# Hardware Specifications

## Overview

This document details the physical hardware components used in the Proxmox OPNsense VLAN Homelab. The setup is designed for cost-effectiveness while maintaining enterprise-like functionality and performance.

**Last Updated:** January 2026  
**Total Investment:** ~$[estimate] AUD  


---

## Network Topology Summary

```
[ISP Router] 
    │
    └─── [Cisco SG300-28 Switch]
            ├─── [OPNsense Firewall] (2x NICs)
            │       ├─── WAN (to ISP)
            │       └─── LAN (Trunk: VLANs 10,20,30,40)
            │
            └─── [Proxmox Host] (1x NIC, VLAN-aware bridge)
                    └─── VMs across all VLANs
```

---

## 1. Proxmox Virtualization Host

### ASUS NUC14 Pro

**Purpose:** Primary virtualization host running Proxmox VE

| Component | Specification | Notes |
|-----------|--------------|-------|
| **Model** | ASUS NUC14 Pro | Compact form factor |
| **CPU** | Intel Core Ultra 5 125H| [14] cores, [18] threads |
| **RAM** | [64]GB DDR5 |
| **Storage** | [1]TB NVMe SSD | Model: [CRUCIAL P310] |
| **Network** | 1x 2.5GbE RJ45 | Single NIC with VLAN tagging |
| **Operating System** | Proxmox VE 9.1 |


**BIOS Settings:**
- Virtualization (VT-x/VT-d): Enabled
- Boot Mode: UEFI
- Secure Boot: Disabled (for Proxmox compatibility)

**Network Configuration:**
- Physical NIC: Connected to Cisco SG300 trunk port
- Bridge: `vmbr0` (VLAN-aware)
- Management IP: 192.168.10.5/24 (VLAN 10)

**Current VM Load:**
- Running VMs: [7]
- Total vCPUs allocated: [14]
- Total RAM allocated: [42]GB


---

## 2. Firewall/Router Appliance

### AOOSTAR N150 Mini PC

**Purpose:** Dedicated OPNsense firewall and inter-VLAN router

| Component | Specification | Notes |
|-----------|--------------|-------|
| **Model** | AOOSTAR N150 |
| **CPU** | Intel N150 | Alder Lake-N, 4 cores |
| **RAM** | [12]GB DDR4 | Sufficient for OPNsense + plugins |
| **Storage** | [256]GB SSD/eMMC | OS and config storage |
| **Network - WAN** | 1x 2.5GbE RJ45 | Connected to ISP router |
| **Network - LAN** | 1x 2.5GbE RJ45 | Trunk port to Cisco switch (VLANs 10,20,30,40) |
| **Operating System** | OPNsense 24.x | Open-source firewall |

**OPNsense Configuration:**
- WAN Interface: `igc0` - DHCP from ISP router
- LAN Interface: `igc1` - VLAN trunk
  - VLAN 10: 192.168.10.1/24 (Management)
  - VLAN 20: 192.168.20.1/24 (Servers)
  - VLAN 30: 192.168.30.1/24 (Trusted Clients)
  - VLAN 40: 192.168.40.1/24 (Attacker/DMZ)

**Performance Notes:**
- Throughput: ~2.5Gbps with basic firewall rules
- Latency: <1ms inter-VLAN routing
- CPU usage: <20% under normal load

---

## 3. Network Switch

### Cisco SG300-28 (SG300-28P)

**Purpose:** Managed Layer 2+ switch with VLAN support

| Component | Specification | Notes |
|-----------|--------------|-------|
| **Model** | Cisco SG300-28(P) | 28-port managed switch |
| **Ports** | 24x 1GbE + 2x combo GbE/SFP | RJ45 ports |
| **Management** | Web UI, CLI (SSH) | Static IP: 192.168.10.254 (VLAN 10) |
| **Firmware** | [version] |
| **VLAN Support** | 802.1Q, 4096 VLANs | Using VLANs 10, 20, 30, 40 |




**VLAN Configuration:**
```
VLAN 10 (Management): Untagged on port 24, Tagged on ports 2,3
VLAN 20 (Servers): Tagged on ports 2,3
VLAN 30 (Clients): Untagged on ports 4-10, Tagged on ports 2,3
VLAN 40 (Attacker): Untagged on ports 11-15, Tagged on ports 2,3
```

---

## 4. ISP Router/Modem

### [Your ISP Router Model]

**Purpose:** Primary internet gateway (creates double-NAT isolation for lab)

| Component | Specification | Notes |
|-----------|--------------|-------|
| **ISP** | [TPG] | [Speed: 100 Mbps down / 18 Mbps up] |
| **Model** | [sagemcom fast 5866tl] | ISP-provided |
| **WAN IP** | Public (dynamic) | From ISP |
| **LAN Subnet** | [192.168.1.0/24] | Home network |


**Isolation Strategy:**
- Lab network (192.168.x.x) sits behind ISP router
- Double-NAT provides complete isolation
- Home devices cannot reach lab VMs
- Lab VMs have internet access through OPNsense → ISP router

---


## 5. Lessons Learned & Recommendations

### What Works Well

- **Single NIC on Proxmox:** VLAN tagging works perfectly, no need for multiple NICs
- **Dedicated firewall appliance:** Better performance than virtualized OPNsense
- **Cisco SG300:** Enterprise features at prosumer price

### Challenges Overcome

1. VLAN Tagging on Single NIC Proxmox Host
**Problem:** Initial confusion on how to trunk multiple VLANs over a single physical interface.  
**Solution:** Enabled "VLAN aware" checkbox on `vmbr0` bridge in Proxmox network settings. Then configured VLAN tags directly on VM network interfaces (e.g., `vmbr0` with VLAN tag 20 for server VMs). 
2. OPNsense Firewall Rules - Internet Access Without Inter-VLAN Communication
**Problem:** VMs needed internet access but should NOT be able to communicate across VLANs freely.  
**Solution:** Implemented hierarchical firewall rules on OPNsense:
- **Allow rules first:** Each VLAN → !interal_networks(VLAN 10 - VLAN 40) (permit HTTP/HTTPS)
- **Explicit allow rules last:** Specific inter-VLAN where needed (e.g., VLAN 30 → VLAN 20 essential ports for AD services)
- **Default deny:** Implicit deny all at bottom of rule stack  
3. DNS Resolution Across VLANs
**Problem:** VMs in VLAN 30 couldn't resolve internal domain names served by AD DC (192.168.20.10).  
**Solution:** 
- Configured OPNsense DHCP to push DNS server 192.168.20.10 to VLAN 30 clients
- Added firewall rule allowing UDP/TCP 53 from all VLANs → 192.168.20.10
- Configured AD DNS server to forward external queries to 8.8.8.8  
4. Windows 11 Domain Join Over VLAN
**Problem:** Windows 11 VMs in VLAN 30 couldn't find domain controller in VLAN 20.  
**Solution:** 
- Verified DNS settings pointed to AD DC (192.168.20.10)
- Confirmed firewall rules allowed necessary AD ports (TCP/UDP 88, 135, 389, 445, 464, etc.)
- Used `nslookup` and to verify connectivity before domain join
- Successfully joined after confirming all requirements   

|

---



## Resources & References

- [Proxmox Documentation](https://pve.proxmox.com/wiki/Main_Page)
- [OPNsense Documentation](https://docs.opnsense.org/)
- [Cisco SG300 Admin Guide](https://www.cisco.com/c/en/us/support/switches/small-business-300-series-managed-switches/products-user-guide-list.html)
- [VLAN Configuration Guide](https://link-to-your-guide)

---

**Author:** Sheng-You Chen  
**Contact:** jack9789@gmail.com  
**GitHub:** https://github.com/jack9789/proxmox-opnsense-vlan-lab
