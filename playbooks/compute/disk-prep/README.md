# compute/disk-prep

Generic oVirt + in-VM disk preparation. Grows the root disk to a target
size and attaches any number of extra data disks per cluster recipe.

Refactored from the old `storage/mgmt-storage-prep/` (which was
mgmt-storage-specific, with hardcoded per-VM UUIDs in `inventory.ini`).

## What it does

Four plays, all idempotent:

1. **Discover** — auth to oVirt, list all VMs matching
   `{{ vm_prefix }}-*`, fetch their IPs and disk attachments, then
   register them in a runtime-only `cluster_vms` group via `add_host`.
   No per-VM UUIDs live in the inventory file.
2. **Grow root (oVirt side)** — if `root_disk_grow_to_gb > 0`, resize
   each VM's boot disk to that value. Skipped otherwise.
3. **Grow root (in-VM side)** — `growpart /dev/sda 4` + `btrfs
   filesystem resize max /`. Skipped when not growing.
4. **Attach extras** — for each entry in `extra_disks`, create and
   attach a new virtio_scsi disk (named `<name_suffix>-<NN>` where
   `NN` is the trailing index of the VM name). Skipped when
   `extra_disks` is empty.

## Per-cluster knobs

Declare these in `compute/vars/<cluster>.yml` alongside the
`deploy_vms.yml` variables for the same cluster:

```yaml
# Target in-VM root size. Set this even when the deploy_vms preset
# already allocates the disk at that size — template images ship with
# /dev/sda4 at ~4 GiB, so the in-VM growpart + btrfs resize must still
# run to reclaim the rest. oVirt-side resize is skipped when the disk
# is already ≥ target. Omit or set to 0 only when you don't want to
# touch the root FS at all.
root_disk_grow_to_gb: 100

# Extra disks per VM. Empty list (or omit) to skip.
extra_disks:
  - { name_suffix: ceph-osd, size_gb: 500 }
```

## Usage

### Standalone

```bash
cd ovirt-setup/playbooks/compute/disk-prep
ansible-playbook disk-prep.yml -e @../vars/mgmt-observ.yml
```

### Via wrapper

```bash
cd ovirt-setup/playbooks/compute
ansible-playbook full-deploy.yml -e @vars/mgmt-observ.yml
```

`full-deploy.yml` imports `deploy_vms.yml` then `disk-prep/disk-prep.yml`
so a single command provisions the VMs, grows the root disk to the
cluster's target, and attaches any extra disks the recipe calls for.

## Why discovery-at-runtime

The old playbook required you to edit an `inventory.ini` with the
`ovirt_vm_id` and `root_disk_id` of each VM — UUIDs that only exist
after oVirt has created the VMs. That makes it impossible to script
a one-shot "create cluster from scratch" flow.

By querying oVirt at run time (`GET /vms?search=name={{ vm_prefix }}-*`),
the playbook works against any freshly-deployed set of VMs, matches the
pattern used by `deploy_vms.yml`, and composes cleanly with
`full-deploy.yml`. This mirrors what enterprise cluster-factory tools
(Terraform + cloud-init, Crossplane + Composition) do: the cluster's
identity IS its name prefix, not a hand-curated UUID list.

See [`cluster/eks-bootstrap-argo.md`](../../../../learning/cluster/eks-bootstrap-argo.md)
§1 "AWS-recommended tooling" for the broader enterprise-lifecycle
context that motivated this refactor.
