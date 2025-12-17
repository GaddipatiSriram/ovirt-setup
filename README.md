# oVirt Infrastructure Deployment

Ansible playbooks for deploying oVirt Engine, networking (pfSense), and compute resources on Rocky Linux 9.

## Project Structure

```
ovirt-setup/
├── ansible.cfg                     # Ansible configuration
├── inventory/
│   ├── hosts.ini                   # Host inventory
│   └── group_vars/
│       └── all.yml                 # Global variables
├── playbooks/
│   ├── ovirt/                      # oVirt Engine setup
│   ├── network/                    # Network & pfSense
│   └── compute/                    # VMs & templates
├── collections/
│   └── requirements.yml            # Ansible collections
├── templates/                      # Jinja2 templates
├── tasks/                          # Reusable task files
└── docs/                           # Documentation
```

## Prerequisites

- Rocky Linux 9 minimal install (bare metal)
- Virtualization extensions enabled (Intel VT-x/AMD-V)
- Minimum: 4-core CPU, 16GB RAM, 50GB disk
- Root access

## Quick Start

```bash
# 1. Install Ansible
dnf install -y ansible-core

# 2. Install required collections
cd /root/ovirt-setup
ansible-galaxy collection install -r collections/requirements.yml

# 3. Configure variables
vi inventory/group_vars/all.yml

# 4. Deploy oVirt Engine
ansible-playbook playbooks/ovirt/site.yml

# 5. Setup networks
ansible-playbook playbooks/network/create_networks.yml
ansible-playbook playbooks/network/configure_pfsense_vm.yml

# 6. Deploy VMs
ansible-playbook playbooks/compute/create_template.yml
ansible-playbook playbooks/compute/deploy_vms.yml
```

## Playbook Categories

| Category | Path | Description |
|----------|------|-------------|
| **oVirt** | `playbooks/ovirt/` | Engine installation, host setup, storage |
| **Network** | `playbooks/network/` | VLAN networks, pfSense firewall |
| **Compute** | `playbooks/compute/` | Cloud images, templates, VM deployment |

See README.md in each playbook directory for details.

## Configuration

Edit `inventory/group_vars/all.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `ovirt_engine_fqdn` | ovirt.engatwork.com | Engine FQDN |
| `ovirt_engine_admin_password` | unix | Admin portal password |
| `host_root_password` | unix | Host enrollment password |
| `ovirt_storage_type` | lun | Storage type (lun/none) |
| `ovirt_lun_id` | (set yours) | Direct LUN ID from `multipath -ll` |

## Access

After deployment:

| Service | URL |
|---------|-----|
| Admin Portal | https://ovirt.engatwork.com/ovirt-engine/ |
| VM Portal | https://ovirt.engatwork.com/ovirt-engine/web-ui |
| pfSense | https://192.168.0.101/ |

**Credentials:** admin@ovirt / unix

## Documentation

- [Network Architecture](docs/network-architecture.md) - VLAN design, pfSense interfaces
- [Post Setup Guide](docs/post-setup.md) - Manual configuration steps

## Requirements

- Rocky Linux 9 / RHEL 9 / AlmaLinux 9
- oVirt 4.5
- Ansible 2.14+
