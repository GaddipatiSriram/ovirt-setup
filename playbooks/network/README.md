# Network Playbooks

Playbooks for creating VLAN networks and deploying pfSense firewall VM.

## Playbooks

| Playbook | Description |
|----------|-------------|
| `create_networks.yml` | Create VLAN networks in oVirt |
| `install_pfsense.yml` | Create pfSense VM and boot from ISO |
| `post_install_pfsense.yml` | Eject ISO, set boot to HD, configure NICs |

## Usage

```bash
# 1. Create VLAN networks
ansible-playbook playbooks/network/create_networks.yml

# 2. Create pfSense VM (boots from ISO for installation)
ansible-playbook playbooks/network/install_pfsense.yml

# 3. Open noVNC console and install pfSense from ISO

# 4. After installation completes, run post-install
ansible-playbook playbooks/network/post_install_pfsense.yml
```

## Network Architecture

11 VLAN networks + WAN = 12 pfSense interfaces

### Management Networks (VLAN 10-14)

| Network | VLAN | Subnet | Gateway | DHCP Range | Purpose |
|---------|------|--------|---------|------------|---------|
| Mgmt-Core-Net | 10 | 10.10.1.0/24 | 10.10.1.1 | 10.10.1.100-200 | CoreDNS, Keycloak, Vault |
| Mgmt-DevOps-Net | 11 | 10.10.2.0/24 | 10.10.2.1 | 10.10.2.100-200 | OKD, ArgoCD, OCM |
| Mgmt-Observ-Net | 12 | 10.10.3.0/24 | 10.10.3.1 | 10.10.3.100-200 | Prometheus, Grafana, Loki |
| Mgmt-Storage-Net | 13 | 10.10.4.0/24 | 10.10.4.1 | 10.10.4.100-200 | Rook-Ceph storage |
| Mgmt-Forge-Net | 14 | 10.10.5.0/24 | 10.10.5.1 | 10.10.5.100-200 | GitLab, Harbor, SonarQube |

### Development Networks (VLAN 20-22)

| Network | VLAN | Subnet | Gateway | DHCP Range | Purpose |
|---------|------|--------|---------|------------|---------|
| Dev-Web-Net | 20 | 10.20.1.0/24 | 10.20.1.1 | 10.20.1.100-200 | Ingress, nginx |
| Dev-Apps-Net | 21 | 10.20.2.0/24 | 10.20.2.1 | 10.20.2.100-200 | APIs, microservices |
| Dev-Data-Net | 22 | 10.20.3.0/24 | 10.20.3.1 | 10.20.3.100-200 | PostgreSQL, Redis, Kafka |

### Production Networks (VLAN 30-32)

| Network | VLAN | Subnet | Gateway | DHCP Range | Purpose |
|---------|------|--------|---------|------------|---------|
| Prod-Web-Net | 30 | 10.30.1.0/24 | 10.30.1.1 | 10.30.1.100-200 | Ingress, nginx |
| Prod-Apps-Net | 31 | 10.30.2.0/24 | 10.30.2.1 | 10.30.2.100-200 | APIs, microservices |
| Prod-Data-Net | 32 | 10.30.3.0/24 | 10.30.3.1 | 10.30.3.100-200 | PostgreSQL, Redis, Kafka |

## pfSense Interface Mapping

| pfSense | vtnet | oVirt NIC | Network | IP |
|---------|-------|-----------|---------|-----|
| WAN | em0 | nic1 | ovirtmgmt | 192.168.0.101 (DHCP) |
| MGMT_CORE | vtnet0 | nic2 | Mgmt-Core-Net | 10.10.1.1 |
| MGMT_DEVOPS | vtnet1 | nic3 | Mgmt-DevOps-Net | 10.10.2.1 |
| MGMT_OBSERV | vtnet2 | nic4 | Mgmt-Observ-Net | 10.10.3.1 |
| MGMT_STORAGE | vtnet3 | nic5 | Mgmt-Storage-Net | 10.10.4.1 |
| MGMT_FORGE | vtnet4 | nic6 | Mgmt-Forge-Net | 10.10.5.1 |
| DEV_WEB | vtnet5 | nic7 | Dev-Web-Net | 10.20.1.1 |
| DEV_APPS | vtnet6 | nic8 | Dev-Apps-Net | 10.20.2.1 |
| DEV_DATA | vtnet7 | nic9 | Dev-Data-Net | 10.20.3.1 |
| PROD_WEB | vtnet8 | nic10 | Prod-Web-Net | 10.30.1.1 |
| PROD_APPS | vtnet9 | nic11 | Prod-Apps-Net | 10.30.2.1 |
| PROD_DATA | vtnet10 | nic12 | Prod-Data-Net | 10.30.3.1 |

## Playbook Details

### create_networks.yml

Creates 11 VLAN networks in oVirt:

1. Creates networks in datacenter with VLAN tags
2. Assigns networks to Default cluster
3. Attaches networks to host NIC (eno1)

### install_pfsense.yml

Creates pfSense firewall VM:

1. Downloads pfSense ISO (if not exists)
2. Creates VM with 4GB RAM, 2 CPU, 50GB disk
3. Adds 12 vNICs (WAN + 11 internal networks)
4. Attaches ISO and boots VM for installation

### post_install_pfsense.yml

Finalizes pfSense after ISO installation:

1. Force stops VM (handles stuck reboot)
2. Ejects ISO
3. Sets boot order to HD only
4. Starts VM
5. Ensures all NICs are linked
6. Hot-replugs WAN NIC for link detection

## Troubleshooting

### pfSense Stuck at Reboot

If pfSense gets stuck in a reboot loop after installation, the `post_install_pfsense.yml` playbook handles this automatically by force-stopping the VM.

If the playbook still times out, use oVirt Admin Portal:
1. Force stop the VM
2. Edit VM > Boot Options > Boot Sequence: Hard Disk only
3. Edit VM > Boot Options > Attach CD: empty
4. Start VM

## Disable pfSense Firewall

During initial setup, you may want to disable the firewall temporarily:

### Via WebGUI
1. Navigate to **System > Advanced > Firewall & NAT**
2. Check **Disable Firewall**
3. Click **Save**

### Via Console
```bash
# Temporarily disable (until reboot)
pfctl -d

# Re-enable
pfctl -e
```

### Via Console Menu
1. Select option **8** (Shell)
2. Run: `pfctl -d`

**Note:** Re-enable the firewall after configuration with `pfctl -e`

## Manual pfSense Configuration

After running post_install_pfsense.yml:

1. Open noVNC console in oVirt
2. Configure interfaces with IPs above
3. Create interface groups:
   - MGMT_GROUP: MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV, MGMT_STORAGE, MGMT_FORGE
   - DEV_GROUP: DEV_WEB, DEV_APPS, DEV_DATA
   - PROD_GROUP: PROD_WEB, PROD_APPS, PROD_DATA
4. Enable DHCP on each interface
5. Configure firewall rules

See [docs/network-architecture.md](../../docs/network-architecture.md) for detailed firewall rules.
