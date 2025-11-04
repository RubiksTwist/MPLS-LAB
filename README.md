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

## Security: Azure Key Vault Integration

For production environments, store VyOS passwords in Azure Key Vault instead of plaintext:

```bash
# See detailed setup guide
cat README-AZURE-KEYVAULT.md

# Quick start:
# 1. Create Azure Key Vault and Service Principal
# 2. Copy .env.example to .env and add your credentials
# 3. Load environment: export $(grep -v '^#' .env | grep -v '^$' | xargs)
# 4. Test: ansible-playbook ansible/playbooks/test_keyvault.yml
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

## How Ansible Works in This Environment

### Connection Method: SSH + CLI (Not NETCONF)

This lab uses **`network_cli`** connection type with **SSH**, not NETCONF. Here's how it works:

1. **Connection Layer**: Ansible connects via SSH using the `paramiko` library (Python SSH client)
2. **Command Execution**: Commands are sent directly to the VyOS CLI (like you would manually)
3. **Configuration**: Uses `vyos.vyos.vyos_config` and `vyos.vyos.vyos_command` modules from the VyOS Ansible collection

### Why network_cli instead of NETCONF?

- **Simplicity**: VyOS CLI is straightforward and well-documented
- **Compatibility**: Works with all VyOS versions (NETCONF requires specific config)
- **Reliability**: Direct CLI commands are predictable in lab environments
- **Debugging**: Easier to troubleshoot - you can see exactly what commands are run

### Key Configuration (in `hosts.ini`)

```ini
ansible_connection=network_cli    # Use CLI-based connection
ansible_network_os=vyos.vyos.vyos # VyOS-specific CLI parser
ansible_transport=paramiko        # Use paramiko for SSH
ansible_user=vyos                 # SSH username
ansible_password=vyos             # SSH password (plaintext for lab)
```

### Example: How a Playbook Works

When you run a playbook like `connectivity_check.yml`:

1. Ansible SSH's into each router (e.g., `ssh vyos@192.168.100.11`)
2. Enters VyOS CLI session
3. Sends commands like `show configuration commands`
4. Parses the output
5. Returns results to Ansible for processing

### Security Note

⚠️ **Lab Environment Only**: This setup uses:
- Plaintext passwords in inventory
- SSH host key checking disabled
- Password-based auth (no SSH keys)

For production, you should use:
- Ansible Vault for encrypted passwords
- SSH key authentication
- Proper host key verification

## Development

See [README-LINTING.md](README-LINTING.md) for linting setup and pre-commit hooks.
