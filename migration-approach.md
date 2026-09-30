# Windows VM Migration: KVM to OpenShift Virtualization 4.21 (SNO)

## Overview

Migrate Windows VMs with BitLocker/TPM disk encryption from a standalone KVM (libvirt) host to OpenShift Virtualization running on Single Node OpenShift (SNO) 4.21.

**Key challenge:** TPM state is not portable. The virtual TPM on the source KVM host and the vTPM provided by OpenShift Virtualization are entirely separate devices. BitLocker keys sealed to the source TPM cannot be unsealed on the destination. This must be handled explicitly before migration or the VMs will not boot.

---

## Phase 0: Inventory and Assessment

### On the KVM host

```bash
# List all VMs and their state
virsh list --all

# For each VM, gather details
virsh dominfo <vm-name>
virsh domblklist <vm-name>
virsh dumpxml <vm-name> > <vm-name>.xml

# Check disk sizes (actual vs allocated)
qemu-img info /var/lib/libvirt/images/<disk>.qcow2

# Check if TPM is configured
virsh dumpxml <vm-name> | grep -A5 tpm
```

### Capture for each VM

| Field | Notes |
|---|---|
| VM name | |
| Windows version | Win10, Win11, Server 2019/2022, etc. |
| vCPUs / RAM | SNO capacity planning |
| Disk count and sizes | Total storage needed |
| Disk format | qcow2, raw, etc. |
| Network config | Static IP, VLAN, bonding, etc. |
| TPM version | 2.0 expected |
| BitLocker status | Which volumes are encrypted |
| Installed drivers | virtio already present? |
| Applications | Any license/activation concerns |

### SNO capacity check

Single Node OpenShift has finite resources. Sum all VM requirements and verify the node can handle the load:

```bash
# On the SNO node
oc describe node | grep -A10 "Allocated resources"
oc get storageclass
```

Consider that OpenShift Virt control plane + operators consume ~16-24 GB RAM and several cores on SNO.

---

## Phase 1: Prepare the OpenShift Virt Environment

### 1.1 Install OpenShift Virtualization Operator

```bash
# Verify the operator is available
oc get packagemanifest kubevirt-hyperconverged -n openshift-marketplace

# Create namespace
oc create namespace openshift-cnv
```

Apply the operator subscription:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  channel: stable
  name: kubevirt-hyperconverged
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

Then create the HyperConverged custom resource:

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec: {}
```

### 1.2 Configure Storage

For SNO, local storage is typical. Options:

- **LVM Storage (lvms)** -- recommended for SNO. The LVMS operator provisions thinly-provisioned local volumes.
- **NFS** -- if an external NFS server is available.
- **ODF (single-node)** -- heavier, less common on SNO.

For LVM Storage:

```yaml
apiVersion: lvm.topolvm.io/v1alpha1
kind: LVMCluster
metadata:
  name: lvmcluster
  namespace: openshift-storage
spec:
  storage:
    deviceClasses:
      - name: vg1
        thinPoolConfig:
          name: thin-pool-1
          sizePercent: 90
          overprovisionRatio: 10
```

Create a StorageClass that supports RWX (needed for live migration, though less relevant on SNO):

Ensure the default StorageClass supports `VolumeMode: Block` -- this is strongly preferred for VM disks in OpenShift Virt.

### 1.3 Configure Networking

For basic connectivity, the default pod network works. For more complex scenarios (bridged networking to match the KVM host's network topology):

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-policy
spec:
  desiredState:
    interfaces:
      - name: br1
        type: linux-bridge
        state: up
        bridge:
          port:
            - name: ens224  # adjust to your interface
```

Then create a NetworkAttachmentDefinition:

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: bridge-network
  namespace: <vm-namespace>
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "bridge-network",
      "type": "bridge",
      "bridge": "br1",
      "vlan": 100,
      "ipam": {}
    }
```

### 1.4 Verify vTPM Support

OpenShift Virtualization supports vTPM natively. Verify the feature gate is enabled:

```bash
oc get kubevirt kubevirt-kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{.spec.configuration.developerConfiguration.featureGates}'
```

No special configuration is needed in OCP 4.21 -- vTPM is GA. Persistent vTPM state is stored in a PVC automatically.

---

## Phase 2: Prepare the Windows VMs (On the Source KVM Host)

This is the most critical phase. Every step here must complete successfully before migrating.

### 2.1 Handle BitLocker / TPM

**Option A: Suspend BitLocker (Recommended -- simpler, reversible)**

Inside each Windows VM:

```powershell
# Check BitLocker status
manage-bde -status

# Suspend BitLocker protection (does NOT decrypt -- leaves data encrypted
# but stores the key in the clear so the volume can boot without TPM)
manage-bde -protectors -disable C:

# Verify status shows "Protection Off" but "Encryption: Fully Encrypted"
manage-bde -status C:
```

> **Important:** Suspending BitLocker survives exactly one reboot by default on some configurations. To ensure it stays suspended across multiple reboots (migration will involve several), specify a reboot count:
>
> ```powershell
> manage-bde -protectors -disable C: -RebootCount 0
> ```
> A RebootCount of `0` keeps protection suspended indefinitely until manually re-enabled.

**Option B: Fully decrypt (More time, but safest)**

```powershell
manage-bde -off C:

# Monitor decryption progress
manage-bde -status C:
```

This can take hours for large volumes. Only necessary if Option A proves problematic.

### 2.2 Install VirtIO Drivers (If Not Already Present)

If the KVM VMs are already using virtio devices, the drivers are likely present. Verify:

```powershell
# In Windows, check Device Manager for:
# - "Red Hat VirtIO SCSI controller"
# - "Red Hat VirtIO Ethernet Adapter"
```

If the VMs are using emulated IDE/e1000 devices, install virtio drivers before migration:

1. Download the latest virtio-win ISO from Red Hat: `virtio-win.iso`
2. Attach the ISO to the VM on the KVM host
3. Install all drivers from the ISO (balloon, network, SCSI, serial, qxl/display)
4. Switch the VM's disk bus to virtio and NIC to virtio on the KVM host to verify the drivers work
5. Reboot and confirm the VM operates correctly with virtio devices

### 2.3 Install QEMU Guest Agent

Inside each Windows VM, install the QEMU Guest Agent from the virtio-win ISO. This provides:
- Filesystem freeze/thaw for consistent snapshots
- Network information reporting to OpenShift Virt
- Graceful shutdown support

### 2.4 Remove KVM-specific Configurations

- Remove any spice/QXL display agents if present
- Remove any libvirt-specific guest tools
- Note down any custom CPU pinning, NUMA topology, or hugepage configs (these will be reconfigured in OpenShift Virt)

### 2.5 Shut Down the VMs

```bash
# Graceful shutdown
virsh shutdown <vm-name>

# Verify the VM is off
virsh domstate <vm-name>
```

---

## Phase 3: Migrate the Disk Images

### Migration Approach: Manual Disk Export + virtctl Upload

For standalone KVM/libvirt hosts, the Migration Toolkit for Virtualization (MTV/Forklift) does not have a direct "libvirt" source provider. The most reliable approach is manual disk migration.

> **Note on MTV:** MTV supports VMware vSphere, Red Hat Virtualization (oVirt), OpenStack, and OVA as source providers. If you have a small number of VMs (typical for a KVM host), the manual approach is straightforward and avoids standing up an intermediary platform.

### 3.1 Export and Convert Disk Images

On the KVM host:

```bash
# If disks are qcow2 with backing files, flatten them
qemu-img convert -f qcow2 -O qcow2 /var/lib/libvirt/images/<vm-disk>.qcow2 /tmp/<vm-disk>-flat.qcow2

# Or convert to raw for upload (raw is often preferred for block-mode PVCs)
qemu-img convert -f qcow2 -O raw /var/lib/libvirt/images/<vm-disk>.qcow2 /tmp/<vm-disk>.img

# Verify the output
qemu-img info /tmp/<vm-disk>-flat.qcow2
```

### 3.2 Transfer Disk Images to a Location Accessible from the SNO Cluster

Options:
- **SCP/rsync** to the SNO node or a bastion host
- **HTTP server** -- place images on an HTTP server and use CDI (Containerized Data Importer) to pull them
- **S3-compatible storage** -- CDI supports S3 as a source
- **Local transfer** -- if you have direct access to the SNO node

```bash
# Example: rsync to a staging location
rsync -avP /tmp/<vm-disk>.img user@sno-node:/tmp/vm-images/
```

### 3.3 Upload Disk Images Using virtctl

```bash
# Create a PVC and upload the disk image in one step
virtctl image-upload dv <vm-name>-disk \
  --size=<disk-size>Gi \
  --image-path=/tmp/<vm-disk>.img \
  --storage-class=<your-storage-class> \
  --volume-mode=Block \
  --insecure \
  --namespace=<vm-namespace>

# Monitor the upload
oc get dv <vm-name>-disk -n <vm-namespace> -w
```

Alternatively, create a DataVolume that imports from an HTTP source:

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: win-server-disk
  namespace: <vm-namespace>
spec:
  source:
    http:
      url: "http://<staging-server>/vm-images/<vm-disk>.img"
  storage:
    accessModes:
      - ReadWriteOnce
    volumeMode: Block
    resources:
      requests:
        storage: 100Gi
    storageClassName: <your-storage-class>
```

---

## Phase 4: Create VirtualMachine Resources in OpenShift Virt

### 4.1 VM Manifest with vTPM

For each Windows VM, create a VirtualMachine CR. Key elements:
- Match CPU/memory to the source VM
- Reference the uploaded disk PVC
- Enable vTPM
- Use virtio devices
- Configure Windows-specific settings

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: win-server-01
  namespace: <vm-namespace>
  labels:
    app: win-server-01
spec:
  running: false  # Don't start yet
  template:
    metadata:
      labels:
        kubevirt.io/vm: win-server-01
    spec:
      domain:
        cpu:
          cores: 4
          sockets: 1
          threads: 1
        memory:
          guest: 8Gi
        devices:
          disks:
            - name: rootdisk
              disk:
                bus: virtio
              bootOrder: 1
          interfaces:
            - name: default
              masquerade: {}
              # Or for bridged networking:
              # bridge: {}
          tpm:
            persistent: true
        features:
          smm:
            enabled: true
          hyperv:
            relaxed:
              enabled: true
            vapic:
              enabled: true
            spinlocks:
              enabled: true
              spinlocks: 8191
            vpindex:
              enabled: true
            runtime:
              enabled: true
            synic:
              enabled: true
            stimer:
              pinned:
                enabled: true
              enabled: true
            reset:
              enabled: true
            frequencies:
              enabled: true
            reenlightenment:
              enabled: true
            tlbflush:
              enabled: true
            ipi:
              enabled: true
        clock:
          utc: {}
          timer:
            hpet:
              present: false
            pit:
              tickPolicy: delay
            rtc:
              tickPolicy: catchup
            hyperv: {}
        firmware:
          bootloader:
            efi:
              secureBoot: true  # Required for BitLocker with TPM
      networks:
        - name: default
          pod: {}
          # Or for bridged:
          # multus:
          #   networkName: bridge-network
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: win-server-01-disk
```

> **Critical settings for Windows with TPM/BitLocker:**
> - `spec.domain.devices.tpm.persistent: true` -- provides a vTPM with persistent state
> - `spec.domain.features.smm.enabled: true` -- System Management Mode, required for UEFI Secure Boot
> - `spec.domain.firmware.bootloader.efi.secureBoot: true` -- BitLocker with TPM typically requires Secure Boot
> - Hyper-V enlightenments -- significantly improve Windows performance on KVM/KubeVirt

### 4.2 Apply and Verify (Don't Start Yet)

```bash
oc apply -f win-server-01-vm.yaml

# Verify the VM object was created
oc get vm win-server-01 -n <vm-namespace>

# Verify the PVC is bound
oc get pvc win-server-01-disk -n <vm-namespace>
```

---

## Phase 5: Boot, Validate, and Re-enable BitLocker

### 5.1 Start the VM

```bash
virtctl start win-server-01 -n <vm-namespace>

# Watch the VMI come up
oc get vmi win-server-01 -n <vm-namespace> -w

# Access the console
virtctl console win-server-01 -n <vm-namespace>

# Or use VNC
virtctl vnc win-server-01 -n <vm-namespace>
```

### 5.2 Post-Boot Validation

Once Windows boots (it should, since BitLocker is suspended/decrypted):

1. **Verify Windows boots correctly** -- log in via VNC console
2. **Check Device Manager** -- ensure no unknown devices, virtio drivers working
3. **Check network connectivity** -- verify IP assignment, DNS, routing
4. **Verify disk access** -- all volumes visible and accessible
5. **Check Windows activation** -- hardware change may trigger reactivation
6. **Verify applications** -- key applications start and function

### 5.3 Re-enable BitLocker with the New vTPM

Inside the Windows VM (via VNC or RDP):

```powershell
# Check current BitLocker status
manage-bde -status C:

# If BitLocker was suspended (Option A), re-enable protection
manage-bde -protectors -enable C:

# If BitLocker was fully decrypted (Option B), re-encrypt
manage-bde -on C: -RecoveryPassword -SkipHardwareTest

# Verify the TPM protector is present
manage-bde -protectors -get C:

# You should see a "TPM" protector listed, bound to the new vTPM
```

> **Important:** Save the new BitLocker recovery keys. The old recovery keys from the source KVM host's vTPM are no longer valid.

```powershell
# Backup recovery key
manage-bde -protectors -get C: > C:\bitlocker-recovery-key.txt
# Or back up to AD/Azure AD if domain-joined
```

### 5.4 Reboot and Verify

```powershell
# Reboot the VM to confirm BitLocker + vTPM works across reboots
Restart-Computer
```

The VM should boot without prompting for a recovery key. If it does prompt, use the recovery key generated in step 5.3 and troubleshoot the TPM binding.

---

## Phase 6: Cleanup

1. **Verify all VMs** are running correctly on OpenShift Virt
2. **Keep source KVM VMs** powered off (not deleted) for at least 2-4 weeks as a rollback option
3. **Document the new recovery keys** and store them securely
4. **Update any monitoring, backup, or management systems** to point to the new VMs
5. **Update DNS/DHCP** if IP addresses changed

---

## Troubleshooting

### VM won't boot -- BitLocker recovery screen

- The BitLocker suspension didn't persist. Boot into recovery, enter the **original** recovery key (from the KVM host era), then suspend BitLocker again and re-seal to the new vTPM.

### VM boots but shows "No bootable device"

- Ensure the VM is configured for EFI boot (not BIOS/legacy)
- Verify the disk was uploaded correctly: `qemu-img check` on the source
- Check that the PVC has the correct size (must be >= source disk size)
- Verify VolumeMode is Block, not Filesystem

### Poor performance

- Ensure virtio drivers are installed (not falling back to emulated IDE)
- Verify Hyper-V enlightenments are enabled in the VM spec
- On SNO, check for resource contention: `oc adm top node`

### Windows activation issues

- Hardware change from migration may trigger reactivation
- For KMS-activated Windows: ensure the KMS server is reachable from the new network
- For OEM/retail: may need to contact Microsoft for reactivation

### vTPM PVC issues

- vTPM state is stored in a small PVC. Ensure the StorageClass can provision small (1Mi) PVCs
- Check: `oc get pvc -n <vm-namespace> | grep tpm`

---

## Migration Order Recommendation

1. **Start with a non-critical/test VM** -- validate the entire process end-to-end
2. **Migrate similar VMs in batches** -- same OS version, same network segment
3. **Save domain controllers and critical infrastructure VMs for last** -- these need the most validation
4. **Avoid migrating VMs that are actively serving users during business hours**

---

## Summary Checklist

- [ ] Inventory all KVM VMs (CPU, RAM, disk, TPM, BitLocker status)
- [ ] Verify SNO has sufficient resources
- [ ] Install and configure OpenShift Virtualization operator
- [ ] Configure storage (LVMS recommended for SNO)
- [ ] Configure networking (bridge if needed)
- [ ] Suspend or decrypt BitLocker on each Windows VM
- [ ] Install virtio drivers and QEMU guest agent (if missing)
- [ ] Shut down source VMs
- [ ] Export and flatten disk images
- [ ] Upload disks to OpenShift Virt via virtctl or CDI
- [ ] Create VirtualMachine CRs with vTPM, EFI, Secure Boot, and Hyper-V enlightenments
- [ ] Boot VMs and validate functionality
- [ ] Re-enable BitLocker, sealed to the new vTPM
- [ ] Save new recovery keys
- [ ] Reboot and verify BitLocker + vTPM works
- [ ] Keep source VMs as rollback for 2-4 weeks
