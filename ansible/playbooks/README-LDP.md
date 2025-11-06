# MPLS LDP Playbook (`configure_ldp.yml`)

This playbook configures MPLS Label Distribution Protocol (LDP) on all provider routers in the MPLS L3VPN lab. It ensures label switching is enabled on all core-facing interfaces and that LDP neighbors are established.

## Background

**Label Distribution Protocol (LDP)** is used for label imposition across the MPLS provider network. As explained in the [reference guide](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/), LDP creates the **transport labels** (outer/top-of-stack labels) that are:

- **Pushed** by the head-end PE router
- **Swapped** at every hop along the MPLS path
- **Popped** at the next-to-last hop (Penultimate Hop Popping - PHP)

LDP neighbors are discovered via UDP, but label mappings are exchanged over TCP sessions using the loopback addresses (configured via `transport-ipv4-address`). This ensures LDP sessions remain stable even if physical links fail (as long as an alternate path exists via OSPF).

**Key Concepts:**
- LDP assigns labels to OSPF-learned loopback prefixes
- Labels are **locally significant** (not globally unique)
- Implicit-null labels indicate PHP behavior

## What it does
- Cleans up old MPLS/LDP configuration
- Configures MPLS and LDP on all core-facing interfaces (eth1/eth2 as appropriate)
- Sets LDP router-ids to loopback addresses
- Commits and saves configuration
- Verifies LDP neighbors, bindings, and interfaces

## Usage
```bash
ansible-playbook -i inventories/hosts.ini playbooks/configure_ldp.yml
```

## Verification
- LDP neighbors should be established between all core routers
- MPLS labels should be assigned for all loopbacks
- Use `show mpls ldp neighbor` and `show mpls ldp binding` for validation

**Expected Output:**
```
vyos@PE1# run show mpls ldp binding
AF   Destination          Nexthop         Local Label Remote Label  In Use
ipv4 10.0.0.1/32          10.0.0.2        imp-null    16                no
ipv4 10.0.0.2/32          10.0.0.2        16          imp-null         yes
ipv4 10.0.0.3/32          10.0.0.2        17          17               yes
ipv4 10.0.0.4/32          10.0.0.2        18          18               yes
```

Note: `imp-null` (implicit-null) indicates PHP - the label is popped before reaching the destination.

---
**Reference:** Based on [MPLS on VyOS – L3VPN](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/) guide.

Edit this file if you change interface mappings or add new routers.
