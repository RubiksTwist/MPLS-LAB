# MPLS Lab

Ansible-managed MPLS L3VPN network lab with VyOS routers running in EVE-NG.

This lab implements a complete **MPLS Layer 3 VPN** service provider network with:
- **OSPF** for IGP routing
- **LDP** for MPLS label distribution  
- **VRF-based L3VPN** for customer traffic isolation
- **VPNv4 iBGP** for inter-PE route exchange
- **eBGP** for PE-CE customer peering

**Reference Implementation**: Based on [MPLS on VyOS – L3VPN](https://lev-0.com/2024/01/18/mpls-on-vyos-l3vpn/)

## Network Topology

### Routers
- **CE1** (192.168.100.11) - Customer Edge 1
- **PE1** (192.168.100.12) - Provider Edge 1
- **P1** (192.168.100.13) - Provider Core 1
- **P2** (192.168.100.14) - Provider Core 2
- **PE2** (192.168.100.15) - Provider Edge 2
- **CE2** (192.168.100.16) - Customer Edge 2

### Physical Connections
```
CE1 eth1 <---> eth1 PE1 eth2 <---> eth1 P1 eth2 <---> eth1 P2 eth2 <---> eth1 PE2 eth2 <---> eth1 CE2
```

**Note**: All routers have eth0 reserved for management (192.168.100.x/24)

### OSPF Network (Area 0)
- **PE1**: Loopback 10.0.0.1/32, eth2 (10.1.2.1/24) connects to P1
- **P1**: Loopback 10.0.0.2/32, eth1 (10.1.2.2/24) connects to PE1, eth2 (10.2.3.2/24) connects to P2
- **P2**: Loopback 10.0.0.3/32, eth1 (10.2.3.3/24) connects to P1, eth2 (10.3.4.3/24) connects to PE2
- **PE2**: Loopback 10.0.0.4/32, eth1 (10.3.4.4/24) connects to P2

## Quick Start

### MPLS L3VPN Configuration (In Order)

Run these playbooks sequentially to build a complete MPLS L3VPN network:

```bash
# 1. Configure OSPF (IGP for loopback reachability)
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ospf_network.yml

# 2. Configure MPLS LDP (Label distribution)
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_ldp.yml

# 3. Configure L3VPN VRF (VRF A on PE routers)
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_l3vpn.yml

# 4. Configure VPNv4 iBGP (PE-to-PE route exchange)
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_vpnv4_bgp.yml

# 5. Configure PE-CE BGP (Customer edge peering)
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/configure_pe_ce_bgp.yml
```

**Expected Result**: CE1 (10.0.1.0/24) can ping CE2 (10.0.2.0/24) across the MPLS core.

### Verification Playbooks

```bash
# Check MPLS connectivity end-to-end
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/verification/connectivity_check.yml

# Test MPLS L3VPN data plane
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/verification/test_mpls_connectivity.yml

# Verify MPLS configuration
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/verification/verify_mpls.yml
```

### Factory Reset

```bash
# Reset all routers to clean state
ansible-playbook -i ansible/inventories/hosts.ini ansible/playbooks/factory_reset.yml
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
- **VPNv4 BGP stuck in "Active"**: Reboot PE routers or manually add router-id
- **No routes on CE routers**: Verify all 5 playbooks ran successfully in order
- **OSPF not forming**: Check interfaces are up with verification playbooks
- **LDP labels missing**: Ensure OSPF is operational first (prerequisite)

## Additional Documentation

- [README-LINTING.md](README-LINTING.md) - Linting setup and pre-commit hooks
- Individual playbook READMEs in `ansible/playbooks/`

## Architecture Overview

### Control Plane
1. **OSPF**: Provides IGP reachability between all provider routers (loopbacks)
2. **LDP**: Distributes MPLS labels for transport LSPs
3. **VPNv4 iBGP**: Exchanges customer routes between PE routers (with RD/RT)
4. **eBGP**: Peers with customer edge routers for route learning

### Data Plane
- **Dual-label stack**: Transport label (outer) + VPN label (inner)
- **PHP (Penultimate Hop Popping)**: Transport label removed at P2
- **VRF lookup**: VPN label identifies VRF A on egress PE
- **End-to-end**: CE1 (10.0.1.0/24) ↔ MPLS Core ↔ CE2 (10.0.2.0/24)

## Project Structure

```
ansible/
├── inventories/
│   └── hosts.ini                    # Router inventory with management IPs
├── playbooks/
│   ├── configure_ospf_network.yml   # Step 1: OSPF IGP configuration
│   ├── configure_ldp.yml            # Step 2: MPLS LDP label distribution
│   ├── configure_l3vpn.yml          # Step 3: VRF A configuration
│   ├── configure_vpnv4_bgp.yml      # Step 4: PE-to-PE VPNv4 iBGP
│   ├── configure_pe_ce_bgp.yml      # Step 5: PE-to-CE eBGP peering
│   ├── factory_reset.yml            # Reset all routers to clean state
│   ├── backup_config.yml            # Backup router configurations
│   ├── README-OSPF.md               # OSPF playbook documentation
│   ├── README-LDP.md                # LDP playbook documentation
│   ├── README-L3VPN.md              # L3VPN/VRF playbook documentation
│   ├── README-VPNV4-BGP.md          # VPNv4 BGP playbook documentation
│   ├── README-PE-CE-BGP.md          # PE-CE BGP playbook documentation
│   ├── PLAYBOOKS-README.md          # Overview of all playbooks
│   ├── host_vars/                   # Per-router configuration
│   ├── group_vars/                  # Common VyOS settings
│   ├── templates/                   # Jinja2 templates for configs
│   ├── verification/                # Verification and testing playbooks
│   └── archive/                     # Deprecated/old playbooks
└── backups/                         # Router config backups (gitignored)
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
