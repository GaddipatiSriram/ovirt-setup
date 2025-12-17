# oVirt Engine Playbooks

Playbooks for installing and configuring oVirt Engine standalone with local databases.

**Reference:** https://www.ovirt.org/documentation/installing_ovirt_as_a_standalone_manager_with_local_databases/

## Playbooks

| Playbook | Description |
|----------|-------------|
| `site.yml` | Master playbook - runs all in sequence |
| `prep_host.yml` | Prepare host OS for oVirt |
| `install_engine.yml` | Install oVirt Engine |
| `post_install.yml` | Add host and storage domains |
| `clear_events.yml` | Clear oVirt event notifications |

## Usage

```bash
# Run all (recommended for fresh install)
ansible-playbook playbooks/ovirt/site.yml

# Or run individually
ansible-playbook playbooks/ovirt/prep_host.yml
ansible-playbook playbooks/ovirt/install_engine.yml
ansible-playbook playbooks/ovirt/post_install.yml
```

## Playbook Details

### prep_host.yml

Prepares Rocky Linux 9 for oVirt Engine:

- Sets hostname to `node.engatwork.com`
- Configures `/etc/hosts` with FQDN
- Sets SELinux to permissive
- Configures chrony (NTP)
- Adds oVirt 4.5 repositories
- Enables required DNF modules (postgresql:12, nodejs:14)
- Installs virtualization packages (qemu-kvm, libvirt)
- Configures NFS storage directories
- Opens firewall ports

### install_engine.yml

Installs oVirt Engine:

- Installs `ovirt-engine` package
- Generates answer file from template
- Runs `engine-setup --accept-defaults`
- Starts and enables oVirt services:
  - ovirt-engine
  - ovirt-engine-dwhd
  - ovirt-imageio
  - ovirt-websocket-proxy

### post_install.yml

Post-installation configuration:

- Installs host packages (vdsm, ovirt-host)
- Creates Default datacenter and cluster
- Adds localhost as hypervisor host
- Configures Direct LUN storage domain
- Configures NFS export domain (optional)
- Creates initial engine backup
- Sets noVNC as default console

### clear_events.yml

Utility to clear oVirt event notifications:

```bash
ansible-playbook playbooks/ovirt/clear_events.yml
```

## Variables

Key variables in `inventory/group_vars/all.yml`:

```yaml
ovirt_engine_fqdn: "ovirt.engatwork.com"
ovirt_engine_admin_password: "unix"
host_root_password: "unix"

# Storage
ovirt_storage_type: "lun"
ovirt_lun_id: "3600062b202f02e0030cd23b26925a504"
ovirt_lun_storage_name: "data_lun"
```

## Post-Installation

After running playbooks:

1. Access Admin Portal: https://ovirt.engatwork.com/ovirt-engine/
2. Login: admin@ovirt / unix
3. Wait for host to reach "Up" status
4. Verify storage domain is active

**Important:** Reboot the system after post_install.yml completes.
