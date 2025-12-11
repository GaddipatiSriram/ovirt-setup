# oVirt Engine Standalone Deployment on Rocky Linux 9

Ansible playbooks for deploying oVirt Engine standalone with local databases on bare metal Rocky Linux 9.

**Reference:** https://www.ovirt.org/documentation/installing_ovirt_as_a_standalone_manager_with_local_databases/

## Prerequisites

- Clean Rocky Linux 9 minimal install (bare metal)
- Virtualization extensions enabled in BIOS (Intel VT-x/AMD-V)
- Working network configuration
- Root access
- Minimum: 4-core CPU, 16GB RAM, 50GB disk

## Quick Start

```bash
# 1. Install ansible-core
dnf install -y ansible-core

# 2. Install required collections
cd /root/ovirt-deploy
ansible-galaxy collection install -r requirements.yml

# 3. Verify ansible uses Python 3.9
ansible --version
# Should show: python version = 3.9.x

# 4. Run all playbooks
ansible-playbook -i inventory.ini site.yml

# Or run individually:
ansible-playbook -i inventory.ini prep_host.yml
ansible-playbook -i inventory.ini install_engine.yml
ansible-playbook -i inventory.ini post_install.yml
```

## Troubleshooting Ansible

If you hit module/import issues:

```bash
# Upgrade ansible-core via pip
python3.9 -m ensurepip
python3.9 -m pip install ansible-core==2.17.4

# Remove dnf ansible-core to avoid path conflicts
dnf remove ansible-core

# Verify and rerun
ansible --version
ansible-playbook -i inventory.ini site.yml
```

## Configuration

Edit `group_vars/all.yml` before running:

| Variable | Default | Description |
|----------|---------|-------------|
| `ovirt_engine_fqdn` | ovirt.engatwork.com | Engine FQDN |
| `ovirt_engine_admin_password` | unix | Admin portal password |
| `host_root_password` | unix | Root password for host enrollment |

## File Structure

```
ovirt-deploy/
├── inventory.ini          # Localhost inventory
├── group_vars/
│   └── all.yml           # Variables (passwords, FQDN, etc.)
├── templates/
│   ├── chrony.conf.j2    # NTP configuration
│   ├── engine-answers.conf.j2  # engine-setup answers
│   └── exports.j2        # NFS exports
├── prep_host.yml         # OS preparation
├── install_engine.yml    # Engine installation
├── post_install.yml      # Add host & storage
├── site.yml              # Master playbook
└── requirements.yml      # Ansible collections
```

## Playbook Details

### prep_host.yml
- Sets hostname to `node.engatwork.com`
- Configures `/etc/hosts`
- Sets SELinux permissive
- Configures chrony (NTP)
- Adds oVirt 4.5 repos
- Enables required dnf modules
- Installs virtualization packages
- Configures NFS storage directories
- Opens firewall ports

### install_engine.yml
- Installs `ovirt-engine` package
- Generates answer file from template
- Runs `engine-setup --accept-defaults`
- Starts oVirt services

### post_install.yml
- Installs host packages (vdsm, ovirt-host)
- Creates datacenter and cluster
- Adds localhost as hypervisor host
- Configures NFS storage domains

## Access oVirt

After installation:

- **Admin Portal:** https://ovirt.engatwork.com/ovirt-engine/
- **VM Portal:** https://ovirt.engatwork.com/ovirt-engine/web-ui
- **Username:** admin@internal
- **Password:** unix

## Important Notes

- **EL9 only** - Do not run on EL8 or EL10
- Engine FQDN must resolve via DNS or `/etc/hosts`
- IPv6 must remain enabled
- Host enrollment requires root password
# ovirt-setup
# ovirt-setup
# ovirt-setup
