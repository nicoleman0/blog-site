---
title: "Building a Segmented Office Network with VLANs and Router-on-a-Stick"
date: 2026-02-08
draft: false
description: "Strengthening my networking skills"
tags: ["networks", "network+", "cisco-packet-tracer"]
categories: ["general"]
toc: false
---

**Project Date:** February 8, 2026  
**Tools Used:** Cisco Packet Tracer  
**Difficulty:** Intermediate

I designed and configured a small office network for a fictional company using enterprise networking concepts. The network implements VLAN segmentation to separate different departments, with centralized routing to enable controlled communication between them.

## The Challenge

Modern networks need to balance two competing requirements:

- **Security**: Different departments shouldn't all share the same broadcast domain
- **Connectivity**: Users still need to communicate across departments when necessary

The solution are VLANs (Virtual Local Area Networks) combined with inter-VLAN routing.

## Network Architecture

### Hardware Components

- 1x Cisco 2911 Router
- 2x Cisco 2960-24TT Switches
- 1x Server (Management)
- 6x PCs (distributed across departments)
- 1x Wireless Access Point
- 1x Laptop (guest device)

### VLAN Design

I created four separate VLANs to segment the network:

| VLAN | Purpose | Subnet | Devices |
|------|---------|--------|---------|
| 10 | Management | 10.0.10.0/24 | Server |
| 20 | Finance | 10.0.20.0/24 | 2 PCs |
| 30 | Staff | 10.0.30.0/24 | 4 PCs |
| 40 | Guest | 10.0.40.0/24 | Wireless laptop |

Each VLAN operates as its own isolated network segment, preventing broadcast traffic from flooding the entire network and limiting lateral movement in case of a security incident.

## Key Technical Concepts

### Router-on-a-Stick

Rather than using multiple physical router interfaces (one per VLAN), I implemented router-on-a-stick. This uses a single physical connection with multiple logical subinterfaces:

```bash
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 10.0.10.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 10.0.20.1 255.255.255.0
```

Each subinterface acts as the default gateway for its respective VLAN, enabling inter-VLAN routing through a single trunk connection.

### Trunk Ports

Trunk ports carry traffic for multiple VLANs simultaneously, tagged with 802.1Q VLAN IDs. I configured trunks between:

- Router → SW1 (carries all four VLANs)
- SW1 → SW2 (inter-switch communication)

### Access Ports

End-user devices connect to access ports, which are assigned to a single VLAN:

```bash
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
```

This ensures that when Server1 sends traffic, it's automatically tagged as VLAN 10 traffic.

## Security Implementation

### Guest Network Isolation

I implemented Access Control List (ACL) 100 to prevent guest devices from accessing internal resources:

```bash
access-list 100 deny ip 10.0.40.0 0.0.0.255 10.0.10.0 0.0.0.255
access-list 100 deny ip 10.0.40.0 0.0.0.255 10.0.20.0 0.0.0.255
access-list 100 deny ip 10.0.40.0 0.0.0.255 10.0.30.0 0.0.0.255
access-list 100 permit ip any any
!
interface GigabitEthernet0/0.40
 ip access-group 100 in
```

This creates a unidirectional barrier. Guests cannot initiate connections to internal networks, but internal users can still reach guest devices for troubleshooting.

## Testing and Validation

I verified the network functionality through systematic testing:

- **Intra-VLAN communication** (same switch): PC1 → PC2 (both VLAN 30)  
- **Cross-switch VLAN communication**: PC1 (SW1) → PC3 (SW2)  
- **Inter-VLAN routing**: PC1 (VLAN 30) → PC0 (VLAN 20)  
- **Gateway reachability**: All devices can reach their default gateways  
- **ACL enforcement**: Guest laptop blocked from accessing 10.0.30.11  

All tests passed, confirming proper VLAN segmentation, trunk operation, and routing functionality.

## Things I learned

**VLANs aren't just about organization** – they're a fundamental security control. By placing Finance in VLAN 20 and Staff in VLAN 30, I created separate broadcast domains that prevent casual network sniffing and limit the blast radius of network attacks.

**Router-on-a-stick is efficient but creates a bottleneck** – All inter-VLAN traffic must traverse the single uplink to the router. For larger networks, Layer 3 switches with Switched Virtual Interfaces (SVIs) would be more scalable.

**ACLs require careful planning** – The order matters (evaluated top-to-bottom), and implicit deny-all means you must explicitly permit legitimate traffic. My ACL 100 blocks specific source/destination pairs but permits everything else.

This project reinforced fundamental enterprise networking concepts: VLAN segmentation, trunk configuration, inter-VLAN routing, and access control. While built in a simulator, these same principles apply to production networks supporting thousands of users.

---

**Technologies:** Cisco IOS, VLANs, 802.1Q Trunking, Router-on-a-Stick, Access Control Lists  
**Skills Demonstrated:** Network design, router configuration, switch configuration, security implementation