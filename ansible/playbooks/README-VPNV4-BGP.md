# VPNv4 BGP Configuration Playbook

## Overview

This playbook configures VPNv4 (ipv4-vpn) iBGP peering between the Provider Edge (PE) routers in the MPLS network. VPNv4 BGP is essential for distributing customer VPN routes across the MPLS backbone.

## What is VPNv4?

VPNv4 (also called BGP/MPLS IP VPN) is a BGP address family used to carry customer IPv4 routes across an MPLS provider network while maintaining VPN isolation. Each route includes:
- **Route Distinguisher (RD)**: Makes customer routes globally unique
- **Route Target (RT)**: Controls import/export between VRFs
- **MPLS Label**: Used for forwarding traffic to the correct destination PE

## Why iBGP for VPNv4?

### Key Benefits:
1. **Next-hop preservation**: Routes maintain their original next-hop (originating PE router)
   - Traffic flows directly from ingress PE to egress PE
   - No unnecessary hops through intermediate routers

2. **Scalability with Route Reflectors**: 
   - Full mesh not required between all PEs
   - Route reflector clients can communicate directly
   - Hub-and-spoke traffic pattern avoided

3. **Stable peering**: 
   - Uses loopback addresses (dum0) as update-source
   - Session survives physical link failures (OSPF reroutes)

## Configuration Details

### PE1 Configuration
```bash
set protocols bgp address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.4 address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.4 remote-as '65000'
set protocols bgp neighbor 10.0.0.4 update-source 'dum0'
set protocols bgp system-as '65000'
```

### PE2 Configuration
```bash
set protocols bgp address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.1 address-family ipv4-vpn
set protocols bgp neighbor 10.0.0.1 remote-as '65000'
set protocols bgp neighbor 10.0.0.1 update-source 'dum0'
set protocols bgp system-as '65000'
```

### Configuration Breakdown

| Command | Purpose |
|---------|---------|
| `set protocols bgp address-family ipv4-vpn` | Enables VPNv4 address family in BGP |
| `set protocols bgp neighbor 10.0.0.X address-family ipv4-vpn` | Activates VPNv4 for the specific neighbor |
| `set protocols bgp neighbor 10.0.0.X remote-as '65000'` | Sets neighbor AS (same AS = iBGP) |
| `set protocols bgp neighbor 10.0.0.X update-source 'dum0'` | Uses loopback IP for BGP session |
| `set protocols bgp system-as '65000'` | Defines local AS number |

## Prerequisites

Before running this playbook, ensure the following are configured:

1. **OSPF** - For IGP reachability between loopbacks
   ```bash
   ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ospf_network.yml
   ```

2. **MPLS LDP** - For label distribution
   ```bash
   ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ldp.yml
   ```

3. **L3VPN (VRF)** - VRF A must exist on both PEs
   ```bash
   ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_l3vpn.yml
   ```

## Usage

### Run the Playbook
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_vpnv4_bgp.yml
```

### Expected Output
The playbook will:
1. Clean any existing BGP configuration
2. Configure VPNv4 iBGP between PE1 and PE2
3. Wait 15 seconds for BGP session establishment
4. Verify BGP peering status
5. Display VPNv4 routes (if any exist)

## Verification

### Check BGP Summary
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp summary'"
```

Expected: BGP neighbor should be in "Established" state

### Check VPNv4 Neighbors
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp ipv4 vpn summary'"
```

### Check VPNv4 Routes
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp ipv4 vpn'"
```

## Troubleshooting

### BGP Session Not Establishing

**Check OSPF reachability:**
```bash
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='ping 10.0.0.4 source-address 10.0.0.1'"
```

**Check LDP labels:**
```bash
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show mpls ldp binding'"
```
- Verify labels exist for 10.0.0.4/32

**Check BGP neighbor state:**
```bash
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp neighbor 10.0.0.4'"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| BGP session stuck in "Active" | No IP reachability | Verify OSPF and routes to loopbacks |
| BGP session stuck in "Idle" | Configuration mismatch | Check AS numbers match (65000) |
| No VPNv4 routes exchanged | VRF not configured | Run `configure_l3vpn.yml` first |
| Session flapping | LDP labels missing | Verify LDP is operational on all links |

## Network Topology

```
PE1 (10.0.0.1) <--iBGP VPNv4--> PE2 (10.0.0.4)
       |                            |
     [OSPF]                      [OSPF]
       |                            |
     [LDP]                        [LDP]
       |                            |
      P1 --------[MPLS]---------- P2
```

- **Control Plane**: VPNv4 routes exchanged via iBGP
- **Data Plane**: Traffic forwarded via MPLS label-switched paths
- **IGP**: OSPF provides reachability between loopbacks
- **Label Distribution**: LDP provides labels for LSPs

## What Happens Next?

After VPNv4 BGP is established:
1. Customer routes in VRF A will be exported to VPNv4
2. VPNv4 routes will be advertised to the remote PE
3. Remote PE will import VPNv4 routes into its VRF A
4. Customer sites can communicate across the MPLS network

## Related Playbooks

- `configure_ospf_network.yml` - IGP for loopback reachability
- `configure_ldp.yml` - Label distribution for MPLS forwarding
- `configure_l3vpn.yml` - VRF configuration on PEs
- Future: Customer edge (CE) BGP peering playbooks

## References

- [RFC 4364 - BGP/MPLS IP VPNs](https://datatracker.ietf.org/doc/html/rfc4364)
- [VyOS VRF Documentation](https://docs.vyos.io/en/latest/configuration/vrf/index.html)
- [VyOS BGP Documentation](https://docs.vyos.io/en/latest/configuration/protocols/bgp.html)
