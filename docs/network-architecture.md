# Network Architecture (Simplified)

## Overview

This document describes the simplified network architecture for the oVirt virtualization environment.
- **3 Management Networks** - Core infrastructure, DevOps, Observability
- **2 Development Networks** - Apps and Data
- **2 Production Networks** - Apps and Data
- **Total: 7 networks + WAN = 8 pfSense interfaces**

## Network Diagram

```
                                            INTERNET
                                                │
                                         192.168.0.1 (Gateway)
                                                │
                                                ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              DEFAULT DATACENTER                                       │
│                           (Host: node, Storage: data_lun)                            │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│                         ┌─────────────────────────────────┐                          │
│                         │           pfSense               │                          │
│                         │      (Central Firewall)         │                          │
│                         │      192.168.0.101 (WAN)        │                          │
│                         ├─────────────────────────────────┤                          │
│                         │  WAN ─────────── 192.168.0.101  │                          │
│                         │  MGMT_CORE ───── 10.10.0.1      │                          │
│                         │  MGMT_DEVOPS ─── 10.10.1.1      │                          │
│                         │  MGMT_OBSERV ─── 10.10.2.1      │                          │
│                         │  DEV_APPS ────── 10.20.0.1      │                          │
│                         │  DEV_DATA ────── 10.20.1.1      │                          │
│                         │  PROD_APPS ───── 10.30.0.1      │                          │
│                         │  PROD_DATA ───── 10.30.1.1      │                          │
│                         └───────────────┬─────────────────┘                          │
│                                         │                                            │
│       ┌─────────────────────────────────┼─────────────────────────────────┐          │
│       │                                 │                                 │          │
│       ▼                                 ▼                                 ▼          │
│  ┌─────────────┐                 ┌─────────────┐                 ┌─────────────┐     │
│  │    MGMT     │                 │     DEV     │                 │    PROD     │     │
│  │ 10.10.0.0/16│                 │ 10.20.0.0/16│                 │ 10.30.0.0/16│     │
│  └──────┬──────┘                 └──────┬──────┘                 └──────┬──────┘     │
│         │                               │                               │            │
│         ▼                               ▼                               ▼            │
│  ┌─────────────────────┐    ┌─────────────────────┐    ┌─────────────────────┐      │
│  │ MANAGEMENT (3 nets) │    │ DEVELOPMENT (2 nets)│    │ PRODUCTION (2 nets) │      │
│  ├─────────────────────┤    ├─────────────────────┤    ├─────────────────────┤      │
│  │                     │    │                     │    │                     │      │
│  │ ┌─────────────────┐ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │      │
│  │ │ Mgmt-Core       │ │    │ │ Dev-Apps        │ │    │ │ Prod-Apps       │ │      │
│  │ │ 10.10.0.0/24    │ │    │ │ 10.20.0.0/24    │ │    │ │ 10.30.0.0/24    │ │      │
│  │ │ VLAN 10         │ │    │ │ VLAN 20         │ │    │ │ VLAN 30         │ │      │
│  │ │ DNS, NTP, LDAP  │ │    │ │ Web, App, API   │ │    │ │ Web, App, API   │ │      │
│  │ │ Bastion, VPN    │ │    │ │ K8s Workers     │ │    │ │ K8s Workers     │ │      │
│  │ └─────────────────┘ │    │ └─────────────────┘ │    │ └─────────────────┘ │      │
│  │                     │    │                     │    │                     │      │
│  │ ┌─────────────────┐ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │      │
│  │ │ Mgmt-DevOps     │ │    │ │ Dev-Data        │ │    │ │ Prod-Data       │ │      │
│  │ │ 10.10.1.0/24    │ │    │ │ 10.20.1.0/24    │ │    │ │ 10.30.1.0/24    │ │      │
│  │ │ VLAN 11         │ │    │ │ VLAN 21         │ │    │ │ VLAN 31         │ │      │
│  │ │ ArgoCD, Jenkins │ │    │ │ PostgreSQL      │ │    │ │ PostgreSQL      │ │      │
│  │ │ GitLab, Harbor  │ │    │ │ Redis, Kafka    │ │    │ │ Redis, Kafka    │ │      │
│  │ └─────────────────┘ │    │ │ Elasticsearch   │ │    │ │ Elasticsearch   │ │      │
│  │                     │    │ └─────────────────┘ │    │ └─────────────────┘ │      │
│  │ ┌─────────────────┐ │    │                     │    │                     │      │
│  │ │ Mgmt-Observ     │ │    │                     │    │                     │      │
│  │ │ 10.10.2.0/24    │ │    │                     │    │                     │      │
│  │ │ VLAN 12         │ │    │                     │    │                     │      │
│  │ │ Grafana, Prom   │ │    │                     │    │                     │      │
│  │ │ Loki, Tempo     │ │    │                     │    │                     │      │
│  │ └─────────────────┘ │    │                     │    │                     │      │
│  └─────────────────────┘    └─────────────────────┘    └─────────────────────┘      │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## Physical Host Setup

```
┌─────────────────────────────────────────────────────────────┐
│                     oVirt Host (node)                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Linux Bridges                       │   │
│  │   VLAN networks on eno1 (802.1q tagged)             │   │
│  │                                                      │   │
│  │  VLAN 10 ──┬── pfSense VM (8 interfaces)            │   │
│  │  VLAN 11 ──┤                                         │   │
│  │  VLAN 12 ──┤   All inter-VLAN traffic goes          │   │
│  │  VLAN 20 ──┤   through pfSense for routing          │   │
│  │  VLAN 21 ──┤                                         │   │
│  │  VLAN 30 ──┤                                         │   │
│  │  VLAN 31 ──┘                                         │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                     ovirtmgmt                               │
│                          │                                  │
│                        eno1                                 │
│                          │                                  │
└──────────────────────────┼──────────────────────────────────┘
                           │
                       Internet
                    (192.168.0.x)
```

## Traffic Flow Diagrams

### 1. Inbound Internet Traffic (External → Internal)

```
Internet ───▶ pfSense (WAN) ───▶ NAT/Firewall Rules ───┬──▶ Prod-DMZ (Public Web)
                                                       ├──▶ Mgmt-DMZ (VPN/Admin)
                                                       └──▶ Dev-DMZ  (Test Access)
```

### 2. Outbound Internet Traffic (Internal → External)

```
ANY Internal VM ───▶ Default Gateway (pfSense) ───▶ NAT ───▶ Internet

Examples:
• Dev-App (10.20.3.x) needs npm packages ───▶ pfSense ───▶ npmjs.org
• Prod-DB (10.30.4.x) needs OS updates   ───▶ pfSense ───▶ Rocky mirrors
• Mgmt-DevOps (ArgoCD) pulls from GitHub ───▶ pfSense ───▶ github.com
```

### 3. Inter-VLAN / Inter-Cluster Traffic (Internal ↔ Internal)

All cross-subnet traffic routes through pfSense for firewall enforcement:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Dev-App (10.20.3.x) ───▶ pfSense ───▶ Dev-Database (10.20.4.x)        │
│                           [ALLOW: port 5432]                            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  Prod-Web (10.30.2.x) ───▶ pfSense ───▶ Prod-App (10.30.3.x)           │
│                            [ALLOW: port 8080]                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  Mgmt-Observability (10.10.2.x) ───▶ pfSense ───▶ Prod-App (10.30.3.x) │
│                                       [ALLOW: port 9090 metrics]        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  Dev-App (10.20.3.x) ───▶ pfSense ───▶ Prod-Database (10.30.4.x)       │
│                           [DENY: No dev→prod DB access]                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4. Same-Subnet Traffic (Intra-Cluster)

Traffic within the same subnet can be direct (Layer 2) or forced through pfSense:

**Option A: Direct (default)**
```
Prod-App-VM1 (10.30.3.10) ───▶ Prod-App-VM2 (10.30.3.11)
                               [Direct L2 switch]
```

**Option B: Forced via pfSense (micro-segmentation)**
```
Prod-App-VM1 ───▶ pfSense ───▶ Prod-App-VM2
                  [Private VLANs / Host firewall rules]
```

## VLAN & Subnet Assignments

### pfSense Interface Configuration

| pfSense Name | vtnet  | oVirt NIC | oVirt Network    | IP Address        | Purpose                    |
|--------------|--------|-----------|------------------|-------------------|----------------------------|
| WAN          | vtnet0 | nic1      | ovirtmgmt        | 192.168.0.101/24  | Internet uplink            |
| MGMT_CORE    | vtnet1 | nic2      | Mgmt-Core-Net    | 10.10.0.1/24      | Core infrastructure        |
| MGMT_DEVOPS  | vtnet2 | nic3      | Mgmt-DevOps-Net  | 10.10.1.1/24      | CI/CD, GitOps              |
| MGMT_OBSERV  | vtnet3 | nic4      | Mgmt-Observ-Net  | 10.10.2.1/24      | Monitoring, Logging        |
| DEV_APPS     | vtnet4 | nic5      | Dev-Apps-Net     | 10.20.0.1/24      | Dev applications           |
| DEV_DATA     | vtnet5 | nic6      | Dev-Data-Net     | 10.20.1.1/24      | Dev databases              |
| PROD_APPS    | vtnet6 | nic7      | Prod-Apps-Net    | 10.30.0.1/24      | Prod applications          |
| PROD_DATA    | vtnet7 | nic8      | Prod-Data-Net    | 10.30.1.1/24      | Prod databases             |

### Network to VLAN Mapping

| Network          | VLAN | Subnet         | DHCP Range          | Purpose                              |
|------------------|------|----------------|---------------------|--------------------------------------|
| Mgmt-Core-Net    | 10   | 10.10.0.0/24   | 10.10.0.100-200     | DNS, NTP, LDAP, Bastion, VPN         |
| Mgmt-DevOps-Net  | 11   | 10.10.1.0/24   | 10.10.1.100-200     | ArgoCD, Jenkins, GitLab, Harbor      |
| Mgmt-Observ-Net  | 12   | 10.10.2.0/24   | 10.10.2.100-200     | Grafana, Prometheus, Loki, Tempo     |
| Dev-Apps-Net     | 20   | 10.20.0.0/24   | 10.20.0.100-200     | Web, App, API servers, K8s workers   |
| Dev-Data-Net     | 21   | 10.20.1.0/24   | 10.20.1.100-200     | PostgreSQL, Redis, Kafka, ES         |
| Prod-Apps-Net    | 30   | 10.30.0.0/24   | 10.30.0.100-200     | Web, App, API servers, K8s workers   |
| Prod-Data-Net    | 31   | 10.30.1.0/24   | 10.30.1.100-200     | PostgreSQL, Redis, Kafka, ES         |

### Static IP Reservations (per network)

| Range         | Purpose                                    |
|---------------|--------------------------------------------|
| .1            | pfSense gateway                            |
| .2-.10        | Core infrastructure (DNS, NTP, etc.)       |
| .11-.50       | K8s control plane nodes (static)           |
| .51-.99       | Other static assignments                   |
| .100-.200     | DHCP pool                                  |
| .201-.254     | Reserved for future use                    |

## Firewall Rules (pfSense)

### Interface Groups

| Group      | Interfaces                           | Purpose                    |
|------------|--------------------------------------|----------------------------|
| MGMT_GROUP | MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV  | All Management networks    |
| DEV_GROUP  | DEV_APPS, DEV_DATA                   | All Development networks   |
| PROD_GROUP | PROD_APPS, PROD_DATA                 | All Production networks    |

### Firewall Rules

| Rule                                     | Action | Notes                        |
|------------------------------------------|--------|------------------------------|
| WAN → Prod-Apps:443                       | ALLOW  | Public HTTPS traffic         |
| WAN → Mgmt-Core:1194                      | ALLOW  | OpenVPN access               |
| WAN → *:*                                 | DENY   | Block all other inbound      |
| Prod-Apps → Prod-Data:5432,6379,9092      | ALLOW  | Apps to databases/cache/msg  |
| Dev-Apps → Dev-Data:5432,6379,9092        | ALLOW  | Apps to databases/cache/msg  |
| DEV_GROUP → PROD_GROUP                    | DENY   | No dev to prod access        |
| PROD_GROUP → DEV_GROUP                    | DENY   | No prod to dev access        |
| Mgmt-Observ → ALL:9090,9100,3100          | ALLOW  | Prometheus/Loki scraping     |
| ALL → Mgmt-Core:53,389,636                | ALLOW  | DNS and LDAP                 |
| ALL → WAN (Outbound)                      | ALLOW  | Internet access (NAT)        |

## Traffic Summary

| Traffic Type       | Path                                              |
|--------------------|---------------------------------------------------|
| Inbound Internet   | Internet → pfSense (WAN) → Prod-Apps              |
| Outbound Internet  | Any VM → pfSense (Gateway) → NAT → Internet       |
| Apps to Data       | Apps-Net → pfSense → Data-Net (same env)          |
| Cross Environment  | BLOCKED (Dev ↔ Prod)                              |
| Monitoring         | Mgmt-Observ → ALL (metrics ports only)            |

## Deployment Steps

1. **Create Networks** - `ansible-playbook -i inventory.ini network/create_networks.yml`
2. **Deploy pfSense VM** - `ansible-playbook -i inventory.ini network/configure_pfsense_vm.yml`
3. **Post-install pfSense** - `ansible-playbook -i inventory.ini network/configure_pfsense_vm.yml --tags post_install_pfsense`
4. **Configure pfSense**:
   - Assign interfaces (WAN, MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV, DEV_APPS, DEV_DATA, PROD_APPS, PROD_DATA)
   - Set static IPs on each interface
   - Create interface groups (MGMT_GROUP, DEV_GROUP, PROD_GROUP)
   - Enable DHCP on each interface
   - Configure firewall rules
5. **Deploy VMs** - `ansible-playbook -i inventory.ini compute/deploy_vms.yml`

## Changes from Previous Architecture

| Before (20 networks)         | After (7 networks)           |
|------------------------------|------------------------------|
| 7 Management networks        | 3 Management networks        |
| 5 Development networks       | 2 Development networks       |
| 8 Production networks        | 2 Production networks        |
| 21 pfSense interfaces        | 8 pfSense interfaces         |
| Complex firewall rules       | Simplified with groups       |
