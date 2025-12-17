# Network Architecture

## Overview

This document describes the enterprise network architecture for the oVirt virtualization environment. The setup uses Open vSwitch (OVS) for internal VLAN handling without requiring a physical switch.

## Network Diagram

```
                                            INTERNET
                                                │
                                                │
                                  ══════════════╪══════════════
                                                │
                                                ▼
  ┌─────────────────────────────────────────────────────────────────────────────────────────┐
  │                              DEFAULT DATACENTER                                          │
  │                           (Host: node, Storage: data_lun)                               │
  ├─────────────────────────────────────────────────────────────────────────────────────────┤
  │                                                                                          │
  │  ┌─────────────────────────────────────────────────────────────────────────────────┐    │
  │  │                              Mgmt-Network                                        │    │
  │  │                    *** CENTRAL GATEWAY FOR ALL TRAFFIC ***                       │    │
  │  ├─────────────────────────────────────────────────────────────────────────────────┤    │
  │  │                                                                                  │    │
  │  │                      ┌─────────────────────────────────┐                        │    │
  │  │                      │           pfSense               │                        │    │
  │  │                      │      (Central Firewall)         │                        │    │
  │  │                      ├─────────────────────────────────┤                        │    │
  │  │                      │  WAN ──── Internet Gateway      │                        │    │
  │  │                      │  MGMT ─── 10.10.0.0/24          │                        │    │
  │  │                      │  DEV ──── 10.20.0.0/16          │                        │    │
  │  │                      │  PROD ─── 10.30.0.0/16          │                        │    │
  │  │                      └───────────────┬─────────────────┘                        │    │
  │  │                                      │                                          │    │
  │  │     ┌─────────────┐    ┌─────────────┴─────────────┐    ┌─────────────┐         │    │
  │  │     │  CoreDNS    │◄───┤  ALL ROUTING & FIREWALL   ├───▶│    NTP      │         │    │
  │  │     │ 10.10.0.10  │    │  RULES ENFORCED HERE      │    │ 10.10.0.11  │         │    │
  │  │     └─────────────┘    └───────────────────────────┘    └─────────────┘         │    │
  │  │                                                                                  │    │
  │  └──────────────────────────────────────┬──────────────────────────────────────────┘    │
  │                                         │                                               │
  │            ┌────────────────────────────┼────────────────────────────────┐              │
  │            │                            │                                │              │
  │            ▼                            ▼                                ▼              │
  │  ┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐            │
  │  │   MGMT VLAN      │       │    DEV VLAN      │       │   PROD VLAN      │            │
  │  │   10.10.0.0/24   │       │   10.20.0.0/16   │       │   10.30.0.0/16   │            │
  │  └────────┬─────────┘       └────────┬─────────┘       └────────┬─────────┘            │
  │           │                          │                          │                       │
  │           ▼                          ▼                          ▼                       │
  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────────────┐     │
  │  │ MANAGEMENT CLUSTERS │  │ DEVELOPMENT CLUSTERS│  │    PRODUCTION CLUSTERS      │     │
  │  │ (7 clusters)        │  │ (5 clusters)        │  │    (8 clusters)             │     │
  │  ├─────────────────────┤  ├─────────────────────┤  ├─────────────────────────────┤     │
  │  │                     │  │                     │  │                             │     │
  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │  │ ┌─────────────────┐         │     │
  │  │ │ Mgmt-DMZ        │ │  │ │ Dev-DMZ         │ │  │ │ Prod-DMZ        │         │     │
  │  │ │ 10.10.1.0/24    │ │  │ │ 10.20.1.0/24    │ │  │ │ 10.30.1.0/24    │         │     │
  │  │ │ Bastion, VPN GW │ │  │ │ LB, Rev Proxy   │ │  │ │ LB, WAF, Proxy  │         │     │
  │  │ └─────────────────┘ │  │ └────────┬────────┘ │  │ └────────┬────────┘         │     │
  │  │                     │  │          ▼          │  │          ▼                  │     │
  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │  │ ┌─────────────────┐         │     │
  │  │ │ Mgmt-Observ.    │ │  │ │ Dev-Web         │ │  │ │ Prod-Web        │         │     │
  │  │ │ 10.10.2.0/24    │ │  │ │ 10.20.2.0/24    │ │  │ │ 10.30.2.0/24    │         │     │
  │  │ │ Grafana,Prom,ELK│ │  │ │ Nginx, Apache   │ │  │ │ Nginx, Apache   │         │     │
  │  │ └─────────────────┘ │  │ └────────┬────────┘ │  │ └────────┬────────┘         │     │
  │  │                     │  │          ▼          │  │          ▼                  │     │
  │  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │  │ ┌─────────────────┐         │     │
  │  │ │ Mgmt-DevOps     │ │  │ │ Dev-App         │ │  │ │ Prod-App        │         │     │
  │  │ │ 10.10.3.0/24    │ │  │ │ 10.20.3.0/24    │ │  │ │ 10.30.3.0/24    │         │     │
  │  │ │ ArgoCD,Jenkins  │ │  │ │ Node, Java      │ │  │ │ Node, Java      │         │     │
  │  │ └─────────────────┘ │  │ └────────┬────────┘ │  │ └────────┬────────┘         │     │
  │  │                     │  │          │          │  │          │                  │     │
  │  │ ┌─────────────────┐ │  │    ┌─────┴─────┐    │  │    ┌─────┴─────┬─────┬────┐ │     │
  │  │ │ Mgmt-Identity   │ │  │    ▼           ▼    │  │    ▼           ▼     ▼    ▼ │     │
  │  │ │ 10.10.4.0/24    │ │  │ ┌───────┐ ┌───────┐ │  │ ┌───────┐ ┌─────┐ ┌─────┐  │     │
  │  │ │ LDAP, Keycloak  │ │  │ │Dev-DB │ │Dev-Msg│ │  │ │Prod-DB│ │Cache│ │ API │  │     │
  │  │ └─────────────────┘ │  │ │10.20. │ │10.20. │ │  │ │10.30. │ │10.30│ │10.30│  │     │
  │  │                     │  │ │4.0/24 │ │5.0/24 │ │  │ │4.0/24 │ │6.0  │ │7.0  │  │     │
  │  │ ┌─────────────────┐ │  │ └───────┘ └───────┘ │  │ └───────┘ └─────┘ └─────┘  │     │
  │  │ │ Mgmt-Security   │ │  │                     │  │                            │     │
  │  │ │ 10.10.5.0/24    │ │  │                     │  │ ┌───────┐ ┌───────┐        │     │
  │  │ │ IDS/IPS, SIEM   │ │  │                     │  │ │Prod-  │ │Prod-DR│        │     │
  │  │ └─────────────────┘ │  │                     │  │ │Msg    │ │10.30. │        │     │
  │  │                     │  │                     │  │ │10.30. │ │9.0/24 │        │     │
  │  │ ┌─────────────────┐ │  │                     │  │ │5.0/24 │ │       │        │     │
  │  │ │ Mgmt-Backup     │ │  │                     │  │ └───────┘ └───────┘        │     │
  │  │ │ 10.10.6.0/24    │ │  │                     │  │                            │     │
  │  │ │ Veeam, Restic   │ │  │                     │  │                            │     │
  │  │ └─────────────────┘ │  │                     │  │                            │     │
  │  └─────────────────────┘  └─────────────────────┘  └────────────────────────────┘     │
  │                                                                                       │
  └───────────────────────────────────────────────────────────────────────────────────────┘
```

## Physical Host Setup (No Physical Switch)

Since there is no physical switch, all VLAN handling is done internally via Open vSwitch (OVS):

```
┌─────────────────────────────────────────────────────────────┐
│                     oVirt Host (node)                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    br-int (OVS)                      │   │
│  │   All VLAN networks are virtual on this bridge      │   │
│  │                                                      │   │
│  │  VLAN 10 ──┬── pfSense VM (All VLANs connected)     │   │
│  │  VLAN 11 ──┤                                         │   │
│  │  VLAN 12 ──┤   All inter-VLAN traffic goes          │   │
│  │  ...      ─┤   through pfSense for routing          │   │
│  │  VLAN 39 ──┘                                         │   │
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

| Interface   | Network        | Purpose                          |
|-------------|----------------|----------------------------------|
| WAN         | DHCP/Static    | Internet uplink (192.168.0.x)    |
| MGMT        | 10.10.0.1/24   | Management network gateway       |
| MGMT_DMZ    | 10.10.1.1/24   | Mgmt-DMZ cluster                 |
| MGMT_OBS    | 10.10.2.1/24   | Mgmt-Observability cluster       |
| MGMT_DEVOPS | 10.10.3.1/24   | Mgmt-DevOps cluster              |
| MGMT_ID     | 10.10.4.1/24   | Mgmt-Identity cluster            |
| MGMT_SEC    | 10.10.5.1/24   | Mgmt-Security cluster            |
| MGMT_BKP    | 10.10.6.1/24   | Mgmt-Backup cluster              |
| DEV_DMZ     | 10.20.1.1/24   | Dev-DMZ cluster                  |
| DEV_WEB     | 10.20.2.1/24   | Dev-Web cluster                  |
| DEV_APP     | 10.20.3.1/24   | Dev-App cluster                  |
| DEV_DB      | 10.20.4.1/24   | Dev-Database cluster             |
| DEV_MSG     | 10.20.5.1/24   | Dev-Messaging cluster            |
| PROD_DMZ    | 10.30.1.1/24   | Prod-DMZ cluster                 |
| PROD_WEB    | 10.30.2.1/24   | Prod-Web cluster                 |
| PROD_APP    | 10.30.3.1/24   | Prod-App cluster                 |
| PROD_DB     | 10.30.4.1/24   | Prod-Database cluster            |
| PROD_MSG    | 10.30.5.1/24   | Prod-Messaging cluster           |
| PROD_CACHE  | 10.30.6.1/24   | Prod-Cache cluster               |
| PROD_API    | 10.30.7.1/24   | Prod-API cluster                 |
| PROD_DR     | 10.30.9.1/24   | Prod-DR cluster                  |

### Cluster to Network Mapping

| Cluster            | Network                | VLAN | Subnet         |
|--------------------|------------------------|------|----------------|
| Mgmt-Network       | Mgmt-Core-Net          | 10   | 10.10.0.0/24   |
| Mgmt-DMZ           | Mgmt-DMZ-Net           | 11   | 10.10.1.0/24   |
| Mgmt-Observability | Mgmt-Observability-Net | 12   | 10.10.2.0/24   |
| Mgmt-DevOps        | Mgmt-DevOps-Net        | 13   | 10.10.3.0/24   |
| Mgmt-Identity      | Mgmt-Identity-Net      | 14   | 10.10.4.0/24   |
| Mgmt-Security      | Mgmt-Security-Net      | 15   | 10.10.5.0/24   |
| Mgmt-Backup        | Mgmt-Backup-Net        | 16   | 10.10.6.0/24   |
| Dev-DMZ            | Dev-DMZ-Net            | 21   | 10.20.1.0/24   |
| Dev-Web            | Dev-Web-Net            | 22   | 10.20.2.0/24   |
| Dev-App            | Dev-App-Net            | 23   | 10.20.3.0/24   |
| Dev-Database       | Dev-Database-Net       | 24   | 10.20.4.0/24   |
| Dev-Messaging      | Dev-Messaging-Net      | 25   | 10.20.5.0/24   |
| Prod-DMZ           | Prod-DMZ-Net           | 31   | 10.30.1.0/24   |
| Prod-Web           | Prod-Web-Net           | 32   | 10.30.2.0/24   |
| Prod-App           | Prod-App-Net           | 33   | 10.30.3.0/24   |
| Prod-Database      | Prod-Database-Net      | 34   | 10.30.4.0/24   |
| Prod-Messaging     | Prod-Messaging-Net     | 35   | 10.30.5.0/24   |
| Prod-Cache         | Prod-Cache-Net         | 36   | 10.30.6.0/24   |
| Prod-API           | Prod-API-Net           | 37   | 10.30.7.0/24   |
| Prod-DR            | Prod-DR-Net            | 39   | 10.30.9.0/24   |

## Firewall Rules (pfSense)

### Example Rules

| Rule                                     | Action | Notes                        |
|------------------------------------------|--------|------------------------------|
| WAN → Prod-DMZ:443                        | ALLOW  | Public HTTPS traffic         |
| WAN → Prod-DMZ:80                         | ALLOW  | Redirect to HTTPS            |
| WAN → Mgmt-DMZ:1194                       | ALLOW  | OpenVPN access               |
| WAN → *:*                                 | DENY   | Block all other inbound      |
| Prod-DMZ → Prod-Web:80,443                | ALLOW  | LB to web servers            |
| Prod-Web → Prod-App:8080                  | ALLOW  | Web to app servers           |
| Prod-App → Prod-DB:5432,3306              | ALLOW  | App to databases             |
| Prod-App → Prod-Cache:6379                | ALLOW  | App to Redis                 |
| Prod-App → Prod-Msg:9092,5672             | ALLOW  | App to Kafka/RabbitMQ        |
| Dev-* → Prod-*                            | DENY   | No dev to prod access        |
| Prod-* → Dev-*                            | DENY   | No prod to dev access        |
| Mgmt-Observability → ALL:9090,9100        | ALLOW  | Prometheus scraping          |
| ALL → Mgmt-Identity:389,636               | ALLOW  | LDAP authentication          |
| ALL → Mgmt-Network:53                     | ALLOW  | DNS queries to CoreDNS       |
| ALL → WAN (Outbound)                      | ALLOW  | Internet access (NAT)        |

## Traffic Summary

| Traffic Type       | Path                                              |
|--------------------|---------------------------------------------------|
| Inbound Internet   | Internet → pfSense (WAN) → Target VLAN            |
| Outbound Internet  | Any VM → pfSense (Gateway) → NAT → Internet       |
| Inter-VLAN         | Source VLAN → pfSense → Destination VLAN          |
| Inter-Cluster      | Source Cluster → pfSense → Destination Cluster    |
| Intra-Cluster      | Direct (L2) or via pfSense (micro-segmentation)   |

## Deployment Steps

1. **Networks Created** - Run `ansible-playbook -i inventory.ini create_networks.yml`
2. **Setup Host Networks** - Attach VLAN networks to host NIC in oVirt UI
3. **Deploy pfSense VM** - Create VM in Mgmt-Network cluster with all VLAN interfaces
4. **Configure pfSense** - Set up interfaces, DHCP, firewall rules
5. **Deploy VMs** - Create VMs in appropriate clusters using dedicated VLAN networks
