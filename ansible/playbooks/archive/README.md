# Archived Playbooks

These playbooks have been archived because they are either:
- **Superseded** by better, more idempotent alternatives
- **One-time use** and no longer needed
- **Development/testing** only

## Archived Files

### configure_routers.yml
**Status:** Superseded  
**Reason:** Non-idempotent, doesn't handle routing protocols  
**Use Instead:** `configure_mpls.yml`  
**Archived:** 2025-11-06

### configure_interfaces_idempotent.yml
**Status:** Merged  
**Reason:** Functionality merged into `configure_mpls.yml` and `cleanup_duplicate_ips.yml`  
**Use Instead:** `cleanup_duplicate_ips.yml` followed by `configure_mpls.yml`  
**Archived:** 2025-11-06

### configure_mpls_idempotent.yml
**Status:** Experimental  
**Reason:** Alternative approach that wasn't fully developed  
**Use Instead:** `configure_mpls.yml` (which already has idempotent delete-then-set logic)  
**Archived:** 2025-11-06

### migrate_mgmt_ips.yml
**Status:** One-time use  
**Reason:** Was used for one-time migration of management IPs, no longer needed  
**Archived:** 2025-11-06

### connectivity_check_keyvault.yml
**Status:** Superseded  
**Reason:** Azure Key Vault integration experimental, standard auth preferred  
**Use Instead:** `connectivity_check.yml`  
**Archived:** 2025-11-06

### test_keyvault.yml
**Status:** Development only  
**Reason:** Testing playbook for Azure Key Vault integration  
**Archived:** 2025-11-06

### ping_between_routers.yml
**Status:** Superseded  
**Reason:** Basic connectivity test, replaced by comprehensive test  
**Use Instead:** `verification/test_mpls_connectivity.yml`  
**Archived:** 2025-11-06

### show_adjacent_routes.yml
**Status:** Manual tool  
**Reason:** Debugging tool for manual use, not part of automated workflow  
**Note:** Can still be used manually if needed  
**Archived:** 2025-11-06

### check_services.yml
**Status:** Incomplete  
**Reason:** Never fully developed, functionality in other playbooks  
**Use Instead:** `verification/verify_mpls.yml`  
**Archived:** 2025-11-06

### router_metrics.yml
**Status:** Incomplete  
**Reason:** Never fully developed  
**Use Instead:** `verification/verify_mpls.yml`  
**Archived:** 2025-11-06

---

## Restoration

If you need to restore any of these playbooks:

```bash
# Move back to main directory
mv ansible/playbooks/archive/PLAYBOOK_NAME.yml ansible/playbooks/

# Or use directly from archive
ansible-playbook ansible/playbooks/archive/PLAYBOOK_NAME.yml
```

---

**Archived:** 2025-11-06  
**Reason:** Playbook cleanup and organization for better maintainability
