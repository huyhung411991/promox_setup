# Proxmox

## I. Setup Proxmox

### 1. Configure BIOS
- Enable **IOMMU**: Intel VT-d (Virtualization Technology for Directed I/O)
  - On Asus X299 mainboard: `Advanced > System Agent (SA) Configuration > Intel@ VT for Directed I/O (VT-d) > Enabled`
- Enable **Above 4G Decoding**
  - On Asus X299 mainboard: `Advanced > PCI Subsystem Settings > Above 4G Decoding > Enabled`
- Enable **SR-IOV** (mostly AMD system)
- Disable **CSM** to enforce using UEFI mode
   - Requirements:
     - All storage devices must use the GPT partition scheme
     - All GPUs must support UEFI
     - Bad HDMI / DisplayPort cables may result in no display output
   - On Asus X299 mainboard: `Boot > CSM > Launch CSM > Disabled`
   - On Asus X299 mainboard with **4 or more** GPUs with high VRAM, disabling **CSM** may cause system unable to boot. Temporary solution:
     - Navigate to `Boot > CSM`
     - `Launch CSM > Enabled`
     - `Boot Device Control > UEFI and Legacy OPROM`
     - `Boot from Network Devices > UEFI driver first`
     - `Boot from Storage Devices > UEFI driver first`
     - `Boot from PCI-E/PCI Expansion Devices > UEFI driver first`

### 2. Install Proxmox
- Notices:
  - On the first boot after Proxmox USB installer creation, GPT header corruption may occur.
    - On Asus X299 mainboard: `Boot > Boot Configuration > Next Boot Recovery Action > Recovery`
  - On AMD systems: If Promox installer fails to boot, try restarting the system a few times
  - On Intel systems with NVIDIA RTX GPUs: Uf Promox installer fails to boot, try using older Proxmox 9.1 instead
- When selecting the target disk, choose **Options** and set **maxroot** to `20GB` to limit the root partition size
- Set Hostname to `vsw#.local` and configure network information

### 3. Access GUI Control Panel from Another PC
- **Username:** root
- **Password:** Set during installation

### 4. Change APT Repositories
- Navigate to: `Datacenter > Node > Updates > Repositories`
- Disable:
   - enterprise repo
   - pve-enterprise repo
- Add:
   - **No-Subscription**
   - **Ceph Squid No-Subscription**
- Open Shell: `Datacenter > Node > Shell`
- Update packages:

```bash
apt update
apt upgrade
```

---

## II. Install VM
### 1. Add Ubuntu ISO
- Navigate to `Datacenter > Node > local > ISO Images`
- Upload from PC, or
- Download from URL

### 2. Create VM
#### a) General
- **VMID:** e.g., 801, 802, etc., on vsw8 node
- **Name:** e.g., vm1, vm2, etc.

#### b) OS
- **ISO image:** For OS installation
- Switch to **Do not use any media** after finishing installation

#### d) System
- **Graphic card**: Default if using virtual display, none if using GPU Passthrough and physical display
- **Machine:** q35
- **BIOS:** OVMF
  - (Need verification) This must match the node's BIOS setting (CSM disabled = UEFI enforced &rarr; OVMF).
  - Temporarily solution in case **CSM** cannot be disabled in BIOS: Set this to SeaBIOS
- **EFI Storage:** Select local-vlm storage option. Only available when BIOS is set to OVMF.
- **SCSI Controller:** VirtIO SCSI single
- **Qemu Agent:** Enable for better Proxmox integration

#### e) Disk
- **Bus:** SCSI (recommended)
- **Disk size (GiB)**
- **Discard:** Enable if using SSD

#### f) CPU
- **Cores:** Number of threads
- **Type:** host

#### g) Memory
<!--- - **Memory (MiB):** Reserve 4-8 (GiB) for the host, then allocate the remaining memory to the virtual machines (MiB = GiB * 1024) -->
- **Memory (MiB):** Reserve 10% for the host, then allocate the remaining memory to the virtual machines (MiB = GiB * 1024)
- **Advanced > Balloning Device:** Disable as it doesn't work with PCI Passthrough

### 3. Setup GPU Passthrough
- Navigate to: `Datacenter > Node > VM > Hardware > Add > PCI Device`
- Settings:
  - **Raw Device**
  - **Device:** Select NVIDIA GPU
  - **All Functions:** Enable
  - **Advanced > PCI-Express:** Enable

### 4. Setup Storage Passthrough
- Open: `Datacenter > Node > Shell`
- Find disk ID:

```bash
lshw -class disk -class storage
```

- Hot-Plug/Add physical device as new virtual SCSI disk:

```bash
qm set {VMID} -scsi{number} /dev/disk/by-id/ata-vendor_disk_id
```

- Hot-Unplug/Remove virtual disk:

```bash
qm unlink {VMID} --idlist scsi{number}
```

- Partition passthrough:

```bash
/dev/disk/by-id/ata-vendor_disk_id_part_number
```

### 5. Install OS on VM

- Click **Start**
- Open **>_ Console**: Show VM display

---

## III. Setup Cluster

### 1. First Node
- Navigate to: `Datacenter > Cluster > Create Cluster`
- Set **Cluster Name**
- Copy **Join Information**

### 2. Other Nodes
- Navigate to: `Datacenter > Cluster > Join Cluster`
- Paste Join **Information**
- Enter Master Node **Password**

---

## IV. Remove Node from Cluster

### 1. Remove a Node on Remaining Node(s)
- Temporarily disconnect the node being removed.
- Open: `Datacenter > (Any Remaining) Node > Shell`
- For a 2-node cluster, need to lower the required quorum count to allow configuration edits:

```bash
pvecm expected 1
```

- Remove the node:

```bash
pvecm delnode <removed_node>
```

- Remove leftover node configuration:

```bash
rm -rf /etc/pve/nodes/<removed_node>
```

### 2. Remove Cluster Configuration on Removed Node

- Open: `Datacenter > (Removed) Node > Shell`
- Remove cluster configuration by running:

```bash
systemctl stop pve-cluster corosync
pmxcfs -l
rm /etc/pve/corosync.conf
rm -r /etc/corosync/*
killall pmxcfs
systemctl start pve-cluster
```

- Remove leftover node configuration:

```bash
rm -rf /etc/pve/nodes/<remaining_nodes>
```

## V. Others

### 1. Change Node IP Address

- Check current IPs & MAC Address:
```bash
ifconfig
```
- Navigate to: `Datacenter > Node > System > Network`, edit **vmbr0**
  - **IPv4/CIDR**
  - **Gateway**
  - **Bridge ports**
- Navigate to: `Datacenter > Node > System > Hosts`, change IP address in the second line and save
- Return to `Datacenter > Node > System > Network`, click **Apply Configuration**
- If node is in a cluster, change IP address in `/etc/pve/corosync.conf` on one node and reboot both nodes (NEED TESTING)

### 2. Downgrade Proxmox Kernel
#### a) Install older kernel via APT:
- If only the latest kernel is installed, install the desired older version:
```bash
apt install pve-kernel-<target-version>-pve-signed
```

#### b) Reboot to the older kernel
Choose one of the following:
- Temporary selection with GRUB: select the target kernel from `GRUB menu > Advanced option`

- Permanent selection on UEFI systems: Pin the target kernel:
```bash
proxmox-boot-tool kernel pin <target-version>-pve
```

- Permanent selection with GRUB: Set `GRUB_DEFAULT` in `/etc/default/grub` to the target kernel index path (e.g., `GRUB_DEFAULT='1>2'`) and run `update-grub`

After rebooting, confirm the current kernel using:
```bash
uname -r
```

#### c) (Optional) Remove the newer kernel
- Verify that the installed `proxmox-kernel-x.y` package matches the target version `x.y.z-t`

- If not, install the required version:
```bash
apt install --allow-downgrades proxmox-kernel-<x.y>=<target-version>
```

- Remove newer kernel:
```bash
apt purge prox-kernel-<newer-version>
```


### 3. Check Device ID of GPU being used for Proxmox's CLI

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