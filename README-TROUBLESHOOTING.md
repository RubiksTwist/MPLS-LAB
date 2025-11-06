# MPLS L3VPN Troubleshooting Guide

## Common Issues and Solutions

### Issue 1: Duplicate IP Addresses After Reconfiguration

**Symptom:** BGP sessions fail to establish, or routes exist but connectivity fails.

**Root Cause:** When reconfiguring IP addresses, VyOS may retain old IP addresses on interfaces alongside new ones.

**Detection:**
```bash
ansible ROUTER -m vyos.vyos.vyos_command -a "commands='show interfaces'"
ansible ROUTER -m vyos.vyos.vyos_command -a "commands='show configuration commands | match \"address\"'"
```

**Solution:** Manually remove old IP addresses before applying new configuration:
```bash
# For each router with duplicate IPs:
ansible ROUTER -m vyos.vyos.vyos_config -a "lines='delete interfaces ethernet ethX address OLD_IP/MASK'"
ansible ROUTER -m vyos.vyos.vyos_config -a "lines='delete interfaces loopback lo address OLD_IP/MASK'"
```

**Prevention:** Always delete old interface IP addresses explicitly before deploying new configuration, or use a complete wipe/rebuild approach.

---

### Issue 2: PE-CE BGP Sessions Not Establishing

**Symptom:** `show ip bgp vrf A summary` shows sessions in "Active" state instead of "Established".

**Root Causes:**
1. Duplicate IP addresses on customer-facing interfaces (see Issue 1)
2. Missing address-family activation on BGP neighbors
3. Missing network statements to advertise customer routes

**Detection:**
```bash
ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show ip bgp vrf A summary'"
ansible CE1,CE2 -m vyos.vyos.vyos_command -a "commands='show ip bgp summary'"
```

**Solution:**

**Step 1:** Ensure template generates neighbor address-family activation:
```jinja2
# In routing.j2 template - BGP neighbor configuration
{% if neighbor_config.address_family is defined %}
{% for af in neighbor_config.address_family %}
set protocols bgp neighbor '{{ neighbor_ip }}' address-family {{ af }}
{% endfor %}
{% endif %}
```

**Step 2:** Ensure CE host_vars includes address-family configuration:
```yaml
protocols:
  bgp:
    asn: "65001"
    neighbors:
      "172.16.0.1":
        remote_as: "65000"
        description: "eBGP to PE1"
        address_family:
          - "ipv4-unicast"
    address_family:
      ipv4_unicast:
        networks:
          - "10.0.1.0/24"
```

**Step 3:** Ensure template generates network advertisements:
```jinja2
# In routing.j2 template - Global BGP section
{% if router_config.protocols.bgp.address_family is defined and router_config.protocols.bgp.address_family.ipv4_unicast is defined %}
{% if router_config.protocols.bgp.address_family.ipv4_unicast.networks is defined %}
{% for network in router_config.protocols.bgp.address_family.ipv4_unicast.networks %}
set protocols bgp address-family ipv4-unicast network '{{ network }}'
{% endfor %}
{% endif %}
{% endif %}
```

---

### Issue 3: VPN Label Not Allocated

**Symptom:** `show bgp ipv4 vpn X.X.X.X/Y` shows "not allocated" for local routes.

**Expected Behavior:** This is actually NORMAL in VyOS. The "not allocated" message appears even when labels ARE allocated. Verify actual label allocation with:

```bash
ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show mpls table'"
```

Look for BGP type labels with VRF name as nexthop.

**Required Configuration:**
```
set vrf name A protocols bgp address-family ipv4-unicast label vpn export 'auto'
```

---

## Systematic Troubleshooting Workflow

### Phase 1: Control Plane Verification

1. **Check OSPF (Provider routers)**
   ```bash
   ansible PE1,PE2,P1,P2 -m vyos.vyos.vyos_command -a "commands='show ip ospf neighbor'"
   ```
   Expected: All neighbors in "Full" state

2. **Check LDP (Provider routers)**
   ```bash
   ansible PE1,PE2,P1,P2 -m vyos.vyos.vyos_command -a "commands='show mpls ldp neighbor'"
   ```
   Expected: All sessions "OPERATIONAL"

3. **Check MP-BGP VPNv4 (PEs only)**
   ```bash
   ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show ip bgp summary'"
   ```
   Expected: PE-to-PE session "Established" with PfxRcd/PfxSnt > 0

4. **Check PE-CE BGP (PEs)**
   ```bash
   ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show ip bgp vrf A summary'"
   ```
   Expected: CE sessions "Established" with PfxRcd/PfxSnt > 0

5. **Check CE BGP**
   ```bash
   ansible CE1,CE2 -m vyos.vyos.vyos_command -a "commands='show ip bgp summary'"
   ```
   Expected: PE session "Established" with remote customer routes learned

---

### Phase 2: Routing Table Verification

1. **Check VRF Routing Tables (PEs)**
   ```bash
   ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show ip route vrf A'"
   ```
   Expected: Local customer routes (via CE), remote customer routes (via MP-BGP with labels)

2. **Check CE Routing Tables**
   ```bash
   ansible CE1,CE2 -m vyos.vyos.vyos_command -a "commands='show ip route'"
   ```
   Expected: Local LAN (connected), remote customer LAN (via BGP from PE)

3. **Check MPLS Forwarding Table (PEs)**
   ```bash
   ansible PE1,PE2 -m vyos.vyos.vyos_command -a "commands='show mpls table'"
   ```
   Expected: LDP labels for all loopbacks, BGP labels for VRFs

---

### Phase 3: Data Plane Testing

1. **Test PE-to-CE Connectivity**
   ```bash
   ansible PE1 -m vyos.vyos.vyos_command -a "commands='ping 10.0.1.1 vrf A count 3'"
   ```

2. **Test PE-to-PE VPN Forwarding**
   ```bash
   ansible PE1 -m vyos.vyos.vyos_command -a "commands='ping 10.0.2.1 vrf A count 3'"
   ```

3. **Test End-to-End Customer Connectivity**
   ```bash
   ansible CE1 -m vyos.vyos.vyos_command -a "commands='ping 10.0.2.1 count 5 source-address 10.0.1.1'"
   ```

---

## Quick Diagnosis Commands

### When BGP Won't Establish
```bash
# Check for duplicate IPs
ansible ALL -m vyos.vyos.vyos_command -a "commands='show interfaces'" | grep -A2 "eth1"

# Check BGP configuration
ansible PE1 -m vyos.vyos.vyos_command -a "commands='show configuration commands | match bgp'"
```

### When Routes Not Learned
```bash
# Check what CE is advertising
ansible CE1 -m vyos.vyos.vyos_command -a "commands='show ip bgp neighbors 172.16.0.1 advertised-routes'"

# Check what PE received
ansible PE1 -m vyos.vyos.vyos_command -a "commands='show ip bgp vrf A neighbors 172.16.0.2 routes'"
```

### When Connectivity Fails
```bash
# Verify route exists
ansible CE1 -m vyos.vyos.vyos_command -a "commands='show ip route 10.0.2.0/24'"

# Check VRF routing
ansible PE1 -m vyos.vyos.vyos_command -a "commands='show ip route vrf A 10.0.2.0/24'"

# Verify MPLS labels
ansible PE1 -m vyos.vyos.vyos_command -a "commands='show mpls table'"
```

---

## Lessons Learned

1. **Always clean up old IPs**: VyOS accumulates IP addresses on reconfiguration rather than replacing them.

2. **CE BGP needs explicit configuration**: The routing.j2 template must explicitly handle:
   - Neighbor address-family activation
   - Network statement generation for global BGP
   
3. **"not allocated" is misleading**: VyOS shows this message even when labels ARE allocated. Trust `show mpls table`.

4. **Systematic verification is key**: Don't skip steps. Verify control plane before testing data plane.

5. **Template completeness matters**: Initial template only handled VRF BGP networks, not global BGP networks for CEs.

---

## Configuration Checklist After Any Changes

- [ ] No duplicate IP addresses on any interface
- [ ] All OSPF neighbors Full
- [ ] All LDP sessions OPERATIONAL  
- [ ] MP-BGP VPNv4 session Established between PEs
- [ ] PE-CE BGP sessions Established
- [ ] VRF routing tables contain remote customer routes with labels
- [ ] CE routing tables contain remote customer routes via BGP
- [ ] MPLS table shows BGP labels for VRFs
- [ ] End-to-end connectivity working

---

**Last Updated:** 2025-11-06
**Based on:** Working L3VPN reference from https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/
