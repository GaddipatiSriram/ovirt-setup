# Network Architecture (3-Tier)

## Overview

This document describes the 3-tier network architecture for the oVirt virtualization environment.
- **5 Management Networks** - Core, DevOps, Observability, Storage, Forge
- **3 Development Networks** - Web, Apps, and Data
- **3 Production Networks** - Web, Apps, and Data
- **Total: 11 networks + WAN = 12 pfSense interfaces**

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
│                         │  MGMT_CORE ───── 10.10.1.1      │                          │
│                         │  MGMT_DEVOPS ─── 10.10.2.1      │                          │
│                         │  MGMT_OBSERV ─── 10.10.3.1      │                          │
│                         │  MGMT_STORAGE ── 10.10.4.1      │                          │
│                         │  MGMT_FORGE ──── 10.10.5.1      │                          │
│                         │  DEV_WEB ─────── 10.20.1.1      │                          │
│                         │  DEV_APPS ────── 10.20.2.1      │                          │
│                         │  DEV_DATA ────── 10.20.3.1      │                          │
│                         │  PROD_WEB ────── 10.30.1.1      │                          │
│                         │  PROD_APPS ───── 10.30.2.1      │                          │
│                         │  PROD_DATA ───── 10.30.3.1      │                          │
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
│  │ MANAGEMENT (5 nets) │    │ DEVELOPMENT (3 nets)│    │ PRODUCTION (3 nets) │      │
│  ├─────────────────────┤    ├─────────────────────┤    ├─────────────────────┤      │
│  │                     │    │                     │    │                     │      │
│  │ ┌─────────────────┐ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │      │
│  │ │ Mgmt-Core       │ │    │ │ Dev-Web         │ │    │ │ Prod-Web        │ │      │
│  │ │ 10.10.1.0/24    │ │    │ │ 10.20.1.0/24    │ │    │ │ 10.30.1.0/24    │ │      │
│  │ │ VLAN 10         │ │    │ │ VLAN 20         │ │    │ │ VLAN 30         │ │      │
│  │ │ DNS, Keycloak   │ │    │ │ Ingress, nginx  │ │    │ │ Ingress, nginx  │ │      │
│  │ │ Vault           │ │    │ │ HAProxy, CDN    │ │    │ │ HAProxy, CDN    │ │      │
│  │ └─────────────────┘ │    │ └─────────────────┘ │    │ └─────────────────┘ │      │
│  │                     │    │         │           │    │         │           │      │
│  │ ┌─────────────────┐ │    │         ▼           │    │         ▼           │      │
│  │ │ Mgmt-DevOps     │ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │      │
│  │ │ 10.10.2.0/24    │ │    │ │ Dev-Apps        │ │    │ │ Prod-Apps       │ │      │
│  │ │ VLAN 11 (OKD)   │ │    │ │ 10.20.2.0/24    │ │    │ │ 10.30.2.0/24    │ │      │
│  │ │ ArgoCD, OCM     │ │    │ │ VLAN 21         │ │    │ │ VLAN 31         │ │      │
│  │ │ Backstage       │ │    │ │ API, Services   │ │    │ │ API, Services   │ │      │
│  │ └─────────────────┘ │    │ │ K8s Workers     │ │    │ │ K8s Workers     │ │      │
│  │                     │    │ └─────────────────┘ │    │ └─────────────────┘ │      │
│  │ ┌─────────────────┐ │    │         │           │    │         │           │      │
│  │ │ Mgmt-Observ     │ │    │         ▼           │    │         ▼           │      │
│  │ │ 10.10.3.0/24    │ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │      │
│  │ │ VLAN 12         │ │    │ │ Dev-Data        │ │    │ │ Prod-Data       │ │      │
│  │ │ Grafana, Prom   │ │    │ │ 10.20.3.0/24    │ │    │ │ 10.30.3.0/24    │ │      │
│  │ │ Loki, Tempo     │ │    │ │ VLAN 22         │ │    │ │ VLAN 32         │ │      │
│  │ └─────────────────┘ │    │ │ PostgreSQL      │ │    │ │ PostgreSQL      │ │      │
│  │                     │    │ │ Redis, Kafka    │ │    │ │ Redis, Kafka    │ │      │
│  │ ┌─────────────────┐ │    │ │ Elasticsearch   │ │    │ │ Elasticsearch   │ │      │
│  │ │ Mgmt-Storage    │ │    │ └─────────────────┘ │    │ └─────────────────┘ │      │
│  │ │ 10.10.4.0/24    │ │    │                     │    │                     │      │
│  │ │ VLAN 13         │ │    │                     │    │                     │      │
│  │ │ Rook-Ceph       │ │    │                     │    │                     │      │
│  │ │ NFS, S3, RBD    │ │    │                     │    │                     │      │
│  │ └─────────────────┘ │    │                     │    │                     │      │
│  │                     │    │                     │    │                     │      │
│  │ ┌─────────────────┐ │    │                     │    │                     │      │
│  │ │ Mgmt-Forge      │ │    │                     │    │                     │      │
│  │ │ 10.10.5.0/24    │ │    │                     │    │                     │      │
│  │ │ VLAN 14         │ │    │                     │    │                     │      │
│  │ │ GitLab, Harbor  │ │    │                     │    │                     │      │
│  │ │ SonarQube       │ │    │                     │    │                     │      │
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
│  │  VLAN 10 ──┬── pfSense VM (12 interfaces)           │   │
│  │  VLAN 11 ──┤                                         │   │
│  │  VLAN 12 ──┤   All inter-VLAN traffic goes          │   │
│  │  VLAN 13 ──┤   through pfSense for routing          │   │
│  │  VLAN 14 ──┤                                         │   │
│  │  VLAN 20 ──┤                                         │   │
│  │  VLAN 21 ──┤                                         │   │
│  │  VLAN 22 ──┤                                         │   │
│  │  VLAN 30 ──┤                                         │   │
│  │  VLAN 31 ──┤                                         │   │
│  │  VLAN 32 ──┘                                         │   │
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
Internet ───▶ pfSense (WAN) ───▶ NAT/Firewall Rules ───┬──▶ Prod-Web (Public HTTPS)
                                                       ├──▶ Mgmt-Core (VPN/Admin)
                                                       └──▶ Dev-Web  (Test Access)
```

### 2. Outbound Internet Traffic (Internal → External)

```
ANY Internal VM ───▶ Default Gateway (pfSense) ───▶ NAT ───▶ Internet

Examples:
• Dev-App (10.20.3.x) needs npm packages ───▶ pfSense ───▶ npmjs.org
• Prod-DB (10.30.4.x) needs OS updates   ───▶ pfSense ───▶ Rocky mirrors
• Mgmt-DevOps (ArgoCD) pulls from GitHub ───▶ pfSense ───▶ github.com
```

### 3. Inter-VLAN / 3-Tier Traffic (Internal ↔ Internal)

All cross-subnet traffic routes through pfSense for firewall enforcement:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  3-TIER FLOW (Web → Apps → Data)                                        │
│                                                                         │
│  Prod-Web (10.30.1.x) ───▶ pfSense ───▶ Prod-Apps (10.30.2.x)          │
│                            [ALLOW: port 8080]                           │
│                                                                         │
│  Prod-Apps (10.30.2.x) ───▶ pfSense ───▶ Prod-Data (10.30.3.x)         │
│                             [ALLOW: port 5432, 6379, 9092]              │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  Dev-Web (10.20.1.x) ───▶ pfSense ───▶ Dev-Apps (10.20.2.x)            │
│                           [ALLOW: port 8080]                            │
│                                                                         │
│  Dev-Apps (10.20.2.x) ───▶ pfSense ───▶ Dev-Data (10.20.3.x)           │
│                            [ALLOW: port 5432, 6379, 9092]               │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  MONITORING (Observability → All)                                       │
│                                                                         │
│  Mgmt-Observ (10.10.3.x) ───▶ pfSense ───▶ ALL tiers                   │
│                                [ALLOW: port 9090, 9100, 3100 metrics]   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  BLOCKED TRAFFIC                                                        │
│                                                                         │
│  Dev-* (10.20.x.x) ───▶ pfSense ───▶ Prod-* (10.30.x.x)                │
│                         [DENY: No dev→prod access]                      │
│                                                                         │
│  *-Web ───▶ pfSense ───▶ *-Data                                        │
│             [DENY: Web tier cannot access Data tier directly]           │
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

| pfSense Name | vtnet    | oVirt NIC | oVirt Network      | IP Address        | Purpose                    |
|--------------|----------|-----------|---------------------|-------------------|----------------------------|
| WAN          | em0      | nic1      | ovirtmgmt           | 192.168.0.101/24  | Internet uplink (DHCP)     |
| MGMT_CORE    | vtnet0   | nic2      | Mgmt-Core-Net       | 10.10.1.1/24      | Core (DNS, Keycloak, Vault)|
| MGMT_DEVOPS  | vtnet1   | nic3      | Mgmt-DevOps-Net     | 10.10.2.1/24      | OKD, ArgoCD, OCM           |
| MGMT_OBSERV  | vtnet2   | nic4      | Mgmt-Observ-Net     | 10.10.3.1/24      | Monitoring, Logging        |
| MGMT_STORAGE | vtnet3   | nic5      | Mgmt-Storage-Net    | 10.10.4.1/24      | Rook-Ceph storage          |
| MGMT_FORGE   | vtnet4   | nic6      | Mgmt-Forge-Net      | 10.10.5.1/24      | GitLab, Harbor, SonarQube  |
| DEV_WEB      | vtnet5   | nic7      | Dev-Web-Net         | 10.20.1.1/24      | Dev web/ingress tier       |
| DEV_APPS     | vtnet6   | nic8      | Dev-Apps-Net        | 10.20.2.1/24      | Dev applications           |
| DEV_DATA     | vtnet7   | nic9      | Dev-Data-Net        | 10.20.3.1/24      | Dev databases              |
| PROD_WEB     | vtnet8   | nic10     | Prod-Web-Net        | 10.30.1.1/24      | Prod web/ingress tier      |
| PROD_APPS    | vtnet9   | nic11     | Prod-Apps-Net       | 10.30.2.1/24      | Prod applications          |
| PROD_DATA    | vtnet10  | nic12     | Prod-Data-Net       | 10.30.3.1/24      | Prod databases             |

### Network to VLAN Mapping

| Network          | VLAN | Subnet         | DHCP Range            | Purpose                              |
|------------------|------|----------------|----------------------|--------------------------------------|
| Mgmt-Core-Net    | 10   | 10.10.1.0/24   | 10.10.1.100-200      | CoreDNS, Keycloak, Vault             |
| Mgmt-DevOps-Net  | 11   | 10.10.2.0/24   | 10.10.2.100-200      | OKD, ArgoCD, OCM, Backstage          |
| Mgmt-Observ-Net  | 12   | 10.10.3.0/24   | 10.10.3.100-200      | Prometheus, Grafana, Loki, Tempo     |
| Mgmt-Storage-Net | 13   | 10.10.4.0/24   | 10.10.4.100-200      | Rook-Ceph, NFS, S3, RBD              |
| Mgmt-Forge-Net   | 14   | 10.10.5.0/24   | 10.10.5.100-200      | GitLab, Harbor, SonarQube            |
| Dev-Web-Net      | 20   | 10.20.1.0/24   | 10.20.1.100-200      | Ingress, nginx, HAProxy, CDN cache   |
| Dev-Apps-Net     | 21   | 10.20.2.0/24   | 10.20.2.100-200      | API servers, microservices, K8s      |
| Dev-Data-Net     | 22   | 10.20.3.0/24   | 10.20.3.100-200      | PostgreSQL, Redis, Kafka, ES         |
| Prod-Web-Net     | 30   | 10.30.1.0/24   | 10.30.1.100-200      | Ingress, nginx, HAProxy, CDN cache   |
| Prod-Apps-Net    | 31   | 10.30.2.0/24   | 10.30.2.100-200      | API servers, microservices, K8s      |
| Prod-Data-Net    | 32   | 10.30.3.0/24   | 10.30.3.100-200      | PostgreSQL, Redis, Kafka, ES         |

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

| Group      | Interfaces                                            | Purpose                    |
|------------|-------------------------------------------------------|----------------------------|
| MGMT_GROUP | MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV, MGMT_STORAGE, MGMT_FORGE | All Management networks    |
| DEV_GROUP  | DEV_WEB, DEV_APPS, DEV_DATA                           | All Development networks   |
| PROD_GROUP | PROD_WEB, PROD_APPS, PROD_DATA                        | All Production networks    |
| WEB_GROUP  | DEV_WEB, PROD_WEB                                     | All Web tier networks      |
| APPS_GROUP | DEV_APPS, PROD_APPS                                   | All Apps tier networks     |
| DATA_GROUP | DEV_DATA, PROD_DATA                                   | All Data tier networks     |

### Firewall Rules

| Rule                                     | Action | Notes                        |
|------------------------------------------|--------|------------------------------|
| **Inbound (WAN)**                        |        |                              |
| WAN → Prod-Web:443,80                     | ALLOW  | Public HTTPS/HTTP traffic    |
| WAN → Mgmt-Core:1194                      | ALLOW  | OpenVPN access               |
| WAN → *:*                                 | DENY   | Block all other inbound      |
| **3-Tier Flow (Web → Apps → Data)**      |        |                              |
| Prod-Web → Prod-Apps:8080,8443            | ALLOW  | Web to application tier      |
| Prod-Apps → Prod-Data:5432,6379,9092      | ALLOW  | Apps to databases/cache/msg  |
| Dev-Web → Dev-Apps:8080,8443              | ALLOW  | Web to application tier      |
| Dev-Apps → Dev-Data:5432,6379,9092        | ALLOW  | Apps to databases/cache/msg  |
| **Tier Isolation**                       |        |                              |
| WEB_GROUP → DATA_GROUP                    | DENY   | Web cannot access data direct|
| DATA_GROUP → WEB_GROUP                    | DENY   | Data cannot access web       |
| DATA_GROUP → APPS_GROUP                   | DENY   | Data cannot initiate to apps |
| **Environment Isolation**                |        |                              |
| DEV_GROUP → PROD_GROUP                    | DENY   | No dev to prod access        |
| PROD_GROUP → DEV_GROUP                    | DENY   | No prod to dev access        |
| **Management Access**                    |        |                              |
| Mgmt-Observ → ALL:9090,9100,3100          | ALLOW  | Prometheus/Loki scraping     |
| ALL → Mgmt-Core:53,123,389,636            | ALLOW  | DNS, NTP, and LDAP           |
| ALL → WAN (Outbound)                      | ALLOW  | Internet access (NAT)        |

## Traffic Summary

| Traffic Type       | Path                                              |
|--------------------|---------------------------------------------------|
| Inbound Internet   | Internet → pfSense (WAN) → Prod-Web → Prod-Apps   |
| Outbound Internet  | Any VM → pfSense (Gateway) → NAT → Internet       |
| Web to Apps        | Web-Net → pfSense → Apps-Net (same env)           |
| Apps to Data       | Apps-Net → pfSense → Data-Net (same env)          |
| Web to Data        | BLOCKED (must go through Apps tier)               |
| Cross Environment  | BLOCKED (Dev ↔ Prod)                              |
| Monitoring         | Mgmt-Observ → ALL (metrics ports only)            |

## Deployment Steps

1. **Create Networks** - `ansible-playbook -i inventory.ini network/create_networks.yml`
2. **Deploy pfSense VM** - `ansible-playbook -i inventory.ini network/configure_pfsense_vm.yml`
3. **Post-install pfSense** - `ansible-playbook -i inventory.ini network/configure_pfsense_vm.yml --tags post_install_pfsense`
4. **Configure pfSense**:
   - Assign interfaces (WAN, MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV, DEV_WEB, DEV_APPS, DEV_DATA, PROD_WEB, PROD_APPS, PROD_DATA)
   - Set static IPs on each interface
   - Create interface groups (MGMT_GROUP, DEV_GROUP, PROD_GROUP, WEB_GROUP, APPS_GROUP, DATA_GROUP)
   - Enable DHCP on each interface
   - Configure 3-tier firewall rules
5. **Deploy VMs** - `ansible-playbook -i inventory.ini compute/deploy_vms.yml`

## Changes from Previous Architecture

| Before (9 networks)          | After (11 networks)           |
|------------------------------|-------------------------------|
| 3 Management networks        | 5 Management networks         |
| 3 Development networks       | 3 Development networks        |
| 3 Production networks        | 3 Production networks         |
| 10 pfSense interfaces        | 12 pfSense interfaces         |
| Storage on mgmt-core         | Dedicated mgmt-storage cluster|
| GitLab on mgmt-devops        | Dedicated mgmt-forge cluster  |

### New Networks Added

| Network          | Purpose                                      |
|------------------|----------------------------------------------|
| Mgmt-Storage-Net | Rook-Ceph storage services (NFS, S3, RBD)   |
| Mgmt-Forge-Net   | Software forge (GitLab, Harbor, SonarQube)  |
