# MPLS L3VPN Configuration Reference

**Source:** [MPLS on VyOS – L3VPN](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/) by R C

This document serves as a reference for configuring MPLS L3VPNs on VyOS based on a proven working example.

---

## Overview

L3VPNs traverse an MPLS environment to communicate at Layer 3 between Provider Edge (PE) routers. They use **two labels**:

1. **Transport Label** (outer/top of stack) - Used by MPLS core for forwarding
2. **VPN Label** (inner/bottom of stack) - Maps to VRF/service on PE router

### Key Concepts

- **VRF (Virtual Routing and Forwarding)**: Separate routing tables per customer
- **Route Distinguisher (RD)**: Makes overlapping customer prefixes unique in MP-BGP
- **Route Target (RT)**: Controls which routes are imported/exported between VRFs
- **MP-BGP VPNv4**: Carries customer routes with VPN labels between PE routers
- **Label Stacking**: VPN label + Transport label enable end-to-end forwarding

---

## Network Topology (Reference Example)

```
CE1 ---- PE1 ---- P1 ---- P2 ---- PE2 ---- CE2
         |         |       |       |
      VRF A     OSPF+LDP OSPF+LDP VRF A
```

### IP Addressing Scheme (Reference)

- **Loopbacks**: 10.0.0.X/32 (where X = device number 1-4)
- **Links**: 10.X.Y.0/24 (where X and Y are device numbers)
  - PE1-P1: 10.1.2.0/24
  - P1-P2: 10.2.3.0/24
  - P2-PE2: 10.3.4.0/24

---

## Configuration Steps

### 1. OSPF Configuration (Underlay)

**Purpose**: Provide IP reachability between all LSRs (Label Switched Routers)

**On all Provider routers (PE1, P1, P2, PE2):**

```bash
# PE1 Example
set interfaces dummy dum0 address '10.0.0.1/32'
set interfaces ethernet eth0 address '10.1.2.1/24'

set protocols ospf interface dum0 area '0'
set protocols ospf interface eth0 area '0'
set protocols ospf interface eth0 network 'point-to-point'
```

**Key Points:**
- Use loopback/dummy interfaces for router IDs
- Configure all provider-facing interfaces in OSPF area 0
- Set network type to `point-to-point` on physical links
- Verify: `show ip route ospf` should show all loopbacks

---

### 2. MPLS LDP Configuration (Label Distribution)

**Purpose**: Distribute labels for IGP routes (creates transport labels)

**On all Provider routers:**

```bash
# PE1 Example
set protocols mpls interface 'eth0'
set protocols mpls ldp discovery transport-ipv4-address '10.0.0.1'
set protocols mpls ldp interface 'eth0'
set protocols mpls ldp router-id '10.0.0.1'
```

**Key Points:**
- Enable MPLS on all provider-facing interfaces
- Use loopback address for LDP router-id and transport address
- LDP automatically creates labels for OSPF-learned routes
- Verify: `show mpls ldp neighbor` (should show OPERATIONAL)
- Verify: `show mpls ldp binding` (should show labels for all loopbacks)

---

### 3. VRF Configuration (Customer Isolation)

**Purpose**: Create separate routing tables for each customer/service

**On PE routers:**

```bash
# VRF Creation
set vrf name A table '101'

# BGP in VRF
set vrf name A protocols bgp system-as '65000'

# CRITICAL: Enable VPN export/import
set vrf name A protocols bgp address-family ipv4-unicast export vpn
set vrf name A protocols bgp address-family ipv4-unicast import vpn

# VPN Label Allocation
set vrf name A protocols bgp address-family ipv4-unicast label vpn export 'auto'

# Route Distinguisher (makes prefixes unique)
set vrf name A protocols bgp address-family ipv4-unicast rd vpn export '65000:1'

# Route Target (controls import/export)
set vrf name A protocols bgp address-family ipv4-unicast route-target vpn both '65000:1'
```

**Key Points:**
- `export vpn` - Exports VRF routes to global VPNv4 table
- `import vpn` - Imports VPNv4 routes into VRF table
- `label vpn export 'auto'` - Automatically allocate VPN labels
- RD format: ASN:number or IP:number (must be unique per VRF)
- RT can use `both` for symmetric import/export

---

### 4. MP-BGP VPNv4 Configuration (Route Exchange)

**Purpose**: Exchange customer routes with VPN labels between PE routers

**On PE routers:**

```bash
# PE1 Configuration
set protocols bgp address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.4 address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.4 remote-as '65000'
set protocols bgp neighbor 10.0.0.4 update-source 'dum0'
set protocols bgp system-as '65000'

# PE2 Configuration (mirror)
set protocols bgp address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.1 address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.1 remote-as '65000'
set protocols bgp neighbor 10.0.0.1 update-source 'dum0'
set protocols bgp system-as '65000'
```

**Key Points:**
- Peer using loopback addresses (reachable via OSPF+LDP)
- Use `ipv4-vpn` address family (NOT ipv4-unicast)
- `update-source` ensures packets come from loopback
- This is iBGP (same AS on both sides)
- Verify: `show ip bgp summary` (look for IPv4 VPN Summary)

---

### 5. PE-to-CE BGP Configuration

**Purpose**: Learn customer routes via eBGP

**On PE routers:**

```bash
# PE1 to CE1
set interfaces ethernet eth2 address '172.16.0.1/30'
set interfaces ethernet eth2 vrf 'A'

set vrf name A protocols bgp address-family ipv4-unicast network 172.16.0.0/30
set vrf name A protocols bgp neighbor 172.16.0.2 address-family ipv4-unicast
set vrf name A protocols bgp neighbor 172.16.0.2 remote-as '65001'
```

**Key Points:**
- Assign customer-facing interface to VRF
- Configure eBGP in VRF context
- Advertise PE-CE subnet in VRF BGP
- Customer routes learned here are auto-exported to VPNv4 (via `export vpn`)

---

## Verification Commands

### Control Plane

```bash
# OSPF
show ip route ospf                    # Should see all loopbacks
show ip ospf neighbor                  # Should see Full state

# LDP
show mpls ldp neighbor                 # Should be OPERATIONAL
show mpls ldp binding                  # Should show labels for all loopbacks

# MP-BGP VPNv4
show ip bgp summary                    # Look for "IPv4 VPN Summary"
show bgp ipv4 vpn                      # Shows VPNv4 routes with RD and labels
show bgp ipv4 vpn 10.0.2.0/24         # Details for specific prefix

# VRF Routes
show ip route vrf A                    # Shows routes in VRF A
show ip bgp vrf A                      # Shows BGP table for VRF A
show vrf                               # Lists all VRFs and member interfaces
```

### Data Plane

```bash
# MPLS Forwarding
show mpls table                        # Local MPLS forwarding table
show mpls ldp binding <prefix>         # Label bindings for specific prefix

# Label Path
traceroute mpls ipv4 <destination>     # Trace MPLS path

# Test Connectivity
ping <remote-customer-ip> source-address <local-customer-ip>
```

---

## Packet Flow Example

**Customer traffic from CE1 to CE2:**

1. **CE1 → PE1**: Plain IP packet (10.0.1.1 → 10.0.2.1)
2. **PE1 lookup**: VRF A routing table shows route via VPNv4
3. **PE1 push labels**: 
   - VPN label (inner): 144 (from VPNv4 BGP)
   - Transport label (outer): 18 (from LDP for PE2 loopback)
4. **P1 swap label**: Swaps transport label 18 → 18 (locally significant)
5. **P2 pop label**: PHP (Penultimate Hop Popping) removes transport label
6. **PE2 with VPN label**: Only VPN label 144 remains
7. **PE2 lookup**: Label 144 maps to VRF A
8. **PE2 → CE2**: Plain IP packet forwarded in VRF A context

---

## Common Issues & Troubleshooting

### MP-BGP Won't Establish

**Symptoms**: BGP session stuck in "Active" or "Connect"

**Solutions**:
- Verify loopbacks are in OSPF: `show ip route 10.0.0.X`
- Check LDP is operational: `show mpls ldp neighbor`
- Verify `update-source` is set to loopback
- Check firewall rules (TCP 179)

### VPN Routes Not Exchanged

**Symptoms**: `show ip bgp summary` shows 0 prefixes received

**Solutions**:
- Verify `export vpn` is configured in VRF
- Verify `import vpn` is configured in VRF
- Check `label vpn export 'auto'` is set
- Verify RD and RT are configured
- Check: `show bgp ipv4 vpn` to see if routes exist

### No Customer Connectivity

**Symptoms**: Pings between customer sites fail

**Solutions**:
- Verify VRF routes exist: `show ip route vrf A`
- Check VPN labels allocated: `show mpls table`
- Verify PE-CE BGP: `show ip bgp vrf A summary`
- Check customer interfaces in correct VRF: `show vrf`
- Verify transport labels: `show mpls ldp binding`

---

## Architecture Notes

### Why P Routers Don't Need BGP

Core routers (P) only need:
- OSPF: To know how to reach PE loopbacks
- LDP: To distribute/swap transport labels

They never see customer routes or VPN labels. The VPN label is preserved through the core via label stacking.

### Label Stacking

```
[Ethernet][Transport Label: 18][VPN Label: 144][IP Packet]
           ^                    ^
           Swapped at P routers Preserved, popped at tail-end PE
```

### Route Distinguishers vs Route Targets

- **RD**: Makes routes unique (technical requirement)
  - Can be unique per VRF or per PE
  - Not used for routing decisions
  
- **RT**: Controls route distribution (policy tool)
  - Determines which routes go into which VRFs
  - Used for route leaking between VRFs

---

## Configuration Differences from Reference

### Our Topology Mapping

| Reference | Our Lab | Notes |
|-----------|---------|-------|
| PE1: 10.0.0.1 | PE1: 10.0.2.2 | Loopback addresses |
| P1: 10.0.0.2 | P1: 10.0.2.1 | Loopback addresses |
| P2: 10.0.0.3 | P2: 10.0.4.1 | Loopback addresses |
| PE2: 10.0.0.4 | PE2: 10.0.4.2 | Loopback addresses |
| Links: /24 | Links: /30 | Subnet sizes |
| CE1 AS: 65001 | CE1 AS: 65001 | Customer ASN (same) |
| CE2 AS: 65002 | CE2 AS: 65002 | Customer ASN (same) |
| VRF: A | VRF: A | VRF name (same) |
| RD: 65000:1 | RD: 65000:1 | Route distinguisher (same) |
| RT: 65000:1 | RT: 65000:1 | Route target (same) |

---

## Critical Configuration Requirements

**Must-have for L3VPN to work:**

1. ✅ OSPF on all provider interfaces (including loopbacks)
2. ✅ LDP on all provider interfaces
3. ✅ VRF created with table number
4. ✅ `export vpn` in VRF BGP address-family
5. ✅ `import vpn` in VRF BGP address-family
6. ✅ `label vpn export 'auto'` in VRF
7. ✅ RD configured in VRF
8. ✅ RT configured in VRF
9. ✅ Customer interface assigned to VRF
10. ✅ MP-BGP VPNv4 session between PEs
11. ✅ PE-CE BGP in VRF context

**Common mistakes:**

- ❌ Forgetting `export vpn` / `import vpn`
- ❌ Not advertising loopbacks in OSPF
- ❌ Using wrong address-family (ipv4-unicast instead of ipv4-vpn)
- ❌ Not setting `update-source` to loopback in MP-BGP
- ❌ Forgetting to assign interface to VRF

---

## Reference Links

- **Original Article**: https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/
- **VyOS Documentation**: https://docs.vyos.io/
- **MPLS Part 1** (Prerequisites): https://lev-0.com/2023/12/21/mpls-on-vyos/

---

## Additional Resources

### Video Walkthrough

The author has a YouTube video covering this configuration:
- Search for "Level Zero Networking MPLS L3VPN VyOS"

### Related Concepts

- **L2VPN**: VPLS/VPWS for Layer 2 services over MPLS
- **6PE/6VPE**: IPv6 over MPLS
- **EVPN**: Ethernet VPN (not yet supported in VyOS with MPLS dataplane)

---

*Last Updated: November 6, 2025*
*Based on: lev-0.com L3VPN article (January 18, 2024)*
