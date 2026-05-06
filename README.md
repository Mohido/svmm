# svmm

Simple/small virtual machine manager built on top of libvirt, `virt-install`, and cloud-init.

svmm gives you five focused commands — `create`, `destroy`, `mount`, `unmount`, `ssh` — and delegates everything else (list, start, stop, console, snapshots, …) to `virsh`.

## How it works

Each VM gets a directory under `--base-dir` (default `/var/lib/libvirt/images/svmm/<name>/`) containing:

```
images/                     cached base cloud images (shared across VMs)
<name>/
    disk.qcow2              thin overlay on top of the cached base image
    user-data               cloud-init user-data (rendered on create)
    meta-data               cloud-init meta-data
    id_ed25519[.pub]        per-VM SSH key (injected via cloud-init)
```

Runtime state (RAM, vCPUs, mounts) lives in the libvirt domain XML — svmm reads and edits it through `virsh`.

## System prerequisites

Install the required system packages:

```bash
# Debian/Ubuntu
sudo apt install virtinst qemu-kvm libvirt-daemon-system virtiofsd python3-yaml

# Fedora/RHEL
sudo dnf install virt-install qemu-kvm libvirt virtiofsd python3-pyyaml
```

Prepare the base directory so libvirt's QEMU user can read disk images:

```bash
sudo mkdir -p /var/lib/libvirt/images/svmm
sudo chgrp libvirt-qemu /var/lib/libvirt/images/svmm
sudo chmod 2775 /var/lib/libvirt/images/svmm
sudo usermod -aG libvirt-qemu $USER
sudo chmod o+r /var/lib/libvirt/images
# log out and back in for the group change to take effect
```

## Installation

```bash
pip install svmm
```

Or install directly from source:

```bash
git clone https://github.com/youruser/svmm
pip install ./svmm
```

## Usage

### create

```bash
svmm create -n myvm \
    --image https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img \
    --ram 4096 --vcpus 2 --disk 20
```

Optional flags:
- `-m src:dst` — virtiofs share, repeatable
- `--cloud-init path/to/user-data.yaml` — custom cloud-init template
- `--no-default-key` — skip injecting svmm's generated SSH key

### destroy

```bash
svmm destroy -n myvm          # prompts for confirmation
svmm destroy -n myvm --yes    # skip confirmation
```

### ssh

```bash
svmm ssh -n myvm
svmm ssh -n myvm -- df -h     # run a command
```

### mount / unmount

```bash
# add a virtiofs share (persists to domain XML; live-mounts if VM is running)
svmm mount -n myvm -m /host/path:/guest/path

# hot-attach to a running VM without rebooting
svmm mount -n myvm -m /host/path:/guest/path --hot

# remove a share
svmm unmount -n myvm -m /guest/path
```

### Other operations via virsh

```bash
virsh list --all
virsh start myvm
virsh shutdown myvm
virsh console myvm
virsh snapshot-create-as myvm snap1
```

## Global flag

`--base-dir PATH` overrides the default state directory for all commands.
