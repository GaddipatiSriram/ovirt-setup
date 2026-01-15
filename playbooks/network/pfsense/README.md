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
   Internet ◄───────┤ WAN (em0)                                               │
   192.168.0.1      │ 192.168.0.101/24                                        │
                    ├─────────────────────────────────────────────────────────┤
                    │                                                         │
                    │  MANAGEMENT NETWORKS (VLAN 10-14)                       │
                    │  ├─ MGMT_CORE    (vtnet0) 10.10.1.1/24  VLAN 10        │
                    │  ├─ MGMT_DEVOPS  (vtnet1) 10.10.2.1/24  VLAN 11        │
                    │  ├─ MGMT_OBSERV  (vtnet2) 10.10.3.1/24  VLAN 12        │
                    │  ├─ MGMT_STORAGE (vtnet3) 10.10.4.1/24  VLAN 13        │
                    │  └─ MGMT_FORGE   (vtnet4) 10.10.5.1/24  VLAN 14        │
                    │                                                         │
                    │  DEVELOPMENT NETWORKS (VLAN 20-22)                      │
                    │  ├─ DEV_WEB      (vtnet5) 10.20.1.1/24  VLAN 20        │
                    │  ├─ DEV_APPS     (vtnet6) 10.20.2.1/24  VLAN 21        │
                    │  └─ DEV_DATA     (vtnet7) 10.20.3.1/24  VLAN 22        │
                    │                                                         │
                    │  PRODUCTION NETWORKS (VLAN 30-32)                       │
                    │  ├─ PROD_WEB     (vtnet8)  10.30.1.1/24  VLAN 30       │
                    │  ├─ PROD_APPS    (vtnet9)  10.30.2.1/24  VLAN 31       │
                    │  └─ PROD_DATA    (vtnet10) 10.30.3.1/24  VLAN 32       │
                    │                                                         │
                    └─────────────────────────────────────────────────────────┘
```

## Interface Details

| Interface | Device | oVirt NIC | IP Address | VLAN | Purpose |
|-----------|--------|-----------|------------|------|---------|
| WAN | em0 | nic1 | 192.168.0.101/24 | - | Internet uplink (e1000) |
| MGMT_CORE | vtnet0 | nic2 | 10.10.1.1/24 | 10 | CoreDNS, Keycloak, Vault |
| MGMT_DEVOPS | vtnet1 | nic3 | 10.10.2.1/24 | 11 | OKD, ArgoCD, OCM, Backstage |
| MGMT_OBSERV | vtnet2 | nic4 | 10.10.3.1/24 | 12 | Prometheus, Grafana, Loki |
| MGMT_STORAGE | vtnet3 | nic5 | 10.10.4.1/24 | 13 | Rook-Ceph, NFS, S3, RBD |
| MGMT_FORGE | vtnet4 | nic6 | 10.10.5.1/24 | 14 | GitLab, Harbor, SonarQube |
| DEV_WEB | vtnet5 | nic7 | 10.20.1.1/24 | 20 | Dev Ingress, nginx, HAProxy |
| DEV_APPS | vtnet6 | nic8 | 10.20.2.1/24 | 21 | Dev Microservices, K8s |
| DEV_DATA | vtnet7 | nic9 | 10.20.3.1/24 | 22 | Dev PostgreSQL, Redis, Kafka |
| PROD_WEB | vtnet8 | nic10 | 10.30.1.1/24 | 30 | Prod Ingress, nginx, HAProxy |
| PROD_APPS | vtnet9 | nic11 | 10.30.2.1/24 | 31 | Prod Microservices, K8s |
| PROD_DATA | vtnet10 | nic12 | 10.30.3.1/24 | 32 | Prod PostgreSQL, Redis, Kafka |

## Interface Groups

| Group | Members | Purpose |
|-------|---------|---------|
| Management | mgmt_core, mgmt_devops, mgmt_observ, mgmt_storage, mgmt_forge | Infrastructure services |
| Development | dev_web, dev_apps, dev_data | Development environment |
| Production | prod_web, prod_apps, prod_data | Production environment |
| WebTier | dev_web, prod_web | Web tier networks |
| AppsTier | dev_apps, prod_apps | Application tier networks |
| DataTier | dev_data, prod_data | Data tier networks |
| AllLANs | All 11 LAN interfaces | All internal networks |

## DHCP Configuration

All LAN interfaces have DHCP enabled with the following ranges:

| Interface | DHCP Range | Domain |
|-----------|------------|--------|
| MGMT_CORE | 10.10.1.100 - 10.10.1.200 | engatwork.com |
| MGMT_DEVOPS | 10.10.2.100 - 10.10.2.200 | engatwork.com |
| MGMT_OBSERV | 10.10.3.100 - 10.10.3.200 | engatwork.com |
| MGMT_STORAGE | 10.10.4.100 - 10.10.4.200 | engatwork.com |
| MGMT_FORGE | 10.10.5.100 - 10.10.5.200 | engatwork.com |
| DEV_WEB | 10.20.1.100 - 10.20.1.200 | engatwork.com |
| DEV_APPS | 10.20.2.100 - 10.20.2.200 | engatwork.com |
| DEV_DATA | 10.20.3.100 - 10.20.3.200 | engatwork.com |
| PROD_WEB | 10.30.1.100 - 10.30.1.200 | engatwork.com |
| PROD_APPS | 10.30.2.100 - 10.30.2.200 | engatwork.com |
| PROD_DATA | 10.30.3.100 - 10.30.3.200 | engatwork.com |

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
│  DEVELOPMENT (3-Tier)                                           │
│  ├─ DEV_WEB  → Internet ✓, Apps ✓, Data ✗ (via Apps only)      │
│  ├─ DEV_APPS → Internet ✓, Data ✓, Management ✓                │
│  └─ DEV_DATA → Internal only (no direct internet)              │
│                                                                 │
│  PRODUCTION (3-Tier, Isolated)                                  │
│  ├─ PROD_WEB  → Internet ✓, Apps ✓, Data ✗, Development ✗      │
│  ├─ PROD_APPS → Internet ✓, Data ✓, Development ✗              │
│  └─ PROD_DATA → Internal only, Development ✗ (blocked)         │
│                                                                 │
│  KEY ISOLATION:                                                 │
│  • Production cannot reach Development networks                 │
│  • Data networks cannot reach Internet directly                 │
│  • Web tier cannot reach Data tier directly                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Firewall Aliases

| Alias | Type | Values |
|-------|------|--------|
| RFC1918 | network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 |
| ManagementNets | network | 10.10.1.0/24-10.10.5.0/24 (5 networks) |
| DevelopmentNets | network | 10.20.1.0/24-10.20.3.0/24 (3 networks) |
| ProductionNets | network | 10.30.1.0/24-10.30.3.0/24 (3 networks) |
| WebPorts | port | 80, 443 |
| AppPorts | port | 8080, 8443 |
| DatabasePorts | port | 5432, 3306, 27017, 6379, 9200, 9092 |

## Usage

### Step 1: Install pfSense VM

```bash
ansible-playbook playbooks/network/install_pfsense.yml
```

This will:
- Download pfSense CE ISO
- Create VM with 4GB RAM, 2 vCPUs, 50GB disk
- Attach 12 NICs (WAN e1000 + 11 LANs virtio)
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
├── configure_dns.yml            # DNS configuration playbook
├── inventory.yml                # pfSense inventory
└── vars/
    ├── pfsense_config.yml       # All configuration variables
    └── dns_config.yml           # DNS configuration variables
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
| NICs | 12 (1 e1000 + 11 virtio) |
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
vdsm-client VM getStats vmID=<vm-id>
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
