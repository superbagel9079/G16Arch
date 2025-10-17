## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Installation Setup](#pre-installation-setup)
3. [Disk Layout Planning]
4. [Disk Preparation]
5. [Partition Creation]
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
