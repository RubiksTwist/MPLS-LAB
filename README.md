# MPLS Lab

Ansible-managed MPLS network lab with VyOS routers running in EVE-NG.

## Network Topology

- **CE1** (192.168.100.11) - Customer Edge 1
- **PE1** (192.168.100.12) - Provider Edge 1
- **P1** (192.168.100.13) - Provider Core 1
- **P2** (192.168.100.14) - Provider Core 2
- **PE2** (192.168.100.15) - Provider Edge 2
- **CE2** (192.168.100.16) - Customer Edge 2

## Quick Start

### Run Playbooks

```bash
# Check connectivity and configuration
ansible-playbook ansible/playbooks/connectivity_check.yml

# Verify inter-router ping
ansible-playbook ansible/playbooks/ping_between_routers.yml

# Check router metrics (uptime, load, memory, interfaces)
ansible-playbook ansible/playbooks/router_metrics.yml

# Check service health (OSPF, LDP, BGP)
ansible-playbook ansible/playbooks/check_services.yml
```

### Linting

```bash
# Run YAML lint
yamllint ansible/

# Run Ansible lint
ansible-lint ansible/

# Or use VS Code tasks (Ctrl+Shift+P -> "Run Task")
```

## Troubleshooting

### After Router Restart

If you restart the routers and get SSH fingerprint errors, clear the cached host keys:

```bash
for i in {11..16}; do ssh-keygen -R 192.168.100.$i -f ~/.ssh/known_hosts 2>/dev/null; done && echo "Cleared SSH fingerprints for 192.168.100.11-16"
```

> **Note:** `ansible.cfg` is already configured with `host_key_checking = False`, but clearing known_hosts may still be needed if paramiko caches keys.

### Common Issues

- **SSH timeout**: Check that all routers are powered on in EVE-NG
- **IP conflicts**: Run `connectivity_check.yml` to verify unique IPs
- **OSPF not forming**: Check interfaces are up with `check_services.yml`

## Project Structure

```
ansible/
├── inventories/
│   └── hosts.ini          # Router inventory with management IPs
├── playbooks/
│   ├── connectivity_check.yml
│   ├── ping_between_routers.yml
│   ├── router_metrics.yml
│   ├── check_services.yml
│   ├── migrate_mgmt_ips.yml
│   ├── host_vars/         # Per-router configuration
│   └── group_vars/        # Common VyOS settings
└── backups/               # Router config backups (gitignored)
```

## Development

See [README-LINTING.md](README-LINTING.md) for linting setup and pre-commit hooks.
