# MPLS-LAB Ansible Playbooks Organization

## Primary Deployment Playbooks (Use These)

### 1. **cleanup_duplicate_ips.yml**
**Purpose:** Remove duplicate IP addresses before deployment  
**When to use:** Before any major reconfiguration or redeployment  
**Idempotent:** ✅ Yes  
```bash
ansible-playbook ansible/playbooks/cleanup_duplicate_ips.yml
```

### 2. **configure_mpls.yml** 
**Purpose:** Main MPLS network configuration (OSPF, BGP, LDP, VRFs)  
**When to use:** Initial deployment or full reconfiguration  
**Idempotent:** ⚠️ Partial (deletes protocols first, then rebuilds)  
**Recommended workflow:**
```bash
# Step 1: Clean up
ansible-playbook ansible/playbooks/cleanup_duplicate_ips.yml

# Step 2: Deploy MPLS
ansible-playbook ansible/playbooks/configure_mpls.yml
```

### 3. **backup_config.yml**
**Purpose:** Backup all router configurations  
**When to use:** Before any changes, or on a schedule  
**Idempotent:** ✅ Yes (read-only)  
```bash
ansible-playbook ansible/playbooks/backup_config.yml
```

---

## Verification Playbooks (Read-Only)

Located in: `verification/`

### verify_mpls.yml
Check MPLS network status (OSPF, LDP, BGP, routes)

### test_mpls_connectivity.yml
Test end-to-end connectivity through MPLS VPN

### connectivity_check.yml
Basic network connectivity verification

**Usage:**
```bash
ansible-playbook ansible/playbooks/verification/verify_mpls.yml
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml
```

---

## Archived Playbooks

Located in: `archive/`

These playbooks are deprecated or superseded by better alternatives:

- **configure_routers.yml** → Use `configure_mpls.yml` instead
- **configure_interfaces_idempotent.yml** → Functionality merged into `configure_mpls.yml`
- **configure_mpls_idempotent.yml** → Experimental, use `configure_mpls.yml`
- **migrate_mgmt_ips.yml** → One-time migration, no longer needed
- **connectivity_check_keyvault.yml** → Use standard `connectivity_check.yml`
- **test_keyvault.yml** → Development/testing only
- **ping_between_routers.yml** → Use `test_mpls_connectivity.yml` instead
- **show_adjacent_routes.yml** → Manual verification, not automated workflow

---

## Recommended Workflow

### Initial Setup
```bash
# 1. Backup existing configs (if any)
ansible-playbook ansible/playbooks/backup_config.yml

# 2. Clean any existing duplicate IPs
ansible-playbook ansible/playbooks/cleanup_duplicate_ips.yml

# 3. Deploy full MPLS configuration
ansible-playbook ansible/playbooks/configure_mpls.yml

# 4. Verify deployment
ansible-playbook ansible/playbooks/verification/verify_mpls.yml
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml
```

### Making Changes
```bash
# 1. Always backup first
ansible-playbook ansible/playbooks/backup_config.yml

# 2. Make changes to host_vars or templates

# 3. Clean and redeploy
ansible-playbook ansible/playbooks/cleanup_duplicate_ips.yml
ansible-playbook ansible/playbooks/configure_mpls.yml

# 4. Verify
ansible-playbook ansible/playbooks/verification/verify_mpls.yml
```

### Daily Operations
```bash
# Quick health check
ansible-playbook ansible/playbooks/verification/verify_mpls.yml

# Test connectivity
ansible-playbook ansible/playbooks/verification/test_mpls_connectivity.yml

# Scheduled backup
ansible-playbook ansible/playbooks/backup_config.yml
```

---

## File Organization

```
ansible/playbooks/
├── PLAYBOOKS-README.md              # This file
├── backup_config.yml                # Backup configurations
├── cleanup_duplicate_ips.yml        # Clean duplicate IPs
├── configure_mpls.yml               # Main MPLS deployment
│
├── verification/                    # Read-only verification
│   ├── verify_mpls.yml
│   ├── test_mpls_connectivity.yml
│   └── connectivity_check.yml
│
├── archive/                         # Deprecated/superseded
│   ├── configure_routers.yml
│   ├── configure_interfaces_idempotent.yml
│   ├── configure_mpls_idempotent.yml
│   ├── migrate_mgmt_ips.yml
│   ├── connectivity_check_keyvault.yml
│   ├── test_keyvault.yml
│   ├── ping_between_routers.yml
│   ├── show_adjacent_routes.yml
│   ├── check_services.yml
│   └── router_metrics.yml
│
├── host_vars/                       # Per-router configuration
├── group_vars/                      # Group variables
└── templates/                       # Jinja2 templates
    ├── interfaces.j2
    └── routing.j2
```

---

## Notes

- **Always run `cleanup_duplicate_ips.yml` before `configure_mpls.yml`** to ensure clean state
- **Backup before making changes** using `backup_config.yml`
- **Use verification playbooks** to confirm deployment success
- **Archived playbooks are kept for reference** but should not be used in production

---

**Last Updated:** 2025-11-06
