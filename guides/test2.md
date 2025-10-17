## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Installation Setup](#pre-installation-setup)
3. [Disk Layout Planning](#disk-layout-planning)
4. [Disk Preparation](#disk-preparation)
5. [Partition Creation](#partition-creation)
6. [Encryption Setup]
7. [LVM Configuration]
8. [Filesystem Creation]
9. [Mounting Filesystems]
10. [Base System Installation]
11. [System Configuration]
12. [Boot Configuration]
13. [Secure Boot Setup]
14. [Package Management]
15. [System Services]
16. [Finalization]
17. [Post-Installation]
18. [Maintenance]
19. [Troubleshooting]
20. [Resources]

## Prerequisites
**Hardware**:
- ASUS ROG Zephyrus G16
- 32GB RAM
- NVIDIA RTX 4070 (8GB VRAM)
- NVMe SSD (identified as /dev/nvme0n1)

**Requirements**:
- UEFI firmware (not Legacy BIOS)
- Active internet connection
- Arch Linux ISO booted in UEFI mode
- Basic command line familiarity


>[!WARNING]
>This guide will **completely erase** `/dev/nvme0n1`. Back up all data before proceeding. Verify your target drive with `lsblk -d | grep disk`.

>[!TIP]
>Confirm UEFI boot mode: ls `/sys/firmware/efi/efivars` should show files. If the directory doesn't exist, you booted in Legacy mode.


## Pre-Installation Setup

### Network Configuration

**Identify your interface**:

```bash
ip link
```

**For Ethernet**:

```bash
# Replace with your network details
ip addr add 192.168.1.100/24 dev enp0s31f6
ip route add default via 192.168.1.1 dev enp0s31f6
echo "nameserver 1.1.1.1" > /etc/resolv.conf
```

**For WiFi**:

```bash
iwctl

# Inside iwctl:
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect "YourSSID"
exit
```

**Test connectivity**:

```bash
ping -c 3 archlinux.org
```

### Time Synchronization

```bash
timedatectl set-ntp true
timedatectl set-timezone Europe/Paris
timedatectl status
```

>[!IMPORTANT]
>Accurate time is critical for HTTPS certificate validation and proper filesystem timestamps.

## Disk Layout Planning

### Storage Structure

```bash
/dev/nvme0n1
└─ nvme0n1p1 Windows partition (untouched)

/dev/nvme1n1
├─ nvme1n1p1    ESP (1 GiB, FAT32) - shared with Windows
├─ nvme1n1p2    LUKS2 → cryptos → LVM (leo-os)
│                └─ root (50G), var (30G), home (100G), swap (32G), data (50G)
└─ nvme1n1p3    LUKS2 → cryptvms → LVM (leo-vms) [OPTIONAL]
                 └─ vms (remaining space)
```

>[!NOTE]
>**Partition purposes**:
>- **Windows (nvme0n1p1)**: Existing Windows installation (DO NOT MODIFY)
>- **ESP (nvme1n1p1)**: Shared boot partition for both Windows and Linux
>- **System (nvme1n1p2)**: Encrypted Linux system with LVM for flexibility
>- **VMs (nvme1n1p3)**: Optional isolated storage for virtual machines


### Volume Breakdown

**System volumes in leo-os**:
- **root**: Core system files (/)
- **var**: Logs, caches, package databases (/var)
- **home**: User files and configurations (/home)
- **swap**: Hibernation support (size = RAM)
- **data**: Nested encrypted storage for sensitive documents

**VM volume in leo-vms (optional)**:
- **vms**: Virtual machine disk images


>[!TIP]
>**Customization options**:
>- Skip nvme0n1p4 if you don't use VMs - expand nvme0n1p3 instead
>- Skip data volume if you don't need nested encryption
>- Adjust volume sizes based on your usage patterns
>- For single-boot systems, you can use a smaller ESP (512MB)

### Why This Layout?

**Security**:
- Everything except ESP is encrypted with LUKS2
- Nested encryption for data adds extra protection layer
- Separate VM storage isolates potential security risks

**Flexibility**:
- LVM allows resizing volumes without repartitioning
- Independent backup strategies for system vs user data
- Easy to add new volumes later

**Performance**:
- VMs on separate partition reduce I/O contention with system
- Dedicated swap improves hibernation reliability

## Disk Preparation

>[!WARNING]
>The following operations are **irreversible**. Triple-check you're targeting the correct drive.

### Verify Target Drive

```bash
lsblk -d | grep disk
fdisk -l /dev/nvme0n1
```

Confirm `/dev/nvme0n1` is your intended target and note existing Windows partitions.

### Secure Erasure

**For Linux partitions only (preserving Windows)**:
Skip to [Partition Creation](#partition-creation) if Windows already exists.

**For complete drive erasure (fresh install, no Windows yet)**:

```bash
# Fastest method for SSDs
blkdiscard -f /dev/nvme0n1
```

**Alternative: Hardware sanitization (most secure)**:

```bash
# Check sanitize capabilities
nvme id-ctrl /dev/nvme0n1 | grep -i sanitize

# Cryptographic erase (if supported)
nvme format /dev/nvme0n1 --ses=1

# OR block erase
nvme sanitize /dev/nvme0n1 --sanact=1
```

>[!WARNING]
>Hardware sanitization can take hours and requires a power cycle. The drive is inaccessible during this process.


















