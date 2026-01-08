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


## Firewall Policy Design

### Security Philosophy

**Defense-in-Depth Strategy:**
1. **Default Deny Approach** - All traffic blocked unless explicitly allowed
2. **Least Privilege Principle** - Grant minimum necessary access
3. **Network Segmentation** - VLANs isolated with controlled inter-VLAN routing
4. **Logging & Monitoring** - Track allowed and denied traffic for security analysis
5. **Port-Specific Rules** - Allow only required protocols/ports, not blanket "any"

### Firewall Architecture

**OPNsense acts as the central firewall and router:**
- All inter-VLAN traffic passes through OPNsense
- Stateful packet inspection with connection tracking
- Rules processed top-to-bottom (first match wins with "quick" flag)
- Aliases used for maintainability and clarity

### Firewall Aliases (Reusable Objects)

**Port Aliases:**

| Alias Name | Ports | Purpose |
|------------|-------|---------|
| `Web_Ports` | 80, 443 | Standard HTTP/HTTPS traffic |
| `Docker_APPs` | 80, 81, 3001, 8001, 8080, 9000, 19999 | Docker containerized services |
| `AD_Ports` | 53, 88, 135, 389, 445, 636, 464, 3268, 3269, 49152-65535 | Active Directory all required ports |

**Network Aliases:**

| Alias Name | Networks | Purpose |
|------------|----------|---------|
| `Internal_networks` | 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24, 192.168.40.0/24 | All lab VLAN subnets |

**Alias Usage Benefits:**
- Single point of change (modify alias, not individual rules)
- Improved readability (semantic names vs. port numbers)
- Reduced errors (consistency across rules)
- Easier auditing and documentation

---

### VLAN 10 (Management) - OPT1 Firewall Rules

**Security Level:** Trusted (Full Administrative Access)

| # | Action | Protocol | Source | Destination | Port/Service | Description |
|---|--------|----------|--------|-------------|--------------|-------------|
| 1 | **Pass** | TCP | VLAN 10 net | OPNsense (self) | 22 | SSH to OPNsense for CLI management |
| 2 | **Pass** | TCP | VLAN 10 net | OPNsense (self) | 443 | HTTPS to OPNsense Web UI |
| 3 | **Pass** | TCP/UDP | Any | OPNsense (self) | 123 | NTP time synchronization |
| 4 | **Pass** | TCP/UDP | VLAN 10 net | OPNsense (self) | 53 | DNS queries to OPNsense |
| 5 | **Pass** | TCP/UDP | VLAN 10 net | 192.168.20.20 | Docker_APPs | Access Docker services (Portainer, Nextcloud, etc.) |
| 6 | **Pass** | TCP | VLAN 10 net | !Internal_networks | Web_Ports | Internet access (HTTP/HTTPS) |
| 7 | **Pass** | ICMP | Any | Any | - | Ping for diagnostics |
| 8 | **Deny** | Any | Any | Any | Any | Implicit deny all (logged) |

**Rationale:**
- Management VLAN has privileged access to OPNsense itself (SSH, Web UI)
- Can access all services for administrative purposes
- Can reach internet for updates and management
- ICMP allowed for network diagnostics (ping, traceroute)
- Full access to Docker applications for monitoring/management

**Key Features:**
- Direct access to OPNsense management interfaces
- Access to servers for administration
- Outbound internet access unrestricted
- No restrictions on inter-VLAN access (trusted admin network)

---

### VLAN 20 (Servers) - OPT2 Firewall Rules

**Security Level:** Semi-Trusted (Production Services)

| # | Action | Protocol | Source | Destination | Port/Service | Description |
|---|--------|----------|--------|-------------|--------------|-------------|
| 1 | **Pass** | TCP | VLAN 20 net | OPNsense (self) | 443 | HTTPS to OPNsense (monitoring, management) |
| 2 | **Pass** | TCP/UDP | VLAN 20 net | OPNsense (self) | 53 | DNS queries for external resolution |
| 3 | **Pass** | TCP | VLAN 20 net | !Internal_networks | Web_Ports | Internet access for updates (HTTP/HTTPS) |
| 4 | **Pass** | ICMP | Any | Internal_networks | - | Ping to other VLANs for monitoring |
| 5 | **Deny** | Any | VLAN 20 net | VLAN 30 net | Any | Block servers from initiating to clients |
| 6 | **Deny** | Any | VLAN 20 net | VLAN 40 net | Any | Block servers from reaching attacker VLAN |
| 7 | **Deny** | Any | VLAN 20 net | VLAN 10 net | Any | Block servers from management VLAN |
| 8 | **Deny** | Any | Any | Any | Any | Implicit deny all (logged) |

**Rationale:**
- Servers can download updates from internet
- Servers cannot initiate connections to client or management VLANs (security)
- ICMP allowed for monitoring and diagnostics
- DNS resolution via OPNsense for external queries
- Limited access to OPNsense (HTTPS only, no SSH from servers)

**Security Principle:**
- Servers are "semi-trusted" - they provide services but should not initiate connections to clients
- Prevents lateral movement in case of server compromise
- Outbound internet access needed for updates and external services
- ICMP monitoring allowed for operational visibility

---

### VLAN 30 (Clients) - OPT3 Firewall Rules

**Security Level:** Trusted (User Workstations)

| # | Action | Protocol | Source | Destination | Port/Service | Description |
|---|--------|----------|--------|-------------|--------------|-------------|
| 1 | **Pass** | TCP/UDP | VLAN 30 net | 192.168.20.10 | AD_Ports | Active Directory authentication and services |
| 2 | **Pass** | TCP | VLAN 30 net | 192.168.20.20 | Docker_APPs | Access to Docker web services (Nextcloud, etc.) |
| 3 | **Pass** | TCP/UDP | VLAN 30 net | OPNsense (self) | 53 | DNS queries (forwarded to AD DC) |
| 4 | **Pass** | TCP | VLAN 30 net | !Internal_networks | Web_Ports | Internet access (HTTP/HTTPS) |
| 5 | **Pass** | ICMP | Any | Internal_networks | - | Ping for diagnostics |
| 6 | **Deny** | Any | VLAN 30 net | VLAN 10 net | Any | Block clients from management VLAN |
| 7 | **Deny** | Any | VLAN 30 net | VLAN 40 net | Any | Block clients from attacker VLAN |
| 8 | **Deny** | Any | Any | Any | Any | Implicit deny all (logged) |

**Rationale:**
- Clients need full access to Active Directory (authentication, GPOs, file shares)
- Clients can access Docker applications (Nextcloud, Portainer, etc.)
- Internet access for productivity (web browsing)
- DNS forwarded through OPNsense to AD DC
- Cannot access management VLAN (separation of duties)
- Cannot reach attacker VLAN (security isolation)

**Active Directory Ports Breakdown:**
- **53** (DNS): Name resolution
- **88** (Kerberos): Authentication
- **135** (RPC): Remote procedure calls
- **389** (LDAP): Directory queries
- **445** (SMB): File sharing, SYSVOL
- **464** (Kerberos Pwd): Password changes
- **636** (LDAPS): Secure LDAP
- **3268/3269** (Global Catalog): Forest-wide directory queries
- **49152-65535** (Dynamic RPC): Various AD services

**Docker Applications Accessible:**
- Port 80: HTTP services
- Port 81: Custom web app
- Port 3001: AnythingLLM
- Port 8001: Custom service
- Port 8080: Nextcloud
- Port 9000: Portainer
- Port 19999: NetData monitoring

---

### VLAN 40 (Attacker/DMZ) - OPT4 Firewall Rules

**Security Level:** Untrusted (Isolated Testing Environment)

| # | Action | Protocol | Source | Destination | Port/Service | Description |
|---|--------|----------|--------|-------------|--------------|-------------|
| 1 | **Pass** | TCP/UDP | Any | OPNsense (self) | 123 | NTP time synchronization |
| 2 | **Pass** | TCP/UDP | VLAN 40 net | OPNsense (self) | 53 | DNS queries for internet resolution |
| 3 | **Pass** | TCP | VLAN 40 net | !Internal_networks | Web_Ports | Internet-only access (tools, updates) |
| 4 | **Pass** | ICMP | Any | Internal_networks | - | Ping to other VLANs (diagnostics) |
| 5 | **Deny** | Any | VLAN 40 net | VLAN 10 net | Any | Block attacker from management |
| 6 | **Deny** | Any | VLAN 40 net | VLAN 20 net | Any | Block attacker from servers (by default) |
| 7 | **Deny** | Any | VLAN 40 net | VLAN 30 net | Any | Block attacker from clients |
| 8 | **Deny** | Any | Any | Any | Any | Implicit deny all (logged) |

**Rationale:**
- Complete isolation from all other VLANs by default
- Internet access only (for downloading penetration testing tools)
- ICMP allowed for network reconnaissance (intentional for testing)
- No access to management, servers, or clients without explicit temporary rules
- DNS queries allowed (attacker needs name resolution for internet)

**Security Design:**
- This VLAN is intentionally isolated for safe security testing
- Kali Linux and vulnerable VMs reside here
- Internet access allows downloading exploits, tools, updates
- Temporary rules created on-demand for specific testing scenarios
- All testing activity logged for analysis

**Temporary Testing Rules (Created as Needed):**
When conducting authorized penetration testing:
1. Admin logs into OPNsense
2. Creates temporary "Pass" rule: VLAN 40 → specific target IP/port
3. Performs testing (logged)
4. Disables rule immediately after completion
5. Reviews logs for detected activity

**Example Temporary Rule:**
```
Action: Pass
Source: 192.168.40.10 (Kali Linux)
Destination: 192.168.30.50 (Test Windows VM)
Protocol: Any
Duration: Enabled for testing session only
Purpose: Authorized vulnerability assessment
```

---

### Rule Processing Order & Logic

**OPNsense Rule Evaluation:**
1. **Top-to-bottom processing** - First matching rule wins (if "quick" flag set)
2. **Interface-specific** - Rules applied on ingress interface
3. **Stateful inspection** - Return traffic automatically allowed for established connections
4. **Implicit deny** - If no rule matches, traffic is blocked and logged

**"Quick" Flag Usage:**
- All rules use "quick" flag (first match wins, stop processing)
- Allows specific exceptions before general rules
- More efficient than evaluating entire ruleset

**Author:** Sheng-You Chen  
**Contact:** jack9789@gmail.com  
**GitHub:** https://github.com/jack9789/proxmox-opnsense-vlan-lab
