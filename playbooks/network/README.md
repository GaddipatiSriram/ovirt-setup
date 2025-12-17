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

7 VLAN networks + WAN = 8 pfSense interfaces

### Management Networks (VLAN 10-12)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Mgmt-Core-Net | 10 | 10.10.0.0/24 | 10.10.0.1 | DNS, NTP, LDAP, Bastion |
| Mgmt-DevOps-Net | 11 | 10.10.1.0/24 | 10.10.1.1 | ArgoCD, Jenkins, GitLab |
| Mgmt-Observ-Net | 12 | 10.10.2.0/24 | 10.10.2.1 | Grafana, Prometheus, Loki |

### Development Networks (VLAN 20-21)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Dev-Apps-Net | 20 | 10.20.0.0/24 | 10.20.0.1 | Web, App, API, K8s |
| Dev-Data-Net | 21 | 10.20.1.0/24 | 10.20.1.1 | PostgreSQL, Redis, Kafka |

### Production Networks (VLAN 30-31)

| Network | VLAN | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| Prod-Apps-Net | 30 | 10.30.0.0/24 | 10.30.0.1 | Web, App, API, K8s |
| Prod-Data-Net | 31 | 10.30.1.0/24 | 10.30.1.1 | PostgreSQL, Redis, Kafka |

## pfSense Interface Mapping

| pfSense | vtnet | oVirt NIC | Network | IP |
|---------|-------|-----------|---------|-----|
| WAN | vtnet0 | nic1 | ovirtmgmt | 192.168.0.101 |
| MGMT_CORE | vtnet1 | nic2 | Mgmt-Core-Net | 10.10.0.1 |
| MGMT_DEVOPS | vtnet2 | nic3 | Mgmt-DevOps-Net | 10.10.1.1 |
| MGMT_OBSERV | vtnet3 | nic4 | Mgmt-Observ-Net | 10.10.2.1 |
| DEV_APPS | vtnet4 | nic5 | Dev-Apps-Net | 10.20.0.1 |
| DEV_DATA | vtnet5 | nic6 | Dev-Data-Net | 10.20.1.1 |
| PROD_APPS | vtnet6 | nic7 | Prod-Apps-Net | 10.30.0.1 |
| PROD_DATA | vtnet7 | nic8 | Prod-Data-Net | 10.30.1.1 |

## Playbook Details

### create_networks.yml

Creates 7 VLAN networks in oVirt:

1. Creates networks in datacenter with VLAN tags
2. Assigns networks to Default cluster
3. Attaches networks to host NIC (eno1)

### install_pfsense.yml

Creates pfSense firewall VM:

1. Downloads pfSense ISO (if not exists)
2. Creates VM with 4GB RAM, 2 CPU, 50GB disk
3. Adds 8 vNICs (WAN + 7 internal networks)
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
   - MGMT_GROUP: MGMT_CORE, MGMT_DEVOPS, MGMT_OBSERV
   - DEV_GROUP: DEV_APPS, DEV_DATA
   - PROD_GROUP: PROD_APPS, PROD_DATA
4. Enable DHCP on each interface
5. Configure firewall rules

See [docs/network-architecture.md](../../docs/network-architecture.md) for detailed firewall rules.
