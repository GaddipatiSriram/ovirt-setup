# pfSense Configuration Playbooks

Ansible playbooks for deploying and configuring pfSense as the network gateway/firewall for the oVirt environment.

## Overview

This directory contains playbooks to:
1. **Install** a new pfSense VM in oVirt
2. **Post-install** configuration (eject ISO, set boot order)
3. **Configure** pfSense with interfaces, DHCP, firewall rules, and NAT

## Prerequisites

```bash
# Install the pfsensible.core Ansible collection
ansible-galaxy collection install pfsensible.core
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `install_pfsense.yml` | Download ISO, create VM, attach NICs |
| `post_install_pfsense.yml` | Eject ISO, set boot order, start VM |
| `configure_pfsense.yml` | Configure interfaces, DHCP, firewall, NAT |

## Network Architecture

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                      pfSense                            │
                    │                   192.168.0.101                         │
                    ├─────────────────────────────────────────────────────────┤
   Internet ◄───────┤ WAN (vtnet0)                                            │
   192.168.0.1      │ 192.168.0.101/24                                        │
                    ├─────────────────────────────────────────────────────────┤
                    │                                                         │
                    │  MANAGEMENT NETWORKS                                    │
                    │  ├─ MGMT_CORE   (vtnet1) 10.10.0.1/24  VLAN 10         │
                    │  ├─ MGMT_DEVOPS (vtnet2) 10.10.1.1/24  VLAN 11         │
                    │  └─ MGMT_OBSERV (vtnet3) 10.10.2.1/24  VLAN 12         │
                    │                                                         │
                    │  DEVELOPMENT NETWORKS                                   │
                    │  ├─ DEV_APPS    (vtnet4) 10.20.0.1/24  VLAN 20         │
                    │  └─ DEV_DATA    (vtnet5) 10.20.1.1/24  VLAN 21         │
                    │                                                         │
                    │  PRODUCTION NETWORKS                                    │
                    │  ├─ PROD_APPS   (vtnet6) 10.30.0.1/24  VLAN 30         │
                    │  └─ PROD_DATA   (vtnet7) 10.30.1.1/24  VLAN 31         │
                    │                                                         │
                    └─────────────────────────────────────────────────────────┘
```

## Interface Details

| Interface | Device | IP Address | VLAN | Purpose |
|-----------|--------|------------|------|---------|
| WAN | vtnet0 | 192.168.0.101/24 | - | Internet uplink |
| MGMT_CORE | vtnet1 | 10.10.0.1/24 | 10 | DNS, NTP, LDAP, Bastion, VPN |
| MGMT_DEVOPS | vtnet2 | 10.10.1.1/24 | 11 | ArgoCD, Jenkins, GitLab, Harbor |
| MGMT_OBSERV | vtnet3 | 10.10.2.1/24 | 12 | Grafana, Prometheus, Loki |
| DEV_APPS | vtnet4 | 10.20.0.1/24 | 20 | Dev Web, App, API, K8s |
| DEV_DATA | vtnet5 | 10.20.1.1/24 | 21 | Dev PostgreSQL, Redis, Kafka |
| PROD_APPS | vtnet6 | 10.30.0.1/24 | 30 | Prod Web, App, API, K8s |
| PROD_DATA | vtnet7 | 10.30.1.1/24 | 31 | Prod PostgreSQL, Redis, Kafka |

## Interface Groups

| Group | Members | Purpose |
|-------|---------|---------|
| Management | mgmt_core, mgmt_devops, mgmt_observ | Infrastructure services |
| Development | dev_apps, dev_data | Development environment |
| Production | prod_apps, prod_data | Production environment |
| AllLANs | All 7 LAN interfaces | All internal networks |

## DHCP Configuration

All LAN interfaces have DHCP enabled with the following ranges:

| Interface | DHCP Range | Domain |
|-----------|------------|--------|
| MGMT_CORE | 10.10.0.100 - 10.10.0.200 | mgmt.engatwork.com |
| MGMT_DEVOPS | 10.10.1.100 - 10.10.1.200 | devops.engatwork.com |
| MGMT_OBSERV | 10.10.2.100 - 10.10.2.200 | observ.engatwork.com |
| DEV_APPS | 10.20.0.100 - 10.20.0.200 | dev.engatwork.com |
| DEV_DATA | 10.20.1.100 - 10.20.1.200 | devdata.engatwork.com |
| PROD_APPS | 10.30.0.100 - 10.30.0.200 | prod.engatwork.com |
| PROD_DATA | 10.30.1.100 - 10.30.1.200 | proddata.engatwork.com |

## Firewall Rules

### Security Policy

```
┌─────────────────────────────────────────────────────────────────┐
│                     FIREWALL POLICY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MANAGEMENT (Full Access)                                       │
│  └─► Can reach: Internet, Dev, Prod, All internal              │
│                                                                 │
│  DEVELOPMENT                                                    │
│  ├─ DEV_APPS → Internet ✓, Management ✓, Dev_Data ✓            │
│  └─ DEV_DATA → Internal only (no direct internet)              │
│                                                                 │
│  PRODUCTION                                                     │
│  ├─ PROD_APPS → Internet ✓, Management ✓, Prod_Data ✓          │
│  │              Development ✗ (blocked)                        │
│  └─ PROD_DATA → Internal only, Development ✗ (blocked)         │
│                                                                 │
│  KEY ISOLATION:                                                 │
│  • Production cannot reach Development networks                 │
│  • Data networks cannot reach Internet directly                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Firewall Aliases

| Alias | Type | Values |
|-------|------|--------|
| RFC1918 | network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| ManagementNets | network | 10.10.0.0/24, 10.10.1.0/24, 10.10.2.0/24 |
| DevelopmentNets | network | 10.20.0.0/24, 10.20.1.0/24 |
| ProductionNets | network | 10.30.0.0/24, 10.30.1.0/24 |
| WebPorts | port | 80, 443 |
| DatabasePorts | port | 5432, 3306, 27017, 6379, 9200 |

## Usage

### Step 1: Install pfSense VM

```bash
ansible-playbook playbooks/network/install_pfsense.yml
```

This will:
- Download pfSense CE ISO
- Create VM with 4GB RAM, 2 vCPUs, 50GB disk
- Attach 8 NICs (WAN + 7 LANs)
- Boot from ISO for manual installation

### Step 2: Install pfSense OS

1. Open VNC console in oVirt Admin Portal
2. Complete pfSense installation wizard
3. Reboot when prompted

### Step 3: Post-Installation

```bash
ansible-playbook playbooks/network/post_install_pfsense.yml
```

This will:
- Eject installation ISO
- Set boot order to HD only
- Restart VM
- Ensure all NICs are linked

### Step 4: Configure pfSense

```bash
# With default credentials (admin/pfsense)
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml

# With custom credentials
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml \
  -e pfsense_host=192.168.0.101 \
  -e pfsense_user=admin \
  -e pfsense_password=YourPassword
```

### Run Specific Phases

```bash
# Configure only interfaces
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml --tags interfaces

# Configure only DHCP
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml --tags dhcp

# Configure only firewall rules
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml --tags rules

# Configure only NAT
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml --tags nat
```

## Configuration Files

```
pfsense/
├── README.md                    # This file
├── configure_pfsense.yml        # Main configuration playbook
└── vars/
    └── pfsense_config.yml       # All configuration variables
```

## Customization

Edit `vars/pfsense_config.yml` to customize:

- **System settings**: hostname, domain, DNS servers, timezone
- **Interface IPs**: Change subnet addressing
- **DHCP ranges**: Adjust pool sizes
- **Firewall aliases**: Add custom network/port groups
- **Firewall rules**: Modify access policies

## VM Specifications

| Resource | Value |
|----------|-------|
| Memory | 4 GB |
| vCPUs | 2 |
| Disk | 50 GB |
| NICs | 8 (virtio) |
| OS | pfSense CE 2.7.2 |
| BIOS | Q35 SeaBIOS |

## Accessing pfSense

| Method | URL/Address |
|--------|-------------|
| WebGUI | https://192.168.0.101 |
| SSH | ssh admin@192.168.0.101 |
| Console | VNC via oVirt Admin Portal |

## Troubleshooting

### Check VM Status
```bash
vdsm-client VM getStats vmID=62c11410-c9c6-4173-b0bf-55c3bfab9534
```

### Verify Network Connectivity
```bash
ping 192.168.0.101
```

### Check Bridge Connections
```bash
bridge link show | grep vnet
```

### View pfSense Logs
Access via WebGUI: Status → System Logs
