# KubeVirt + Windows VM

Runs a Windows Server VM inside Kubernetes using [KubeVirt](https://kubevirt.io/).

## Components

| Resource | Purpose |
|----------|---------|
| `install-job.yaml` | One-shot Job that installs KubeVirt operator v1.7.1 + CDI v1.62.0 |
| `kubevirt-manager.yaml` | Web UI + built-in noVNC console ([kubevirt-manager](https://kubevirt-manager.io/)) |
| `windows-vm.yaml` | DataVolumes (Windows ISO, empty disk, VirtIO drivers) + VirtualMachine + RDP Service |

## Prerequisites

- Node must support hardware virtualization (`/dev/kvm` available)
- Sufficient RAM: Windows needs 8Gi guest + ~2Gi overhead
- Storage: ~70Gi total (8Gi ISO + 60Gi disk + 1Gi VirtIO drivers)
- `local-path-provisioner` StorageClass (installed separately)

### Verify KVM support on the node

```bash
ssh packet@192.168.0.125
ls -la /dev/kvm
# Should show: crw-rw-rw- 1 root kvm ... /dev/kvm

# If missing, load the module:
sudo modprobe kvm_intel  # or kvm_amd
```

### Install local-path-provisioner (required for DataVolumes)

```bash
K="sudo /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml"
$K apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
$K patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## Workflow

1. **Flux deploys** the namespaces, install Jobs, and Windows VM manifests
2. **Install Jobs** apply KubeVirt operator, CDI, and kubevirt-manager from upstream
3. **CDI imports** the Windows ISO and VirtIO drivers ISO into DataVolumes
4. **Access kubevirt-manager** web UI:

```bash
# Get the NodePort
$K get svc kubevirt-manager -n kubevirt-manager
# Open: http://192.168.0.125:<nodeport>
```

5. **Start the VM** via kubevirt-manager UI or CLI:

```bash
$K patch vm windows -n kubevirt-vms --type merge -p '{"spec":{"running":true}}'
```

6. **Open VNC console** in kubevirt-manager:
   - Navigate to Virtual Machines → namespace `kubevirt-vms` → `windows`
   - Click the Console (monitor) icon
7. **Install Windows** — during setup, load VirtIO drivers from the
   second CD-ROM (E: drive → `viostor\2k22\amd64`) so Windows can see the virtio disk.
   Also load the network driver from `NetKVM\2k22\amd64`.
8. **Connect via RDP** after install:

```bash
# Get the NodePort
$K get svc windows-rdp -n kubevirt-vms -o jsonpath='{.spec.ports[0].nodePort}'
# Connect: mstsc /v:192.168.0.125:<nodeport>
```

## After Windows Install

Once Windows is installed, remove the ISO volumes to free space and simplify the VM:

```bash
K="sudo /var/lib/rancher/rke2/bin/kubectl --kubeconfig /etc/rancher/rke2/rke2.yaml"

# Stop VM
$K patch vm windows -n kubevirt-vms --type merge -p '{"spec":{"running":false}}'

# Delete the ISO DataVolumes
$K delete dv windows-iso virtio-drivers -n kubevirt-vms

# Edit the VM to remove the cdrom disks and volumes, then restart
$K edit vm windows -n kubevirt-vms
```

## Notes

- The VM is created with `running: false` to avoid starting before ISOs are imported
- Windows ISO URL points to Microsoft's Server 2022 Eval — replace with your own if needed
- VirtIO drivers are required for Windows to see the disk and network
- TPM 2.0 is enabled (required for Windows 11, optional for Server)
- Hyper-V enlightenments are enabled for better performance
- RDP is exposed via NodePort — accessible from meshnetwork
- kubevirt-manager provides a browser-based VNC console via native KubeVirt API websocket
