# Compute Playbooks

Playbooks for managing cloud images, VM templates, and VM deployment.

## Playbooks

| Playbook | Description |
|----------|-------------|
| `download_cloud_images.yml` | Download and upload cloud images to oVirt |
| `create_template.yml` | Create VM template from cloud image |
| `deploy_vms.yml` | Deploy VMs from template |

## Usage

```bash
# Download cloud images
ansible-playbook playbooks/compute/download_cloud_images.yml
ansible-playbook playbooks/compute/download_cloud_images.yml -e "images_to_download=['rocky9','ubuntu2404']"
ansible-playbook playbooks/compute/download_cloud_images.yml -e "images_to_download=['all']"

# Create template
ansible-playbook playbooks/compute/create_template.yml
ansible-playbook playbooks/compute/create_template.yml -e "image_source=rocky9 template_name=rocky-9-prod"

# Deploy VMs
ansible-playbook playbooks/compute/deploy_vms.yml
ansible-playbook playbooks/compute/deploy_vms.yml -e "vm_count=5 vm_size=large"
```

## Available Cloud Images

| Key | Image | OS Type |
|-----|-------|---------|
| `fedora41` | Fedora 41 Cloud Base | fedora_64 |
| `fedora40` | Fedora 40 Cloud Base | fedora_64 |
| `rocky9` | Rocky Linux 9 GenericCloud | rhel_9x64 |
| `rocky8` | Rocky Linux 8 GenericCloud | rhel_8x64 |
| `alma9` | AlmaLinux 9 GenericCloud | rhel_9x64 |
| `ubuntu2404` | Ubuntu 24.04 LTS Noble | ubuntu_64 |
| `ubuntu2204` | Ubuntu 22.04 LTS Jammy | ubuntu_64 |
| `debian12` | Debian 12 Bookworm | debian_64 |
| `debian11` | Debian 11 Bullseye | debian_64 |

## Playbook Details

### download_cloud_images.yml

Downloads cloud images and uploads to oVirt storage:

- Downloads qcow2/img files from official sources
- Uploads as bootable disks to storage domain
- Skips images that already exist

**Variables:**
```yaml
images_to_download:
  - fedora41      # Default
# Or download all: ['all']
```

### create_template.yml

Creates VM template from cloud image:

1. Downloads image if not in oVirt
2. Creates builder VM with image disk
3. Runs cloud-init for initial setup
4. Installs qemu-guest-agent
5. Creates template from VM
6. Cleans up builder VM

**Variables:**
```yaml
image_source: "fedora41"
template_name: "fedora-41-mgmt"
template_description: "Fedora 41 template for Mgmt-Core-Net"
network_name: "Mgmt-Core-Net"
vm_memory: 2GiB
vm_cpu_cores: 2
vm_disk_size: 20GiB
```

### deploy_vms.yml

Deploys multiple VMs from template:

1. Clones VMs from template
2. Configures cloud-init (hostname, users, SSH)
3. Starts VMs
4. Displays IP addresses

**Variables:**
```yaml
template_name: "fedora-41-mgmt"
vm_prefix: "fedora-mgmt"
vm_count: 3
vm_size: "medium"  # small, medium, large

# Size definitions
vm_sizes:
  small:   { memory: 1GiB, cpu_cores: 1, disk: 20GiB }
  medium:  { memory: 2GiB, cpu_cores: 2, disk: 40GiB }
  large:   { memory: 4GiB, cpu_cores: 4, disk: 80GiB }
```

## Default Credentials

VMs created with these playbooks have:

| User | Password | Notes |
|------|----------|-------|
| root | unix | SSH enabled |
| admin | unix | sudo NOPASSWD |

## Examples

```bash
# Create Rocky 9 template for production
ansible-playbook playbooks/compute/create_template.yml \
  -e "image_source=rocky9" \
  -e "template_name=rocky-9-prod" \
  -e "network_name=Prod-Apps-Net"

# Deploy 5 large VMs
ansible-playbook playbooks/compute/deploy_vms.yml \
  -e "template_name=rocky-9-prod" \
  -e "vm_prefix=prod-app" \
  -e "vm_count=5" \
  -e "vm_size=large"
```
