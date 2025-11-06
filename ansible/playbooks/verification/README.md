# Verification Playbooks

These playbooks are **read-only** and used for verifying the MPLS network state.  
They do not make any configuration changes.

## Available Playbooks

### verify_mpls.yml
**Purpose:** Comprehensive MPLS network status check  
**Checks:**
- OSPF neighbor relationships
- LDP sessions and bindings
- BGP sessions (MP-BGP VPNv4 and PE-CE)
- VRF routing tables
- MPLS forwarding tables

**Usage:**
```bash
ansible-playbook ansible/playbooks/verification/verify_mpls.yml
```

**When to use:**
- After initial deployment
- After making configuration changes
- Daily health checks
- Troubleshooting

---

### test_mpls_connectivity.yml
**Purpose:** Test end-to-end connectivity through MPLS VPN  
**Tests:**
- CE1 → CE2 connectivity (customer site to site)
- PE → CE local connectivity
- Cross-VPN reachability

**Usage:**
```bash
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml
```

**When to use:**
- Verify customer connectivity works
- After MPLS deployment
- Testing data plane functionality
- Troubleshooting connectivity issues

---

### connectivity_check.yml
**Purpose:** Basic network connectivity verification  
**Checks:**
- Router reachability
- Basic IP connectivity
- Interface status

**Usage:**
```bash
ansible-playbook ansible/playbooks/verification/connectivity_check.yml
```

**When to use:**
- Quick sanity check
- Before making changes
- Verify Ansible can reach all devices

---

## Typical Workflow

### After Deployment
```bash
# 1. Verify control plane
ansible-playbook ansible/playbooks/verification/verify_mpls.yml

# 2. Test data plane
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml
```

### Daily Health Check
```bash
ansible-playbook ansible/playbooks/verification/verify_mpls.yml
```

### Troubleshooting
```bash
# 1. Check basic connectivity
ansible-playbook ansible/playbooks/verification/connectivity_check.yml

# 2. Check MPLS protocols
ansible-playbook ansible/playbooks/verification/verify_mpls.yml

# 3. Test end-to-end
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml
```

---

## Notes

- All verification playbooks are **idempotent** (can be run multiple times safely)
- They do **not modify** router configurations
- Output is displayed for manual review
- Can be run at any time without disrupting service

---

**Created:** 2025-11-06
