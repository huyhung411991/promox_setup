# Proxmox

## I. Setup Proxmox

### 1. Configure BIOS
- Enable **IOMMU**: Intel VT-d (Virtualization Technology for Directed I/O)
- Enable **Above 4G Decoding**
- Enable **SR-IOV** (mostly AMD system)
- Disable **CSM** to enforce using UEFI mode
   - All storage devices must use the GPT partition scheme
   - All GPUs must support UEFI
   - Bad HDMI / DisplayPort cables may result in no display output
- AMD systems: if Promox installer fails to boot, try restarting the system a few times
- Intel systems with NVIDIA RTX GPUs: if Promox installer fails to boot, try using older Proxmox 9.1 instead

### 2. Install Proxmox
- When selecting the target disk, choose **Options** and set `maxroot` to limit the root partition size.

### 3. Access GUI Control Panel from Another PC
- **Username:** `root`
- **Password:** Set during installation

### 4. Change APT Repositories
1. Navigate to:
   - `Datacenter > Node > Updates > Repositories`
2. Disable:
   - enterprise repo
   - pve-enterprise repo
3. Add:
   - **No-Subscription**
   - **Ceph Squid No-Subscription**
4. Open Shell:
   - `Datacenter > Node > Shell`
5. Update packages:
   ```bash
   apt update
   apt upgrade
   ```

---

## II. Install VM

### 1. Add Ubuntu ISO
Navigate to:

`Datacenter > Node > local > ISO Images`

- Upload from PC, or
- Download from URL

### 2. Create VM

#### General
- **VMID:** e.g., 801, 802, etc., on vsw8 node
- **Name:** e.g., vm1, vm2, etc.

#### OS
- **ISO image:** For OS installation
- Switch to **Do not use any media** after finishing installation

#### System
- **Graphic card**: Default if using virtual display, none if using GPU Passthrough and physical display
- **Machine:** q35
- **BIOS:** OVMF
- **EFI Storage:** Select local-vlm storage option
- **SCSI Controller:** VirtIO SCSI single
- **Qemu Agent:** Enable for better Proxmox integration

#### Disk
- **Bus:** SCSI (recommended)
- **Disk size (GiB)**
- **Discard:** Enable if using SSD

#### CPU
- **Cores:** Number of threads
- **Type:** host

#### Memory
<!--- - **Memory (MiB):** Reserve 4-8 (GiB) for the host, then allocate the remaining memory to the virtual machines (MiB = GiB * 1024) -->
- **Memory (MiB):** Reserve 10% for the host, then allocate the remaining memory to the virtual machines (MiB = GiB * 1024)
- **Advanced > Balloning Device:** Disable as it doesn't work with PCI Passthrough

### 3. Setup GPU Passthrough
Navigate to:

`Datacenter > Node > VM > Hardware > Add > PCI Device`

Settings:
- **Raw Device**
- **Device:** Select NVIDIA GPU
- **All Functions:** Enable
- **Advanced > PCI-Express:** Enable

### 4. Setup Storage Passthrough

Open:

`Datacenter > Node > Shell`

Find disk ID:

```bash
lshw -class disk -class storage
```

Hot-Plug/Add physical device as new virtual SCSI disk:

```bash
qm set {VMID} -scsi{number} /dev/disk/by-id/ata-vendor_disk_id
```

Hot-Unplug/Remove virtual disk:

```bash
qm unlink {VMID} --idlist scsi{number}
```

Partition passthrough:

```bash
/dev/disk/by-id/ata-vendor_disk_id_part_number
```

### 5. Install OS on VM

1. Click **Start**
2. Open **>_ Console**: show VM display

---

## III. Setup Cluster

### 1. First Node

Navigate to:

`Datacenter > Cluster > Create Cluster`

- Set **Cluster Name**
- Copy **Join Information**

### 2. Other Nodes

Navigate to:

`Datacenter > Cluster > Join Cluster`

- Paste Join **Information**
- Enter Master Node **Password**

---

## IV. Remove Node from Cluster

### 1. Remove a Node on Remaining Node(s)

1. Temporarily disconnect the node being removed.
2. Open:

   `Datacenter > (Any Remaining) Node > Shell`

3. For a 2-node cluster, need to lower the required quorum count to allow configuration edits:

   ```bash
   pvecm expected 1
   ```

4. Remove the node:

   ```bash
   pvecm delnode <removed_node>
   ```

5. Remove leftover node configuration:

   ```bash
   rm -rf /etc/pve/nodes/<removed_node>
   ```

### 2. Remove Cluster Configuration on Removed Node

Open:

`Datacenter > (Removed) Node > Shell`

Remove cluster configuration by running:

```bash
systemctl stop pve-cluster corosync
pmxcfs -l
rm /etc/pve/corosync.conf
rm -r /etc/corosync/*
killall pmxcfs
systemctl start pve-cluster
```

Remove leftover node configuration:

```bash
rm -rf /etc/pve/nodes/<remaining_nodes>
```

## V. Others

### 1. Check Device ID of GPU being used for Proxmox's CLI
```bash
ls -l /sys/class/graphics/fb0/device
```

```
For vsw7: 
1: 41
2: 61
3: 42 (vertical card)
4: 01
```