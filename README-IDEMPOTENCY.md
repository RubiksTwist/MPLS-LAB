# Idempotency Strategies for VyOS Configuration with Ansible

## The Challenge

VyOS configuration commands use `set` which is **additive** - it adds to existing configuration rather than replacing it. This causes duplicate entries when re-running playbooks.

## Solution Approaches

### 1. **Delete-Then-Set** (Current Approach)
```yaml
- name: Clear existing configuration
  vyos.vyos.vyos_config:
    lines:
      - delete protocols ospf
      - delete protocols bgp
      - delete vrf
    save: false
  ignore_errors: true

- name: Apply new configuration
  vyos.vyos.vyos_config:
    lines: "{{ config_lines }}"
    save: true
```

**Pros:**
- Simple and explicit
- Guarantees clean state
- Works with any VyOS version

**Cons:**
- **Disruptive**: Tears down protocols completely before rebuilding
- Brief service outage during reconfiguration
- OSPF/BGP sessions flap

**Best for:** Initial setup, major reconfigurations

---

### 2. **Configuration Replace** (VyOS Native)
```yaml
- name: Generate complete configuration
  template:
    src: complete_config.j2
    dest: /tmp/{{ inventory_hostname }}.conf

- name: Load configuration (replace mode)
  vyos.vyos.vyos_config:
    src: /tmp/{{ inventory_hostname }}.conf
    config_format: set
    replace: yes  # Replace entire config
```

**Pros:**
- Atomic configuration replacement
- VyOS handles diff internally
- Most idempotent approach

**Cons:**
- Requires complete configuration template
- Still causes protocol flaps
- Complex template maintenance

**Best for:** Infrastructure-as-Code with version control

---

### 3. **Selective Deletion** (Hybrid Approach)
```yaml
- name: Delete only changed interfaces
  vyos.vyos.vyos_config:
    lines:
      - delete interfaces ethernet {{ item.key }} address
  loop: "{{ interfaces_to_update }}"
  ignore_errors: yes

- name: Set new addresses
  vyos.vyos.vyos_config:
    lines:
      - set interfaces ethernet {{ item.key }} address '{{ item.value }}'
  loop: "{{ interfaces_to_update }}"
```

**Pros:**
- Minimizes disruption
- Only changes what needs changing
- Faster than full delete-set

**Cons:**
- Requires tracking what changed
- More complex logic
- Still brief interface flaps

**Best for:** Incremental updates, operational changes

---

### 4. **Configuration Comparison** (Most Sophisticated)
```yaml
- name: Get current configuration
  vyos.vyos.vyos_command:
    commands:
      - show configuration commands | match "{{ section }}"
  register: current_config

- name: Generate desired configuration
  set_fact:
    desired_config: "{{ lookup('template', 'config.j2') }}"

- name: Calculate configuration diff
  set_fact:
    config_to_delete: "{{ current_config | difference(desired_config) }}"
    config_to_add: "{{ desired_config | difference(current_config) }}"

- name: Apply only differences
  vyos.vyos.vyos_config:
    lines: "{{ config_to_add }}"
```

**Pros:**
- Truly idempotent - no changes if config matches
- Minimal disruption
- Efficient

**Cons:**
- Complex to implement
- Parsing VyOS config can be tricky
- Edge cases with nested configuration

**Best for:** Production environments, automated pipelines

---

## Recommended Strategy for MPLS-LAB

### Phase 1: Initial Deployment
Use **Delete-Then-Set** for clean initialization:

```bash
ansible-playbook configure_interfaces_idempotent.yml
ansible-playbook configure_mpls.yml
```

### Phase 2: Operational Changes
Create targeted playbooks for specific changes:

**For IP address changes:**
```yaml
# cleanup_duplicate_ips.yml - Run this before redeployment
```

**For routing protocol changes:**
```yaml
# configure_mpls.yml - Already has delete logic
```

### Phase 3: Maintenance Mode
Use configuration comparison for ongoing management:

```yaml
# Future: configure_full_idempotent.yml
# Would compare desired vs actual state
```

---

## Why Interfaces Need Special Handling

**Problem:** VyOS allows multiple IP addresses per interface (by design - for multihoming, VRRP, etc.)

**Example:**
```bash
set interfaces ethernet eth1 address '10.0.1.1/30'  # First run
set interfaces ethernet eth1 address '172.16.0.1/30'  # Second run
# Result: BOTH IPs exist on eth1!
```

**Solution:** Always `delete interfaces ethernet ethX address` before setting new ones.

---

## Best Practices

### 1. **Pre-Flight Cleanup**
Always run cleanup before major reconfigurations:
```bash
ansible-playbook cleanup_duplicate_ips.yml
ansible-playbook configure_mpls.yml
```

### 2. **Use Configuration Backups**
Before any change:
```bash
ansible-playbook backup_config.yml
```

### 3. **Idempotent Playbook Design**
Structure all playbooks with this pattern:
```yaml
- name: Remove old configuration
  vyos.vyos.vyos_config:
    lines: "delete {{ item }}"
  ignore_errors: yes

- name: Apply new configuration
  vyos.vyos.vyos_config:
    lines: "set {{ item }}"

- name: Verify result
  vyos.vyos.vyos_command:
    commands: "show {{ item }}"
```

### 4. **Template-Driven Configuration**
Generate all configs from templates - easier to maintain and diff:
```jinja2
{# routing.j2 #}
{% if router_config.protocols.ospf is defined %}
set protocols ospf router-id '{{ router_config.router_id }}'
...
{% endif %}
```

### 5. **Use Tags for Selective Runs**
```yaml
tasks:
  - name: Configure OSPF
    tags: ['ospf', 'routing']
    ...

  - name: Configure BGP
    tags: ['bgp', 'routing']
    ...
```

Run specific parts:
```bash
ansible-playbook configure_mpls.yml --tags ospf
```

---

## Future Improvements

1. **Create `vyos_config_replace` custom module**
   - Handles delete-then-set automatically
   - Built-in diff generation
   - Idempotent by default

2. **Implement configuration validation**
   - Pre-deployment config syntax check
   - Post-deployment verification
   - Automatic rollback on failure

3. **Use Ansible diff mode**
   ```bash
   ansible-playbook --check --diff configure_mpls.yml
   ```
   Shows what WOULD change without applying

4. **Migrate to declarative state management**
   - Define desired state in data files
   - Let playbook handle all diff calculation
   - Similar to Terraform approach

---

## Quick Reference

| Task | Command | Idempotent? |
|------|---------|-------------|
| Clean duplicates | `ansible-playbook cleanup_duplicate_ips.yml` | ✅ Yes |
| Configure interfaces | `ansible-playbook configure_interfaces_idempotent.yml` | ✅ Yes |
| Configure MPLS | `ansible-playbook configure_mpls.yml` | ⚠️ Partial (deletes first) |
| Backup configs | `ansible-playbook backup_config.yml` | ✅ Yes |
| Verify setup | `ansible-playbook verify_mpls.yml` | ✅ Yes (read-only) |

---

**Last Updated:** 2025-11-06
**Author:** Documented after encountering duplicate IP issues during MPLS L3VPN deployment
