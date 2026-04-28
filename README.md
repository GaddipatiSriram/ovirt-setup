# oVirt Infrastructure Deployment

Ansible playbooks for deploying oVirt Engine, networking (pfSense), and compute resources on Rocky Linux 9. This is the **lowest layer** of the homelab — VMs and networks; everything above runs on top.

## How this fits

Three sibling repos build on each other:

| Repo | Layer | Where |
|---|---|---|
| **`ovirt-setup/`** (this repo) | Hypervisor + VMs + pfSense networks + Kea DHCP | `/root/learning/ovirt-setup/` |
| **`cluster/`** | kubeadm bootstrap + authoritative CoreDNS, runs on the VMs created here | `/root/learning/cluster/` |
| **`okd/`** | OKD agent-based installer (separate from kubeadm), also targets VMs from here | `/root/learning/okd/` |
| **`cplanes/`** | GitOps content (ArgoCD AppSets + Helm charts) — what runs *inside* the clusters | `/root/learning/cplanes/` |

`ovirt-setup` provides the substrate. `cluster/` and `okd/` turn VMs into Kubernetes. `cplanes/` reconciles platform services onto those clusters.

## Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                  Physical Host (Rocky Linux 9)                         │
│                       node.engatwork.com                               │
├───────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │     oVirt Engine — ovirt.engatwork.com (Hosted Engine)     │       │
│  └────────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Management VLANs (10–14)                                              │
│  ┌──────────┐ ┌─────────────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐│
│  │mgmt-core │ │mgmt-devops-okd  │ │mgmt-obsv │ │mgmt-stg │ │mgmt-fge││
│  │  VLAN 10 │ │  + mgmt-workload│ │  VLAN 12 │ │  VLAN 13│ │ VLAN 14││
│  │10.10.1/24│ │     VLAN 11     │ │10.10.3/24│ │10.10.4/24│ │.5/24  ││
│  │CoreDNS   │ │OKD + ARC/runner │ │Prom/Graf │ │Rook-Ceph│ │Vault/ ││
│  │authoritv │ │mgmt-workload at │ │+ search  │ │server   │ │Keyclk ││
│  │+ ingress │ │  10.10.2.120    │ │co-locate │ │only     │ │+ data ││
│  └──────────┘ └─────────────────┘ └──────────┘ └─────────┘ └────────┘│
│                                                                        │
│  Dev / Prod VLANs (20–32) — empty (templates kept for rebuild)         │
│                                                                        │
│  ┌────────────────────────────────────────────────────────────┐       │
│  │     pfSense VM — Gateway, Firewall, Kea DHCP, DNS forwarder│       │
│  │     WAN: 192.168.0.101 ←→ Internet                         │       │
│  │     12 interfaces (1 WAN + 11 LAN VLANs)                   │       │
│  └────────────────────────────────────────────────────────────┘       │
│                                                                        │
│  Storage: Direct LUN (iSCSI/FC)                                        │
└───────────────────────────────────────────────────────────────────────┘
```

## Project structure

```
ovirt-setup/
├── ansible.cfg
├── requirements.yml
├── README.md
├── collections/                      Local Ansible collections (gitignored)
├── inventory/
│   ├── hosts.ini
│   └── group_vars/all.yml
├── playbooks/
│   ├── ovirt/                        Engine + hosts + storage
│   │   ├── site.yml
│   │   ├── prep_host.yml
│   │   ├── install_engine.yml
│   │   └── post_install.yml
│   ├── network/                      VLANs + pfSense
│   │   ├── create_networks.yml
│   │   ├── install_pfsense.yml
│   │   ├── post_install_pfsense.yml
│   │   └── pfsense/
│   │       ├── configure_pfsense.yml      Firewall + NAT + interface IPs
│   │       ├── configure_dns.yml          DNS resolver + forwarder
│   │       ├── pin_dhcp_leases.yml        Reserve IP per deterministic MAC
│   │       └── release_dhcp_leases.yml    Release leases by MAC / IP / hostname
│   └── compute/                      Cloud images + templates + VMs
│       ├── download_cloud_images.yml
│       ├── create_template.yml
│       ├── deploy_vms.yml
│       ├── full-deploy.yml                deploy_vms + disk-prep chained
│       ├── cleanup_vms.yml                Delete VMs + release Kea leases
│       └── vars/
│           ├── mgmt-core.yml
│           ├── mgmt-storage.yml
│           ├── mgmt-observ.yml
│           ├── mgmt-forge.yml
│           ├── mgmt-workload.yml          Single-VM workload cluster
│           ├── okd-devops.yml
│           └── dev-{web,apps,data}.yml    (templates; clusters not currently deployed)
└── docs/
    ├── network-architecture.md
    └── post-setup.md
```

## Cluster topology — current state

The set of clusters this repo provisions VMs for:

| Cluster | VLAN | IPs | VM size | Count | Status | What runs there |
|---|---|---|---|---|---|---|
| **mgmt-core** | 10 | 10.10.1.100-102 | medium | 3 | ✅ live | Authoritative CoreDNS for `*.engatwork.com` (MetalLB VIP `10.10.1.200`) |
| **mgmt-devops-okd** | 11 | 10.10.2.110-112 | okd-master (24/24 GiB) | 3 compact | ✅ live | OKD 3-node compact; hosts ArgoCD SRE + DevOps |
| **mgmt-workload** | 11 | 10.10.2.120 | okd (16/8 GiB) | 1 | 📋 planned | ARC controller + runners + ephemeral workloads (single-node, accepted SPOF) |
| **mgmt-observability** | 12 | 10.10.3.100-102 | okd | 3 | ✅ live | Prom/Graf/Alertmgr/Loki/Tempo + (future) search/SIEM (OpenSearch, Wazuh) |
| **mgmt-storage** | 13 | 10.10.4.100-102 | okd + extra disks | 3 | ✅ live | Rook-Ceph server (RBD + CephFS + RGW). No other workloads. |
| **mgmt-forge** | 14 | 10.10.5.103-105 | okd | 3 | ✅ live | Vault, Keycloak; planned: Harbor, SonarQube, GitLab, CNPG, Strimzi/Kafka, Redis, Apicurio |
| dev-{web,apps,data} | 20-22 | (deleted) | medium / large | (3 each) | 🗑️ rebuild template | Were idle; deleted to free memory. Vars files retained for rebuild when 2nd oVirt host arrives. |
| prod-{web,apps,data} | 30-32 | — | — | — | 📋 never deployed | Reserved VLANs only |

Memory budget at the host (~251 GiB physical): roughly 226 GiB committed after the dev-* cleanup; `mgmt-workload` adds 16 GiB when it lands → ~242 GiB → ~96% headroom on the single host. A 2nd oVirt host is the unlock for the deferred dev/prod tiers.

### Deterministic MAC convention

Cluster IDs map to a byte in the MAC, so co-located clusters in shared VLANs (mgmt-workload + OKD on VLAN 11; future search on VLAN 12) stay distinguishable in Kea's lease table:

```
56:6f:1b:<cluster_id>:<vlan_hex>:<idx>
   └────┬───┘ └────┬───┘ └─┬──┘  └┬─┘
   oVirt OUI  cluster id  vlan   vm idx
```

| Cluster | cluster_id byte | Sample MAC |
|---|---|---|
| mgmt-workload | `fd` | `56:6f:1b:fd:0b:01` |
| mgmt-observability | `ff` (planned) | `56:6f:1b:ff:0c:01` |
| mgmt-forge (rebuild) | `f0` | `56:6f:1b:f0:0e:01` |

Older clusters (mgmt-core, mgmt-storage, mgmt-observability today, OKD) use oVirt's auto-assigned MACs in the `56:6f:1b:6a:*` range — predates the deterministic convention.

## Network configuration

### VLANs

| Network | VLAN | CIDR | Gateway | Purpose | Cluster occupant |
|---|---|---|---|---|---|
| Mgmt-Core-Net | 10 | 10.10.1.0/24 | .1 | Foundational (DNS) | mgmt-core |
| Mgmt-DevOps-Net | 11 | 10.10.2.0/24 | .1 | OKD + workload plane | mgmt-devops-okd + mgmt-workload |
| Mgmt-Observ-Net | 12 | 10.10.3.0/24 | .1 | Observability + (future) search | mgmt-observability |
| Mgmt-Storage-Net | 13 | 10.10.4.0/24 | .1 | Storage (Ceph backend) | mgmt-storage |
| Mgmt-Forge-Net | 14 | 10.10.5.0/24 | .1 | Forge + (future) data services | mgmt-forge |
| Dev-Web-Net | 20 | 10.20.1.0/24 | .1 | Dev web tier | (empty — rebuild template) |
| Dev-Apps-Net | 21 | 10.20.2.0/24 | .1 | Dev app tier | (empty) |
| Dev-Data-Net | 22 | 10.20.3.0/24 | .1 | Dev data tier | (empty) |
| Prod-Web-Net | 30 | 10.30.1.0/24 | .1 | Prod web tier | (placeholder) |
| Prod-Apps-Net | 31 | 10.30.2.0/24 | .1 | Prod app tier | (placeholder) |
| Prod-Data-Net | 32 | 10.30.3.0/24 | .1 | Prod data tier | (placeholder) |

Per-VLAN MetalLB pool conventions live in `cplanes/networking/metallb/` and the per-cluster AppSet in `cplanes/argo-applications/sre/networking/metallb-appset.yaml` — `.200-.220` is reserved for LB VIPs in every workload-cluster VLAN.

### pfSense + Kea

The pfSense playbook configures:

- **Firewall + NAT**: floating pass-out rules + allow-all on management LANs
- **DNS forwarder**: `engatwork.com` → CoreDNS at `10.10.1.200` (mgmt-core MetalLB VIP)
- **DNS rebind exception**: `engatwork.com` may resolve to RFC1918 internal IPs
- **Kea DHCP**: per-VLAN subnet + pool. The `lease_cmds` hook is loaded so `pin_dhcp_leases.yml` and `release_dhcp_leases.yml` can manipulate leases via the control socket at `/tmp/kea4-ctrl-socket`.

`pin_dhcp_leases.yml` creates "soft reservations" (long-valid-lft leases pinning a MAC to a specific IP). Use when a cluster needs deterministic IPs across rebuilds — supply `vm_macs` and `vm_ips` in the cluster vars file, then run the playbook.

## Quick start

```bash
# 1. Install Ansible
dnf install -y ansible-core

# 2. Install required collections (gitignored, regenerated each clone)
cd /root/learning/ovirt-setup
ansible-galaxy collection install -r requirements.yml -p ./collections

# 3. Set credentials + LUN ID
vi inventory/group_vars/all.yml

# 4. Build the engine + host + storage
ansible-playbook playbooks/ovirt/site.yml

# 5. Create networks + bring up pfSense
ansible-playbook playbooks/network/create_networks.yml
ansible-playbook playbooks/network/install_pfsense.yml
ansible-playbook playbooks/network/post_install_pfsense.yml
ansible-playbook playbooks/network/pfsense/configure_pfsense.yml
ansible-playbook playbooks/network/pfsense/configure_dns.yml

# 6. Cloud images + template
ansible-playbook playbooks/compute/download_cloud_images.yml
ansible-playbook playbooks/compute/create_template.yml

# 7. Per-cluster: deploy VMs + grow disks (single playbook chains both)
ansible-playbook playbooks/compute/full-deploy.yml -e @playbooks/compute/vars/mgmt-core.yml
ansible-playbook playbooks/compute/full-deploy.yml -e @playbooks/compute/vars/mgmt-storage.yml
ansible-playbook playbooks/compute/full-deploy.yml -e @playbooks/compute/vars/mgmt-forge.yml
ansible-playbook playbooks/compute/full-deploy.yml -e @playbooks/compute/vars/mgmt-observ.yml
ansible-playbook playbooks/compute/full-deploy.yml -e @playbooks/compute/vars/mgmt-workload.yml

# 8. Optional — pin DHCP leases for clusters with deterministic MACs
ansible-playbook -i playbooks/network/pfsense/inventory.yml \
  playbooks/network/pfsense/pin_dhcp_leases.yml \
  -e @playbooks/compute/vars/mgmt-workload.yml

# 9. Hand off to cluster bootstrap
#    cd /root/learning/cluster/k8s-bstrp
#    ansible-playbook -i inventory/mgmt-workload.ini bootstrap-k8s.yml
```

## Cluster cleanup (destroy a cluster's VMs + release leases)

Single command — captures MACs from oVirt before delete, then releases the corresponding Kea leases on pfSense.

```bash
# Tear down a cluster (3 VMs + their pfSense leases)
ansible-playbook playbooks/compute/cleanup_vms.yml -e "vm_prefix=mgmt-workload vm_count=1"
ansible-playbook playbooks/compute/cleanup_vms.yml -e "vm_prefix=dev-web        vm_count=3"

# Release stale pinned leases (where VMs never created or were renamed)
ansible-playbook -i playbooks/network/pfsense/inventory.yml \
  playbooks/network/pfsense/release_dhcp_leases.yml \
  -e 'release_ips=["10.10.3.103","10.10.3.104","10.10.3.105"]'

# Skip pfSense cleanup if pfSense is unreachable
ansible-playbook playbooks/compute/cleanup_vms.yml \
  -e "vm_prefix=mgmt-workload vm_count=1 skip_dhcp_cleanup=true"
```

## Configuration variables

Edit `inventory/group_vars/all.yml`:

| Variable | Default | Description |
|---|---|---|
| `ovirt_engine_fqdn` | ovirt.engatwork.com | Engine FQDN |
| `ovirt_engine_admin_password` | unix | Admin portal password |
| `host_root_password` | unix | Host enrollment password |
| `ovirt_storage_type` | lun | Storage type (`lun` / `none`) |
| `ovirt_lun_id` | (set yours) | Direct LUN ID from `multipath -ll` |

## Ansible collections used

| Collection | Purpose |
|---|---|
| `ovirt.ovirt` | oVirt API operations |
| `pfsensible.core` | pfSense configuration |
| `community.general` | Modprobe, GitHub release fetches |
| `ansible.posix` | sysctl, SELinux |

## Access after deployment

| Service | URL | Credentials |
|---|---|---|
| oVirt Admin Portal | https://ovirt.engatwork.com/ovirt-engine/ | admin@ovirt / unix |
| oVirt VM Portal | https://ovirt.engatwork.com/ovirt-engine/web-ui | admin@ovirt / unix |
| pfSense WebUI | https://10.10.2.1/ | admin / pfsense |

## Troubleshooting

### oVirt Engine

```bash
hosted-engine --vm-status
systemctl status ovirt-engine
journalctl -u ovirt-engine -f
```

### pfSense / Kea

```bash
# SSH (use internal LAN, not WAN)
ssh admin@10.10.2.1     # default password: pfsense

# Inspect Kea state
echo -n '{"command":"lease4-get-all"}' | nc -U /tmp/kea4-ctrl-socket
echo -n '{"command":"config-get"}'      | nc -U /tmp/kea4-ctrl-socket
```

### Network connectivity

```bash
# From a VM on a managed VLAN
ping 10.10.2.1                        # gateway
dig @10.10.1.200 api.okd.engatwork.com  # internal DNS
```

### "VM won't start — host has only 3465 MB available"

oVirt's scheduler refuses to place a VM when the host's `memory_max` headroom is exhausted, even if `memory_guaranteed` math says there's room. Common when too many no-overcommit VMs (OKD masters at 24/24) consume the host. Either:

1. Stop idle VMs first (`cleanup_vms.yml` or shut down via UI).
2. Shrink existing VMs' `memory_max` (rolling for stateful clusters).
3. Add a 2nd oVirt host.

## Documentation

- [Network architecture](docs/network-architecture.md) — VLAN design, pfSense interfaces
- [Post-setup guide](docs/post-setup.md) — Manual configuration steps
- Cluster topology + plane assignments → [`/root/learning/cplanes/README.md`](../cplanes/README.md)

## Security notes

- Homelab passwords (`unix` / `pfsense`) — replace with Ansible Vault for real use
- pfSense WAN: NAT outbound; no inbound
- Inter-VLAN routing controlled by pfSense firewall rules (currently permissive on management LANs)
- oVirt uses self-signed certs by default

## Requirements

- Rocky Linux 9 / RHEL 9 / AlmaLinux 9
- oVirt 4.5+
- Ansible 2.14+
- Python 3.9+

## License

MIT
