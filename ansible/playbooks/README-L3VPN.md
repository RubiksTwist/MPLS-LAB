# L3VPN (VRF) Playbook (`configure_l3vpn.yml`)

This playbook configures the L3VPN VRF (Virtual Routing and Forwarding) instance on the Provider Edge (PE) routers. It sets up VRF A, assigns route-distinguisher and route-target, and enables BGP VPN import/export for MPLS L3VPN services.

## Background

**L3VPNs** traverse an MPLS environment to communicate at Layer 3 between PE routers. As explained in the [reference guide](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/), L3VPNs introduce a **VPN label** (inner/bottom-of-stack label) in addition to the transport label. This creates a **dual-label stack**:

1. **Transport Label (outer)** - Routes packets across the MPLS core
2. **VPN Label (inner)** - Identifies which VRF to perform the IP lookup in

**Important:** L3VPNs provide traffic segmentation but **NOT encryption**. All traffic traverses the provider network unencrypted unless additional encryption is added.

### VRF Components Explained

**VRF Table:**
- Maps VRF to Linux routing table (101 in this lab)
- Allows overlapping IP spaces across different customers

**Import/Export VPN:**
- `export vpn` - Advertises local VRF routes to VPNv4 address-family
- `import vpn` - Accepts VPNv4 routes into local VRF

**Label Assignment:**
- `auto` - Zebra Routing Daemon automatically assigns VPN labels
- Alternative: Manually specify labels for deterministic behavior

**Route Distinguisher (RD) - 65000:1:**
- Makes identical prefixes unique across VRFs
- Converts `10.0.0.0/24` to `65000:1:10.0.0.0/24`
- Format: `ASN:number` or `IP:number`
- Only needs to be unique per PE router

**Route Target (RT) - 65000:1:**
- Attached as BGP extended-community to VPNv4 routes
- Controls which VRFs can import specific routes
- `both` = import AND export this RT
- Enables route leaking by importing multiple RTs

## What it does
- Cleans up old VRF configuration
- Creates VRF A (table 101) on PE1 and PE2
- Configures BGP VPN import/export, label assignment, RD, and RT
- Commits and saves configuration
- Verifies VRF, VRF routing table, and BGP VRF summary

## Usage
```bash
ansible-playbook -i inventories/hosts.ini playbooks/configure_l3vpn.yml
```

## Verification
- VRF A should exist on both PE routers
- VRF routing table should show customer and core routes
- Use `show vrf`, `show ip route vrf A`, and `show bgp vrf A summary` for validation

**Example VPNv4 Route with Labels:**
```
vyos@PE1# run show bgp ipv4 vpn 10.0.2.0/24
BGP routing table entry for 172.16.0.5:2:10.0.2.0/24, version 14
  Extended Community: RT:65000:1
  Remote label: 144
```

**Dual-Label Stack in Action:**
```
vyos@PE1# run show ip route vrf A
VRF A:
B>  10.0.2.0/24 [200/0] via 10.0.0.4 (vrf default) (recursive), label 144
 *                       via 10.1.2.2, eth0 (vrf default), label 18/144
```
- Label 18 = Transport label (MPLS core path to PE2)
- Label 144 = VPN label (identifies VRF A on PE2)

---
**Reference:** Based on [MPLS on VyOS – L3VPN](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/) guide.

Edit this file if you change VRF names, RD/RT values, or add new VRFs.
