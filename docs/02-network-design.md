# Network Design Documentation

## Overview

This document details the network architecture, VLAN segmentation, routing, and security design of the Proxmox OPNsense VLAN Homelab. The design follows enterprise best practices for network segmentation, defense-in-depth security, and scalable infrastructure.

**Last Updated:** January 2026  
**Network Architecture:** Multi-VLAN with centralized firewall routing  
**Security Model:** Zero-trust between VLANs, explicit allow rules only  

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Network Topology](#network-topology)
3. [VLAN Architecture](#vlan-architecture)
4. [IP Addressing Scheme](#ip-addressing-scheme)
5. [Routing & Gateway Configuration](#routing--gateway-configuration)
6. [Firewall Policy Design](#firewall-policy-design)
7. [DNS & DHCP Services](#dns--dhcp-services)
8. [Security Zones & Trust Boundaries](#security-zones--trust-boundaries)
9. [Traffic Flow Examples](#traffic-flow-examples)
10. [High Availability Considerations](#high-availability-considerations)
11. [Future Expansion Plan](#future-expansion-plan)

---

## Design Philosophy

### Core Principles

1. **Defense in Depth**: Multiple layers of security (network segmentation, firewall rules, host-based security)
2. **Least Privilege**: Services only communicate where business logic requires it
3. **Isolation by Default**: VLANs cannot communicate unless explicitly permitted
4. **Centralized Management**: Single firewall point for all inter-VLAN routing and policy enforcement
5. **Double-NAT Isolation**: Lab completely isolated from home network for safety and testing

### Design Goals

- ✅ Simulate enterprise network segmentation
- ✅ Practice secure firewall rule management
- ✅ Enable safe security testing (isolated attacker VLAN)
- ✅ Support Active Directory multi-subnet environment
- ✅ Provide foundation for advanced scenarios (SIEM, monitoring, IDS/IPS)
- ✅ Maintain cost-effectiveness (single NIC on Proxmox, prosumer hardware)

---

## Network Topology

### Physical Topology

```
                    INTERNET
                        │
                        ↓
        ┌───────────────────────────┐
        │   ISP Router/Modem        │
        │   192.168.1.0/24          │
        │   (Home Network)          │
        └───────────┬───────────────┘
                    │ WAN (DHCP)
                    ↓
        ┌───────────────────────────┐
        │   OPNsense Firewall       │
        │   AOOSTAR N150            │
        │   - igc1: WAN (uplink)    │
        │   - igc0: LAN (trunk)     │
        └───────────┬───────────────┘
                    │ Trunk (VLANs 10,20,30,40)
                    ↓
        ┌───────────────────────────┐
        │   Cisco SG300-28 Switch   │
        │   Managed L2+ Switch      │
        └─────┬─────────────────┬───┘
              │                 │
              │ Trunk           │ Trunk
              │ (all VLANs)     │ (all VLANs)
              ↓                 ↓
    ┌─────────────────┐   [Access Ports]
    │ Proxmox Host    │   - VLAN 10 (Mgmt)
    │ ASUS NUC14 Pro  │   - VLAN 20 (Servers)
    │ - vmbr0 (VLAN-  │   - VLAN 30 (Clients)
    │   aware bridge) │   - VLAN 40 (Attacker)
    └─────────────────┘
```

### Logical Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     VLAN 10 - Management                    │
│  192.168.10.0/24                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  OPNsense    │  │  Proxmox     │  │  Cisco SG300 │     │
│  │  .10.1 (GW)  │  │  .10.5       │  │  .10.254       │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     VLAN 20 - Servers                       │
│  192.168.20.0/24                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ OPNsense GW  │  │ Windows DC   │  │ AlmaLinux    │     │
│  │ .20.1        │  │ .20.10 (AD)  │  │ .20.20 (Docker)   │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                     DNS, DHCP, AD      Portainer, Nextcloud │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                  VLAN 30 - Trusted Clients                  │
│  192.168.30.0/24                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ OPNsense GW  │  │ Windows 11   │  │ Ubuntu       │     │
│  │ .30.1        │  │ DHCP         │  │ Desktop DHCP │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                     Domain-joined     Domain-joined        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   VLAN 40 - Attacker/DMZ                    │
│  192.168.40.0/24                                            │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ OPNsense GW  │  │ Kali Linux   │                        │
│  │ .40.1        │  │ .40.10       │                        │
│  └──────────────┘  └──────────────┘                        │
│                     Penetration testing                     │
└─────────────────────────────────────────────────────────────┘
```

---

## VLAN Architecture

### VLAN Segmentation Strategy

| VLAN ID | Name | Purpose | Subnet | Security Zone | Trust Level |
|---------|------|---------|--------|---------------|-------------|
| **10** | Management | Infrastructure management (Proxmox, switch, OPNsense) | 192.168.10.0/24 | Trusted | High |
| **20** | Servers | Production servers, Active Directory, Docker services | 192.168.20.0/24 | Semi-Trusted | Medium |
| **30** | Clients | End-user workstations and devices | 192.168.30.0/24 | Trusted | Medium |
| **40** | Attacker/DMZ | Security testing, isolated lab VMs | 192.168.40.0/24 | Untrusted | Low |

### VLAN Selection Rationale

**VLAN 10 (Management):**
- Houses critical infrastructure components
- Direct management access to all devices
- Highest security controls
- Minimal surface area (only admin access)

**VLAN 20 (Servers):**
- Centralized services that other VLANs need (AD, DNS, DHCP)
- Docker containerized applications
- Controlled access from client VLANs
- No direct access from Attacker VLAN

**VLAN 30 (Clients):**
- User workstations and testing machines
- Can access servers for legitimate services (AD, file shares, web apps)
- Cannot access management VLAN (except through controlled access)
- Simulates corporate user network

**VLAN 40 (Attacker/DMZ):**
- Completely isolated by default
- Internet access only
- Used for penetration testing and malware analysis
- Temporary firewall rules created for specific testing scenarios

### VLAN Assignment on Cisco SG300

```
Port Configuration:

```

---

## IP Addressing Scheme

### Address Allocation Strategy

**Private RFC1918 Addressing:**
- Using Class C private ranges (192.168.x.0/24)
- /24 subnets provide 254 usable hosts each (sufficient for homelab)
- Consistent .1 gateway addressing across all VLANs

### VLAN 10 - Management (192.168.10.0/24)

| IP Address | Device/Service | Type | Notes |
|------------|----------------|------|-------|
| 192.168.10.1 | OPNsense VLAN 10 Interface | Static | Default gateway |
| 192.168.10.254 | Cisco SG300-28 Management | Static | Switch web UI/SSH |
| 192.168.10.5 | Proxmox VE Host | Static | Hypervisor management |
| 192.168.10.40 | Proxmox Backup Server | Static | Backup Server |


### VLAN 20 - Servers (192.168.20.0/24)

| IP Address | Device/Service | Type | Notes |
|------------|----------------|------|-------|
| 192.168.20.1 | OPNsense VLAN 20 Interface | Static | Default gateway |
| 192.168.20.10 | Windows Server 2022 (AD DC) | Static | Primary DC, DNS, DHCP |
| 192.168.20.20 | AlmaLinux Docker Host | Static | Container services |


**Docker Services on 192.168.20.20:**
- Portainer: Port 9000
- Zammad: Port 8080
- SearXNG: Port 8001
- AnythingLLM: Port 3001
- NetData: Port 19999
- Nginx Proxy Manager: Port 81

### VLAN 30 - Clients (192.168.30.0/24)

| IP Address | Device/Service | Type | Notes |
|------------|----------------|------|-------|
| 192.168.30.1 | OPNsense VLAN 30 Interface | Static | Default gateway |
| 192.168.30.10-50 | Static client assignments | Static | Admin workstations |
| 192.168.30.100-200 | DHCP scope (AD DC) | DHCP | Workstations, test VMs |


**DHCP Scope Details:**
- Scope: 192.168.30.100 - 192.168.30.200
- Gateway: 192.168.30.1 (OPNsense)
- DNS: 192.168.20.10 (AD DC)
- Lease time: 8 hours
- Managed by: OPNsense

### VLAN 40 - Attacker/DMZ (192.168.40.0/24)

| IP Address | Device/Service | Type | Notes |
|------------|----------------|------|-------|
| 192.168.40.1 | OPNsense VLAN 40 Interface | Static | Default gateway |
| 192.168.40.10 | Kali Linux | Static | Primary pentesting VM |



### Subnet Summary Table

| VLAN | Network | Usable IPs | Gateway | Broadcast | DHCP Server | DHCP Range |
|------|---------|------------|---------|-----------|-------------|------------|
| 10 | 192.168.10.0/24 | 192.168.10.1-254 | .1 | .255 | N/A | Static only |
| 20 | 192.168.20.0/24 | 192.168.20.1-254 | .1 | .255 | N/A | Static only|
| 30 | 192.168.30.0/24 | 192.168.30.1-254 | .1 | .255 | OPNsense (.1) | .100-.200 |
| 40 | 192.168.40.0/24 | 192.168.40.1-254 | .1 | .255 | OPNsense (.1) | Static only |

---

## Routing & Gateway Configuration

### Default Gateway Architecture

**All VLANs use OPNsense as default gateway:**
- VLAN 10: 192.168.10.1
- VLAN 20: 192.168.20.1
- VLAN 30: 192.168.30.1
- VLAN 40: 192.168.40.1

**OPNsense acts as router for:**
- Inter-VLAN routing (controlled by firewall rules)
- Internet access (NAT to WAN interface)
- DNS forwarding (conditional forwarding to AD DC)



### NAT Configuration

**Double-NAT Architecture:**

```
VM (192.168.20.10) → Internet:

Step 1: VM source NAT by OPNsense
  Source: 192.168.20.10
  Destination: 8.8.8.8
          ↓ [OPNsense NAT]
  Source: 192.168.1.X (OPNsense WAN IP from ISP router)
  Destination: 8.8.8.8

Step 2: OPNsense WAN NAT by ISP Router
  Source: 192.168.1.X
  Destination: 8.8.8.8
          ↓ [ISP Router NAT]
  Source: [Public IP]
  Destination: 8.8.8.8
```

**Benefits of Double-NAT in this context:**
- Complete isolation from home network
- Lab traffic doesn't interfere with home devices
- Additional security layer
- Can be modified/destroyed without affecting home network

**Outbound NAT Rules (OPNsense):**
- VLAN 10 → WAN: Allowed (management needs internet)
- VLAN 20 → WAN: Allowed (servers need updates)
- VLAN 30 → WAN: Allowed (clients need internet)
- VLAN 40 → WAN: Allowed (attacker needs internet for tools)

---


## Conclusion

This network design provides a realistic simulation of enterprise network architecture with proper segmentation, security controls, and centralized services. The modular VLAN approach allows for easy expansion and testing of new scenarios while maintaining security isolation.

**Key Achievements:**
- ✅ Four isolated VLANs with controlled inter-VLAN routing
- ✅ Enterprise-style Active Directory domain
- ✅ Secure firewall policy with default-deny approach
- ✅ Production-ready services (Docker containers, AD, DNS, DHCP)
- ✅ Isolated penetration testing environment
- ✅ Comprehensive documentation for portfolio

**This design demonstrates:**
- Network segmentation best practices
- Firewall policy development and management
- Multi-subnet Active Directory implementation  
- Enterprise service integration (DNS, DHCP, AD)
- Security-first thinking (defense in depth, least privilege)
- Practical troubleshooting and problem-solving skills

---



**Author:** Sheng-You Chen  
**Contact:** jack9789@gmail.com  
**GitHub:** https://github.com/jack9789/proxmox-opnsense-vlan-lab
