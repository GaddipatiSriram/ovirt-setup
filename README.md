# oVirt Infrastructure Deployment

Ansible playbooks for deploying oVirt Engine, networking (pfSense), and compute resources on Rocky Linux 9.

## Overview

This project automates the complete setup of an oVirt virtualization environment, including:
- oVirt Engine installation and configuration
- Network infrastructure (VLANs, pfSense firewall/router)
- VM templates and provisioning
- Storage configuration

## Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                       Physical Host (Rocky Linux 9)                         │
│                         node.engatwork.com                                  │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                        oVirt Engine                                │     │
│  │                     ovirt.engatwork.com                           │     │
│  │                       (Hosted Engine)                             │     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  Management Networks (VLAN 10-14):                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │Mgmt-Core │ │Mgmt-DevOp│ │Mgmt-Obsrv│ │Mgmt-Strg │ │Mgmt-Forge│         │
│  │10.10.1.0 │ │10.10.2.0 │ │10.10.3.0 │ │10.10.4.0 │ │10.10.5.0 │         │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘         │
│                                                                             │
│  Dev Networks (VLAN 20-22):        Prod Networks (VLAN 30-32):             │
│  ┌──────────┐ ┌──────────┐ ┌─────┐ ┌──────────┐ ┌──────────┐ ┌─────┐      │
│  │ Dev-Web  │ │ Dev-Apps │ │Data │ │ Prod-Web │ │Prod-Apps │ │Data │      │
│  │10.20.1.0 │ │10.20.2.0 │ │.3.0 │ │10.30.1.0 │ │10.30.2.0 │ │.3.0 │      │
│  └──────────┘ └──────────┘ └─────┘ └──────────┘ └──────────┘ └─────┘      │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────┐     │
│  │                         pfSense VM                                 │     │
│  │                      Gateway & Firewall                            │     │
│  │                                                                    │     │
│  │  WAN: 192.168.0.101 ←→ Internet     12 Interfaces (1 WAN + 11 LAN)│     │
│  └───────────────────────────────────────────────────────────────────┘     │
│                                                                             │
│  Storage: Direct LUN (iSCSI/FC)                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
ovirt-setup/
├── ansible.cfg                     # Ansible configuration
├── requirements.yml                # Ansible collection dependencies
├── README.md
├── collections/                    # Local Ansible collections (gitignored)
├── inventory/
│   ├── hosts.ini                   # Host inventory
│   └── group_vars/
│       └── all.yml                 # Global variables
├── playbooks/
│   ├── ovirt/                      # oVirt Engine setup
│   │   └── site.yml                # Main oVirt playbook
│   ├── network/                    # Network & pfSense
│   │   ├── create_networks.yml     # Create oVirt networks
│   │   └── pfsense/                # pfSense configuration
│   │       └── configure_pfsense.yml
│   └── compute/                    # VMs & templates
│       ├── create_template.yml     # Create VM templates
│       └── deploy_vms.yml          # Deploy VMs from templates
├── tasks/                          # Reusable task files
├── templates/                      # Jinja2 templates
└── docs/                           # Documentation
    ├── network-architecture.md
    └── post-setup.md
```

## Prerequisites

- Rocky Linux 9 minimal install (bare metal)
- Virtualization extensions enabled (Intel VT-x/AMD-V)
- Minimum: 4-core CPU, 16GB RAM, 50GB disk
- Root access
- Direct LUN storage (optional but recommended)

## Quick Start

```bash
# 1. Install Ansible
dnf install -y ansible-core

# 2. Install required collections
cd /root/learning/ovirt-setup
ansible-galaxy collection install -r requirements.yml -p ./collections

# 3. Configure variables
vi inventory/group_vars/all.yml

# 4. Deploy oVirt Engine
ansible-playbook playbooks/ovirt/site.yml

# 5. Setup networks
ansible-playbook playbooks/network/create_networks.yml

# 6. Configure pfSense
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml

# 7. Deploy VMs
ansible-playbook playbooks/compute/create_template.yml
ansible-playbook playbooks/compute/deploy_vms.yml
```

## Network Configuration

### VLANs (11 Networks)

| Network | VLAN | CIDR | Gateway | Purpose |
|---------|------|------|---------|---------|
| Mgmt-Core-Net | 10 | 10.10.1.0/24 | 10.10.1.1 | Foundational (DNS, Keycloak, Vault) |
| Mgmt-DevOps-Net | 11 | 10.10.2.0/24 | 10.10.2.1 | OKD cluster (ArgoCD, OCM) |
| Mgmt-Observ-Net | 12 | 10.10.3.0/24 | 10.10.3.1 | Monitoring (Prometheus, Grafana) |
| Mgmt-Storage-Net | 13 | 10.10.4.0/24 | 10.10.4.1 | Storage (Rook-Ceph) |
| Mgmt-Forge-Net | 14 | 10.10.5.0/24 | 10.10.5.1 | Software forge (GitLab, Harbor) |
| Dev-Web-Net | 20 | 10.20.1.0/24 | 10.20.1.1 | Dev web tier |
| Dev-Apps-Net | 21 | 10.20.2.0/24 | 10.20.2.1 | Dev app tier |
| Dev-Data-Net | 22 | 10.20.3.0/24 | 10.20.3.1 | Dev data tier |
| Prod-Web-Net | 30 | 10.30.1.0/24 | 10.30.1.1 | Prod web tier |
| Prod-Apps-Net | 31 | 10.30.2.0/24 | 10.30.2.1 | Prod app tier |
| Prod-Data-Net | 32 | 10.30.3.0/24 | 10.30.3.1 | Prod data tier |

### pfSense Configuration

The pfSense playbook configures:

```yaml
# DNS Resolver - Forward internal domains to CoreDNS
domainoverrides:
  - domain: engatwork.com
    ip: 10.10.1.200  # CoreDNS on mgmt-core cluster

# Access Control - Allow DNS queries from internal networks
acls:
  - 10.10.0.0/16 allow
  - 192.168.0.0/24 allow

# Firewall Rules
# - Allow inter-VLAN routing
# - NAT outbound to WAN
```

## Configuration Variables

Edit `inventory/group_vars/all.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `ovirt_engine_fqdn` | ovirt.engatwork.com | Engine FQDN |
| `ovirt_engine_admin_password` | unix | Admin portal password |
| `host_root_password` | unix | Host enrollment password |
| `ovirt_storage_type` | lun | Storage type (lun/none) |
| `ovirt_lun_id` | (set yours) | Direct LUN ID from `multipath -ll` |

## Playbook Categories

| Category | Path | Description |
|----------|------|-------------|
| **oVirt** | `playbooks/ovirt/` | Engine installation, host setup, storage |
| **Network** | `playbooks/network/` | VLAN networks, pfSense firewall |
| **Compute** | `playbooks/compute/` | Cloud images, templates, VM deployment |

## Ansible Collections Used

| Collection | Purpose |
|------------|---------|
| `ovirt.ovirt` | oVirt API operations |
| `pfsensible.core` | pfSense configuration |
| `community.general` | General utilities |
| `ansible.posix` | POSIX operations |

## Access After Deployment

| Service | URL | Credentials |
|---------|-----|-------------|
| oVirt Admin Portal | https://ovirt.engatwork.com/ovirt-engine/ | admin@ovirt / unix |
| oVirt VM Portal | https://ovirt.engatwork.com/ovirt-engine/web-ui | admin@ovirt / unix |
| pfSense WebUI | https://10.10.2.1/ | admin / pfsense |

## Troubleshooting

### oVirt Engine Issues
```bash
# Check hosted-engine status
hosted-engine --vm-status

# Check engine service
systemctl status ovirt-engine

# Engine logs
journalctl -u ovirt-engine -f
```

### pfSense SSH Access
```bash
# Use internal LAN IP, not WAN
ssh admin@10.10.2.1

# Or via console in oVirt
```

### Network Connectivity
```bash
# Test from VM to gateway
ping 10.10.2.1

# Test DNS resolution
dig @10.10.1.200 api.okd.engatwork.com
```

## Documentation

- [Network Architecture](docs/network-architecture.md) - VLAN design, pfSense interfaces
- [Post Setup Guide](docs/post-setup.md) - Manual configuration steps

## Security Notes

- Store passwords in Ansible Vault for production
- pfSense WAN uses NAT for outbound traffic
- Inter-VLAN routing controlled by pfSense firewall rules
- oVirt uses self-signed certificates by default

## Requirements

- Rocky Linux 9 / RHEL 9 / AlmaLinux 9
- oVirt 4.5+
- Ansible 2.14+
- Python 3.9+

## License

MIT
