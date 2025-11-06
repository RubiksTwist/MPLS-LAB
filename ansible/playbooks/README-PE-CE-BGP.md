# PE to CE BGP Configuration Playbook

## Overview

This playbook configures the final piece of the MPLS L3VPN setup: eBGP peering between Provider Edge (PE) routers and Customer Edge (CE) routers. This allows customer sites to exchange routes across the MPLS backbone while maintaining VPN isolation through VRF A.

## What This Playbook Does

### Provider Edge Routers (PE1 & PE2)
- Assigns customer-facing interfaces to VRF A
- Configures eBGP peering with CE routers
- Advertises PE-CE link networks into VRF A

### Customer Edge Routers (CE1 & CE2)
- Configures customer network interfaces (loopbacks)
- Sets up customer-facing links to PE routers
- Establishes eBGP peering with PE routers
- Redistributes connected routes into BGP

## Network Topology

```
Customer Site 1                                      Customer Site 2
┌──────────────┐                                    ┌──────────────┐
│     CE1      │                                    │     CE2      │
│  10.0.1.1/24 │                                    │  10.0.2.1/24 │
│  AS 65001    │                                    │  AS 65002    │
└──────┬───────┘                                    └──────┬───────┘
       │ eth1: 172.16.0.2/30                              │ eth1: 172.16.0.6/30
       │ eBGP                                             │ eBGP
       │                                                  │
┌──────┴───────┐         MPLS Core            ┌──────────┴─────┐
│     PE1      │                               │      PE2       │
│  VRF A       │         iBGP VPNv4           │   VRF A        │
│  AS 65000    ├───────────────────────────────┤   AS 65000     │
└──────────────┘    P1 ──[LDP]── P2           └────────────────┘
 eth1: 172.16.0.1/30                            eth2: 172.16.0.5/30
 (in VRF A)                                     (in VRF A)
```

## Configuration Details

### PE1 Configuration
```bash
# Interface in VRF A
set interfaces ethernet eth1 address '172.16.0.1/30'
set interfaces ethernet eth1 description 'Link to CE1 (Customer Edge)'
set interfaces ethernet eth1 vrf 'A'

# eBGP to CE1
set vrf name A protocols bgp address-family ipv4-unicast network 172.16.0.0/30
set vrf name A protocols bgp neighbor 172.16.0.2 address-family ipv4-unicast
set vrf name A protocols bgp neighbor 172.16.0.2 remote-as '65001'
```

### PE2 Configuration
```bash
# Interface in VRF A
set interfaces ethernet eth2 address '172.16.0.5/30'
set interfaces ethernet eth2 description 'Link to CE2 (Customer Edge)'
set interfaces ethernet eth2 vrf 'A'

# eBGP to CE2
set vrf name A protocols bgp address-family ipv4-unicast network 172.16.0.4/30
set vrf name A protocols bgp neighbor 172.16.0.6 address-family ipv4-unicast
set vrf name A protocols bgp neighbor 172.16.0.6 remote-as '65002'
```

### CE1 Configuration
```bash
# Customer network
set interfaces dummy dum0 address '10.0.1.1/24'
set interfaces dummy dum0 description 'Customer Network A - Site 1'

# Link to PE1
set interfaces ethernet eth1 address '172.16.0.2/30'
set interfaces ethernet eth1 description 'Link to PE1 (Provider Edge)'

# eBGP to PE1
set protocols bgp address-family ipv4-unicast redistribute connected
set protocols bgp neighbor 172.16.0.1 address-family ipv4-unicast
set protocols bgp neighbor 172.16.0.1 remote-as '65000'
set protocols bgp system-as '65001'
```

### CE2 Configuration
```bash
# Customer network
set interfaces dummy dum0 address '10.0.2.1/24'
set interfaces dummy dum0 description 'Customer Network A - Site 2'

# Link to PE2
set interfaces ethernet eth1 address '172.16.0.6/30'
set interfaces ethernet eth1 description 'Link to PE2 (Provider Edge)'

# eBGP to PE2
set protocols bgp address-family ipv4-unicast redistribute connected
set protocols bgp neighbor 172.16.0.5 address-family ipv4-unicast
set protocols bgp neighbor 172.16.0.5 remote-as '65000'
set protocols bgp system-as '65002'
```

## Configuration Breakdown

### PE Router Configuration

| Command | Purpose |
|---------|---------|
| `set interfaces ethernet ethX vrf 'A'` | Assigns interface to VRF A (customer isolation) |
| `set vrf name A protocols bgp neighbor X.X.X.X address-family ipv4-unicast` | Enables IPv4 unicast for CE neighbor in VRF |
| `set vrf name A protocols bgp neighbor X.X.X.X remote-as '6500X'` | Sets CE AS number (eBGP) |
| `set vrf name A protocols bgp address-family ipv4-unicast network X.X.X.X/30` | Advertises PE-CE link into VRF |

### CE Router Configuration

| Command | Purpose |
|---------|---------|
| `set protocols bgp address-family ipv4-unicast redistribute connected` | Advertises connected networks to PE |
| `set protocols bgp neighbor X.X.X.X remote-as '65000'` | Sets provider AS (eBGP to PE) |
| `set protocols bgp system-as '6500X'` | Defines customer AS number |

## Prerequisites

**CRITICAL**: The following playbooks MUST be run in order before this one:

### 1. OSPF Configuration
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ospf_network.yml
```
- Provides IGP reachability between PE loopbacks
- Required for VPNv4 BGP session establishment

### 2. MPLS LDP Configuration
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ldp.yml
```
- Distributes MPLS labels for LSPs
- Required for MPLS data plane forwarding

### 3. L3VPN (VRF) Configuration
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_l3vpn.yml
```
- Creates VRF A on PE routers
- Configures route targets and route distinguishers

### 4. VPNv4 BGP Configuration
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_vpnv4_bgp.yml
```
- Establishes iBGP VPNv4 peering between PE1 and PE2
- Required for VPN route exchange across the MPLS core

## Usage

### Run the Playbook
```bash
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_pe_ce_bgp.yml
```

### Expected Output
The playbook will:
1. Configure PE1 eth1 in VRF A and eBGP to CE1
2. Configure PE2 eth2 in VRF A and eBGP to CE2
3. Configure CE1 with customer network and eBGP to PE1
4. Configure CE2 with customer network and eBGP to PE2
5. Wait 20 seconds for BGP sessions to establish
6. Display BGP summary and routes on all routers
7. **Test end-to-end connectivity** from CE1 to CE2

## Verification

### Check PE-CE BGP Peering on PE Routers
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp vrf A summary'"
```

Expected: CE neighbors in "Established" state

### Check BGP Routes on CE Routers
```bash
ansible CE1,CE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip bgp'"
```

**Expected on CE1:**
```
Network          Next Hop            Path
10.0.1.0/24      0.0.0.0             ?               (local)
10.0.2.0/24      172.16.0.1          65000 65002 ?   (from PE1)
172.16.0.0/30    0.0.0.0             ?               (local)
172.16.0.4/30    172.16.0.1          65000 65002 ?   (from PE1)
```

### Check VRF Routes on PE Routers
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route vrf A'"
```

PE1 should see routes to 10.0.2.0/24 (CE2's network)

### End-to-End Connectivity Test
```bash
ansible CE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='ping 10.0.2.1 source-address 10.0.1.1 count 5'"
```

**Expected:** 5 successful ping replies from CE2

## How It Works: Route Flow

### Route Advertisement Flow (CE1 → CE2)

1. **CE1** redistributes connected routes (10.0.1.0/24) into BGP
2. **CE1** advertises 10.0.1.0/24 to **PE1** via eBGP
3. **PE1** imports route into VRF A
4. **PE1** exports route to VPNv4 (adds RD 65000:1, RT 65000:1, MPLS label)
5. **PE1** advertises VPNv4 route to **PE2** via iBGP
6. **PE2** imports VPNv4 route into VRF A (matches RT 65000:1)
7. **PE2** advertises 10.0.1.0/24 to **CE2** via eBGP
8. **CE2** installs route: 10.0.1.0/24 via 172.16.0.5 (PE2)

### Data Plane Forwarding (CE1 → CE2)

```
CE1 (10.0.1.1) sends to 10.0.2.1:
  └─> [IP: 10.0.1.1 → 10.0.2.1] to PE1 (172.16.0.1)

PE1 receives in VRF A:
  └─> Lookup in VRF A table → Next-hop: PE2 (10.0.0.4)
  └─> Push VPN label + Transport label (LDP)
  └─> [MPLS: Transport Label | VPN Label][IP: 10.0.1.1 → 10.0.2.1] to P1

P1 (Label Switch):
  └─> Swap transport label
  └─> [MPLS: New Transport Label | VPN Label][IP] to P2

P2 (Label Switch):
  └─> Swap/Pop transport label
  └─> [MPLS: VPN Label][IP: 10.0.1.1 → 10.0.2.1] to PE2

PE2 receives:
  └─> Pop VPN label → Route in VRF A
  └─> [IP: 10.0.1.1 → 10.0.2.1] to CE2 (172.16.0.6)

CE2 receives:
  └─> Delivers to 10.0.2.1 ✓
```

## Troubleshooting

### BGP Session Not Establishing

**Check physical connectivity:**
```bash
# From CE1
ansible CE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='ping 172.16.0.1'"
```

**Check BGP neighbor state:**
```bash
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp vrf A neighbor 172.16.0.2'"
```

### Routes Not Being Exchanged

**Verify route redistribution on CE:**
```bash
ansible CE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route connected'"
```

**Check VPNv4 routes between PEs:**
```bash
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show bgp ipv4 vpn'"
```

### Ping Fails Between CE1 and CE2

**Check full route path:**
```bash
# On CE1 - Should see route to 10.0.2.0/24
ansible CE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route 10.0.2.0/24'"

# On PE1 - Should see route in VRF A
ansible PE1 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route vrf A 10.0.2.0/24'"

# On PE2 - Should see route in VRF A
ansible PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route vrf A 10.0.1.0/24'"

# On CE2 - Should see route to 10.0.1.0/24
ansible CE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show ip route 10.0.1.0/24'"
```

**Verify MPLS labels:**
```bash
ansible PE1,PE2 -i ansible/inventories/hosts.ini -m vyos.vyos.vyos_command -a "commands='show mpls ldp binding'"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| CE BGP session stuck in "Active" | No IP reachability to PE | Check interface IPs and VRF assignment |
| No routes from CE in VRF | Redistribute not configured | Verify `redistribute connected` on CE |
| Routes learned but ping fails | Missing VPNv4 peering or LDP labels | Verify previous playbooks (VPNv4 BGP, LDP) |
| Routes on CE but wrong next-hop | VRF misconfiguration on PE | Check interface is in correct VRF |

## Important Notes

### VRF Isolation
- PE interfaces assigned to VRF A are **isolated** from global routing table
- This provides **customer traffic separation** (VPN functionality)
- Different customers could use same IP space (10.0.1.0/24) in different VRFs

### BGP AS Numbers
- **Provider AS (65000)**: Used by PE routers
- **Customer AS (65001, 65002)**: Unique per customer site
- Different AS numbers = eBGP (external BGP)
- Same AS = iBGP (like PE1 ↔ PE2 for VPNv4)

### Route Distinguisher & Route Target
- Configured in earlier playbook (`configure_l3vpn.yml`)
- **RD 65000:1**: Makes routes globally unique in MP-BGP
- **RT 65000:1**: Controls import/export (both sites use same RT)

## Success Criteria

After running this playbook, you should have:

✅ **BGP sessions established:**
- PE1 ↔ CE1 (eBGP, AS 65000 ↔ AS 65001)
- PE2 ↔ CE2 (eBGP, AS 65000 ↔ AS 65002)

✅ **Routes exchanged:**
- CE1 learns 10.0.2.0/24 from PE1
- CE2 learns 10.0.1.0/24 from PE2

✅ **End-to-end connectivity:**
- `ping 10.0.2.1 source-address 10.0.1.1` succeeds

✅ **Full MPLS L3VPN operational!**

## Related Playbooks

1. `configure_ospf_network.yml` - IGP for PE loopback reachability
2. `configure_ldp.yml` - MPLS label distribution
3. `configure_l3vpn.yml` - VRF configuration on PEs
4. `configure_vpnv4_bgp.yml` - VPNv4 iBGP between PEs
5. **`configure_pe_ce_bgp.yml`** ← You are here

## References

- [RFC 4364 - BGP/MPLS IP VPNs](https://datatracker.ietf.org/doc/html/rfc4364)
- [RFC 4271 - BGP-4](https://datatracker.ietf.org/doc/html/rfc4271)
- [VyOS VRF Documentation](https://docs.vyos.io/en/latest/configuration/vrf/index.html)
- [VyOS BGP Documentation](https://docs.vyos.io/en/latest/configuration/protocols/bgp.html)
