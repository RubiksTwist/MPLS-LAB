# OSPF Playbook (`configure_ospf_network.yml`)

This playbook configures OSPF on all core routers in the MPLS L3VPN lab. It sets up loopback (dummy) interfaces for router-IDs, assigns point-to-point OSPF interfaces, and ensures all provider routers (PE1, P1, P2, PE2) form OSPF adjacencies.

## Background

OSPF serves as the **Interior Gateway Protocol (IGP)** for the MPLS provider network. Like in the reference guide from [lev-0.com](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/), all **Label Switched Routers (LSR)** - including both P (Provider) and PE (Provider Edge) routers - need OSPF to:

- Establish reachability between all loopback addresses
- Provide the underlay routing for MPLS label distribution (LDP)
- Enable PE routers to reach each other for VPNv4 iBGP sessions

All MPLS devices are interconnected using IPs based on device numbers (e.g., PE1 and P1 use 10.1.2.0/24, P1 and P2 use 10.2.3.0/24).

**Network Design:**
- Loopbacks: PE1 (10.0.0.1/32), P1 (10.0.0.2/32), P2 (10.0.0.3/32), PE2 (10.0.0.4/32)
- All core links use point-to-point OSPF network type for faster convergence
- Area 0 (backbone) for all interfaces

## What it does
- Cleans up old OSPF and interface configs
- Configures loopback (dummy) interfaces for router-IDs
- Assigns OSPF to all core-facing interfaces (eth1/eth2 as appropriate)
- Sets point-to-point OSPF network type
- Commits and saves configuration
- Verifies OSPF neighbors, interfaces, and routes

## Usage
```bash
ansible-playbook -i inventories/hosts.ini playbooks/configure_ospf_network.yml
```

## Verification
- OSPF neighbors should be established between all core routers
- All loopbacks should be reachable via OSPF
- Use `show ip ospf neighbor` and `show ip route ospf` for validation

**Expected Output:**
```
vyos@PE1# run show ip route ospf
O   10.0.0.1/32 [110/1] via 0.0.0.0, dum0 onlink
O>* 10.0.0.2/32 [110/65536] via 10.1.2.2, eth2
O>* 10.0.0.3/32 [110/65537] via 10.1.2.2, eth2
O>* 10.0.0.4/32 [110/65538] via 10.1.2.2, eth2
```

---
**Reference:** Based on [MPLS on VyOS – L3VPN](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/) guide.

Edit this file if you change interface mappings or add new routers.
