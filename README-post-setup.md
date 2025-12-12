# oVirt Post-Installation Configuration

This document covers the configuration performed after the base oVirt Engine installation, including networking, storage, and VM setup.

## Table of Contents

- [Environment Overview](#environment-overview)
- [Storage Domains](#storage-domains)
- [Network Configuration](#network-configuration)
- [pfSense Firewall VM](#pfsense-firewall-vm)
- [Playbook Reference](#playbook-reference)
- [Troubleshooting](#troubleshooting)

---

## Environment Overview

| Component | Value |
|-----------|-------|
| oVirt Version | 4.5.6 |
| OS | AlmaLinux 9.7 |
| Host | node.engatwork.com |
| Engine FQDN | ovirt.engatwork.com |
| Datacenter | Default |
| Cluster | Default |
| CPU | Intel Xeon Gold 6148 (80 vCPUs) |
| RAM | 251 GB |

### Architecture

Single-host setup with VLAN-based network isolation. All VMs run on the Default cluster with pfSense providing inter-VLAN routing and firewall functionality.

---

## Storage Domains

| Domain | Type | Path/LUN | Purpose |
|--------|------|----------|---------|
| data_lun | Data (FCP) | `3600062b202f02e0030cd23b26925a504` | VM disks, ISOs |
| backup_export | Export (NFS) | `/home/iso` | VM export/import, backups |

### Notes

- **data_lun**: Primary storage for all VM disks and uploaded ISOs
- **backup_export**: Originally named `iso_local`, renamed to reflect actual purpose (Export domain, not ISO domain)
- ISOs are uploaded as disks to `data_lun` with `content_type: iso`

---

## Network Configuration

### Single-Host VLAN Architecture

Network isolation is achieved through VLANs, not separate clusters. All networks are assigned to the Default cluster and attached to the host NIC (`eno1`).

### VLAN Networks

#### Management Networks (VLAN 10-16)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Mgmt-Core-Net | 10 | 10.10.0.0/24 | 10.10.0.1 | pfSense, CoreDNS, NTP |
| Mgmt-DMZ-Net | 11 | 10.10.1.0/24 | 10.10.1.1 | Bastion, VPN Gateway |
| Mgmt-Observability-Net | 12 | 10.10.2.0/24 | 10.10.2.1 | Grafana, Prometheus, ELK |
| Mgmt-DevOps-Net | 13 | 10.10.3.0/24 | 10.10.3.1 | ArgoCD, Jenkins |
| Mgmt-Identity-Net | 14 | 10.10.4.0/24 | 10.10.4.1 | LDAP, Keycloak |
| Mgmt-Security-Net | 15 | 10.10.5.0/24 | 10.10.5.1 | IDS/IPS, SIEM |
| Mgmt-Backup-Net | 16 | 10.10.6.0/24 | 10.10.6.1 | Veeam, Restic |

#### Development Networks (VLAN 21-25)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Dev-DMZ-Net | 21 | 10.20.1.0/24 | 10.20.1.1 | Load Balancer, Reverse Proxy |
| Dev-Web-Net | 22 | 10.20.2.0/24 | 10.20.2.1 | Nginx, Apache |
| Dev-App-Net | 23 | 10.20.3.0/24 | 10.20.3.1 | Node.js, Java |
| Dev-Database-Net | 24 | 10.20.4.0/24 | 10.20.4.1 | PostgreSQL, MySQL |
| Dev-Messaging-Net | 25 | 10.20.5.0/24 | 10.20.5.1 | Kafka, RabbitMQ |

#### Production Networks (VLAN 31-39)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Prod-DMZ-Net | 31 | 10.30.1.0/24 | 10.30.1.1 | LB, WAF, Proxy |
| Prod-Web-Net | 32 | 10.30.2.0/24 | 10.30.2.1 | Nginx, Apache |
| Prod-App-Net | 33 | 10.30.3.0/24 | 10.30.3.1 | Node.js, Java |
| Prod-Database-Net | 34 | 10.30.4.0/24 | 10.30.4.1 | PostgreSQL, MySQL |
| Prod-Messaging-Net | 35 | 10.30.5.0/24 | 10.30.5.1 | Kafka, RabbitMQ |
| Prod-Cache-Net | 36 | 10.30.6.0/24 | 10.30.6.1 | Redis, Memcached |
| Prod-API-Net | 37 | 10.30.7.0/24 | 10.30.7.1 | API Gateway, Service Mesh |
| Prod-DR-Net | 39 | 10.30.9.0/24 | 10.30.9.1 | Disaster Recovery |

### Network Setup Command

```bash
ansible-playbook -i inventory.ini create_networks.yml
```

This playbook:
1. Creates all VLAN networks in the datacenter
2. Assigns networks to Default cluster (required=false)
3. Attaches networks to host NIC (creates eno1.XX interfaces)

---

## pfSense Firewall VM

Central firewall/router VM that handles inter-VLAN routing and security.

### VM Specifications

| Setting | Value |
|---------|-------|
| Name | pfSense |
| Memory | 4 GB (guaranteed), 8 GB (max) |
| CPU | 1 socket x 2 cores |
| Disk | 50 GB on data_lun |
| BIOS | i440fx SeaBIOS |
| Display | VNC (virtio-vga) |
| OS | FreeBSD (pfSense CE 2.7.2) |

### Network Interfaces (21 vNICs)

| NIC | Network | VLAN | Gateway IP |
|-----|---------|------|------------|
| nic1 | ovirtmgmt | - | WAN (DHCP/Static) |
| nic2 | Mgmt-Core-Net | 10 | 10.10.0.1 |
| nic3 | Mgmt-DMZ-Net | 11 | 10.10.1.1 |
| nic4 | Mgmt-Observability-Net | 12 | 10.10.2.1 |
| nic5 | Mgmt-DevOps-Net | 13 | 10.10.3.1 |
| nic6 | Mgmt-Identity-Net | 14 | 10.10.4.1 |
| nic7 | Mgmt-Security-Net | 15 | 10.10.5.1 |
| nic8 | Mgmt-Backup-Net | 16 | 10.10.6.1 |
| nic9 | Dev-DMZ-Net | 21 | 10.20.1.1 |
| nic10 | Dev-Web-Net | 22 | 10.20.2.1 |
| nic11 | Dev-App-Net | 23 | 10.20.3.1 |
| nic12 | Dev-Database-Net | 24 | 10.20.4.1 |
| nic13 | Dev-Messaging-Net | 25 | 10.20.5.1 |
| nic14 | Prod-DMZ-Net | 31 | 10.30.1.1 |
| nic15 | Prod-Web-Net | 32 | 10.30.2.1 |
| nic16 | Prod-App-Net | 33 | 10.30.3.1 |
| nic17 | Prod-Database-Net | 34 | 10.30.4.1 |
| nic18 | Prod-Messaging-Net | 35 | 10.30.5.1 |
| nic19 | Prod-Cache-Net | 36 | 10.30.6.1 |
| nic20 | Prod-API-Net | 37 | 10.30.7.1 |
| nic21 | Prod-DR-Net | 39 | 10.30.9.1 |

### pfSense Setup Commands

```bash
# Download pfSense ISO
ansible-playbook -i inventory.ini download_pfsense.yml

# Configure pfSense VM (create VM + attach NICs)
ansible-playbook -i inventory.ini configure_pfsense_vm.yml

# Or just add NICs to existing VM
ansible-playbook -i inventory.ini configure_pfsense_nics.yml
```

### pfSense Post-Installation

After installing pfSense from the ISO:

1. **Assign WAN interface** (vtnet0 -> nic1/ovirtmgmt)
2. **Configure WAN IP** (DHCP or static)
3. **Assign LAN interfaces** (vtnet1-vtnet20)
4. **Configure gateway IPs** for each interface
5. **Set up firewall rules** for inter-VLAN traffic
6. **Enable SSH** (option 14) for remote management

---

## Playbook Reference

| Playbook | Purpose | Command |
|----------|---------|---------|
| `create_networks.yml` | Create 20 VLAN networks | `ansible-playbook -i inventory.ini create_networks.yml` |
| `download_pfsense.yml` | Download pfSense ISO to data_lun | `ansible-playbook -i inventory.ini download_pfsense.yml` |
| `configure_pfsense_vm.yml` | Create pfSense VM with all settings | `ansible-playbook -i inventory.ini configure_pfsense_vm.yml` |
| `configure_pfsense_nics.yml` | Add 21 vNICs to pfSense | `ansible-playbook -i inventory.ini configure_pfsense_nics.yml` |
| `clear_events.yml` | Clear all oVirt events | `ansible-playbook -i inventory.ini clear_events.yml` |

---

## Troubleshooting

### VM Won't Start - Video Device Issues

If VM fails with QXL/video errors, update the video device:

```bash
# Set to virtio-vga (recommended)
PGPASSWORD=unix psql -U engine -d engine -h localhost -c \
  "UPDATE vm_device SET device = 'virtio', spec_params = '{}'
   WHERE vm_id = (SELECT vm_guid FROM vm_static WHERE vm_name = 'pfSense')
   AND type = 'video';"
```

### VM Won't Start - Smartcard/SPICE Errors

Remove incompatible devices:

```bash
PGPASSWORD=unix psql -U engine -d engine -h localhost -c \
  "DELETE FROM vm_device
   WHERE vm_id = (SELECT vm_guid FROM vm_static WHERE vm_name = 'pfSense')
   AND device IN ('smartcard', 'spicevmc');"
```

### noVNC Console - Connection Closed

1. Accept the SSL certificate:
   - Open `https://ovirt.engatwork.com:6100` in browser
   - Accept the certificate warning
   - Return to oVirt console

2. Restart websocket proxy:
   ```bash
   systemctl restart ovirt-websocket-proxy
   ```

### noVNC Console - No Keyboard/Mouse

Fix USB controller:

```bash
PGPASSWORD=unix psql -U engine -d engine -h localhost -c \
  "UPDATE vm_device SET spec_params = '{\"index\": \"0\", \"model\": \"qemu-xhci\"}'
   WHERE vm_id = (SELECT vm_guid FROM vm_static WHERE vm_name = 'pfSense')
   AND type = 'controller' AND device = 'usb';"
```

Then restart the VM.

### Hot-Plug NIC Limit

Can only hot-plug ~11 NICs to a running VM. Stop VM first to add more:

```bash
# In playbook, VM is stopped automatically
ansible-playbook -i inventory.ini configure_pfsense_nics.yml
```

### Check VM Errors

```bash
PGPASSWORD=unix psql -U engine -d engine -h localhost -c \
  "SELECT message FROM audit_log WHERE vm_name = 'pfSense'
   ORDER BY log_time DESC LIMIT 5;"
```

---

## File Structure

```
ovirt-setup/
├── inventory.ini
├── group_vars/
│   └── all.yml
├── templates/
│   ├── chrony.conf.j2
│   ├── engine-answers.conf.j2
│   └── exports.j2
├── prep_host.yml              # OS preparation
├── install_engine.yml         # Engine installation
├── post_install.yml           # Add host & storage
├── site.yml                   # Master playbook
├── create_networks.yml        # VLAN network setup
├── download_pfsense.yml       # Download pfSense ISO
├── configure_pfsense_vm.yml   # Create pfSense VM
├── configure_pfsense_nics.yml # Add NICs to pfSense
├── clear_events.yml           # Clear audit log
├── requirements.yml
├── README.md                  # Base installation docs
├── README-network-architecture.md  # Network diagrams
├── README-post-setup.md       # This file
└── MANUAL_NOTES.md
```
