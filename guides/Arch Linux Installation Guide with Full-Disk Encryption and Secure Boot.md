## Table of Contents

1. [Prerequisites and Assumptions](#prerequisites-and-assumptions)
   - [System Requirements](#system-requirements)

2. [Network and Time Configuration](#network-and-time-configuration)
   - [Network Setup](#network-setup)
   - [Time Synchronization](#time-synchronization)

3. [Disk Preparation](#disk-preparation)
   - [Understanding the Storage Layout](#understanding-the-storage-layout)
   - [Partition Purposes](#partition-purposes)
   - [Benefits of This Approach](#benefits-of-this-approach)
   - [Secure Drive Erasure](#secure-drive-erasure)

4. [Disk Partitioning](#disk-partitioning)
   - [Create GPT Partition Table](#create-gpt-partition-table)
   - [Create Partitions](#create-partition)

5. [Encryption Setup](#encryption-setup)
   - [LUKS2 Configuration](#luks2-configuration)
   - [Open Encrypted Containers](#open-encrypted-containers)

6. [LVM Configuration](#lvm-configuration)
   - [Initialize Physical Volumes](#initialize-physical-volumes)
   - [Create Volume Groups](#create-volume-groups)
   - [Create Logical Volumes](#create-logical-volumes)
   - [Optional: Nested Encryption for User Data](#optional-nested-encryption-for-user-data)

7. [Filesystem Creation](#filesystem-creation)
   - [Format Partitions](#format-partitions)

8. [Mount Filesystems](#mount-filesystems)
   - [Create Mount Hierarchy](#create-mount-hierarchy)

9. [Base System Installation](#base-system-installation)
   - [Install Essential Packages](#install-essential-packages)
   - [Generate Filesystem Table](#generate-filesystem-table)

10. [System Configuration](#system-configuration)
    - [Enter Chroot Environment](#enter-chroot-environment)
    - [Timezone and Clock](#timezone-and-clock)
    - [Locale Configuration](#locale-configuration)
    - [Console Keymap](#console-keymap)
    - [Hostname Configuration](#hostname-configuration)
    - [User Management](#user-management)
    - [Configure Sudo Access](#configure-sudo-access)

11. [Initramfs Configuration](#initramfs-configuration)
    - [Dracut Setup](#dracut-setup)

13. [Kernel Module Configuration](#kernel-module-configuration)
    - [Blacklist Nouveau](#blacklist-nouveau)
    - [Configure NVIDIA KMS](#configure-nvidia-kms)

13. [Kernel Command Line Configuration](#kernel-command-line-configuration)
    - [Retrieve System UUID](#retrieve-system-uuid)
    - [Create Kernel Command Line](#create-kernel-command-line)

14. [Unified Kernel Image Setup](#unified-kernel-image-setup)
    - [Install Systemd-Boot](#install-systemd-boot)
    - [Configure Kernel-Install](#configure-kernel-install)
    - [Configure Systemd-Boot](#configure-systemd-boot)
    - [Generate Initial UKIs](#generate-initial-ukis)

15. [Pacman Hooks: Automated Maintenance Hooks](#pacman-hooks-automated-maintenance-hooks)
    - [Create the shared rebuild script](#create-the-shared-rebuild-script)
    - [UKI Rebuild and Signing Hook](#uki-rebuild-and-signing-hook)
    - [NVIDIA Driver Update Hook](#nvidia-driver-update-hook)
    - [Systemd-Boot Update Hook](#systemd-boot-update-hook)
    - [Secure Boot Signing Hook](#secure-boot-signing-hook)

16. [Understanding Pacman Hooks](#understanding-pacman-hooks)
    - [Hook Anatomy and Syntax](#hook-anatomy-and-syntax)
    - [The Install Command](#the-install-command)
    - [Hook Structure](#hook-structure)
    - [The Exec Command Logic](#the-exec-command-logic)
    - [Why Multiple Hook Files?](#why-multiple-hook-files?)

17. [System Services](#system-services)
    - [Enable Essential Services](#enable-essential-services)
    - [Disable Automounting](#disable-automounting)

18. [Secure Boot Configuration](#secure-boot-configuration)
    - [Generate and Enroll Keys](#generate-and-enroll-keys)

20. [Package Management Optimization](#package-management-optimization)
    - [Reflector Configuration](#reflector-configuration)
    - [Pacman Configuration](#pacman-configuration)

20. [Finalization](#finalization)
    - [Exit Chroot and Unmount](#exit-chroot-and-unmount)
    - [Reboot](#reboot)

20. [Post-Installation Tasks](#post-installation-tasks)
    - [Enable Secure Boot in Firmware](#enable-secure-boot-in-firmware)
    - [Verify System Configuration](#verify-system-configuration)
    - [Mount Additional Volumes](#mount-additional-volumes)

21. [Maintenance and Recovery](#maintenance-and-recovery)
    - [Routine Kernel Updates](#routine-kernel-updates)
    - [Systemd-Boot Maintenance](#systemd-boot-maintenance)
    - [Secure Boot Troubleshooting](#secure-boot-troubleshooting)
    - [LUKS Key Management](#luks-key-management)
    - [Backup Critical Data](#backup-critical-data)
    - [Emergency Recovery](#emergency-recovery)

22. [Troubleshooting](#troubleshooting)
    - [System Won't Boot](#system-won't-boot)
    - [LUKS Unlock Fails](#luks-unlocks-fails)
    - [Graphics Issues](#graphics-issues)
    - [Hibernation Not Working](#hibernation-not-working)

23. [Additional Resources](#additional-resources)
    - [Documentation References](#documentation-references)

## Prerequisites and Assumptions

### System Requirements

- Target drive: `/dev/nvme0n1` (verify with `lsblk -d | grep disk`)
- UEFI firmware (not Legacy/CSM)
- Active internet connection
- Booted from Arch Linux ISO
- Current date: 2025-10-14

> [!warning] 
> This procedure will completely erase all data on the target drive. Verify the correct drive designation before proceeding. Double-check with `lsblk` that `/dev/nvme0n1` is your intended target.

---

## Network and Time Configuration

### Network Setup

Identify your network interface:

```bash
ip link
```

**For Ethernet (static IP):**

```bash
ip addr add <ADDRESS/CIDR> dev <IFACE>
ip route add default via <GATEWAY> dev <IFACE>
```

**For WiFi**:

```bash
iwctl

# Inside iwctl prompt:
device list
station <DEVICE> scan
station <DEVICE> get-networks
station <DEVICE> connect "<SSID>"
# Enter password when prompted
exit
```

> [!tip] 
> Test connectivity with `ping -c 3 archlinux.org` before proceeding.

### Time Synchronization

```bash
timedatectl set-ntp true
timedatectl set-timezone Europe/Paris
timedatectl status
```

> [!important] 
> Accurate time is critical for HTTPS certificate validation during package downloads and for proper filesystem timestamps.

---

## Disk Preparation

### Understanding the Storage Layout

```
/dev/nvme0n1
├─ nvme0n1p1    ESP (1 GiB, FAT32)
├─ nvme0n1p2    LUKS2 → cryptos → LVM (leo-os)
│                └─ root, var, home, swap, data
└─ nvme0n1p3    LUKS2 → cryptvms → LVM (leo-vms)
                 └─ vms
```

### Partition Purposes

**ESP (nvme0n1p1):**
- Stores EFI bootloader and signed boot components
- Must be FAT32 for UEFI firmware compatibility
- Contains: systemd-boot, UKIs, Secure Boot signatures

**System partition (nvme0n1p2):**
- Encrypted container for the entire operating system
- Houses LVM volumes for flexible space management
- Standard volumes:
  - `root`: Core system files (/bin, /usr, /lib, etc.)
  - `var`: Variable data (logs, caches, package databases)
  - `home`: User personal files and configurations
  - `swap`: Hibernation and memory overflow
  - `data`: Additional encrypted storage (optional, see note below)

**VM partition (nvme0n1p3):**
- Separate encrypted container for virtual machine storage
- Isolates VM disk images from system storage
- Allows independent backup/snapshot strategies

> [!important]
> **This is my personal layout.** The `data` and `vms` volumes are optional and specific to my workflow:
> 
> - `data`: Additional encrypted volume for sensitive documents (nested encryption example)
> - `vms`: Dedicated storage for libvirt/QEMU virtual machines
> 
> **Adapt to your needs:**
> - Single-user laptop: Skip `data`, merge everything into `home`
> - No virtualization: Omit `nvme0n1p3` entirely, expand `nvme0n1p2` to 100%
> - Server use: Add separate volumes for `/srv`, `/opt`, or database storage
> - Dual-boot: Reserve unencrypted space or shrink partitions accordingly

### Benefits of This Approach

- **Full system encryption**: Everything except ESP is protected by LUKS2
- **Flexible volume management**: LVM allows resizing without repartitioning
- **Logical separation**: System, user data, and VMs isolated for security and backups
- **Optional nested encryption**: Extra protection layer for highly sensitive data
- **Performance optimization**: VMs on separate physical partition reduce I/O contention

### Secure Drive Erasure

> [!warning] 
> These operations are destructive and irreversible. Confirm the target drive before executing.

**Quick erasure for SSDs (fastest):**

```bash
blkdiscard -f /dev/nvme0n1
```

**Hardware-based NVMe sanitization (most secure):**

First, check capabilities:

```bash
nvme id-ctrl /dev/nvme0n1 | grep -i sanitize
```

Then sanitize:

```bash
# Cryptographic erase (if supported)
nvme format /dev/nvme0n1 --ses=1

# OR block erase
nvme sanitize /dev/nvme0n1 --sanact=1

# OR overwrite pattern
nvme sanitize /dev/nvme0n1 --sanact=2
```

> [!warning] 
> NVMe sanitize operations can take considerable time and may require a power cycle to complete. The drive will be inaccessible during this process.

> [!important] 
> For rotational drives only, the traditional cryptsetup open + wipe pattern applies. SSDs benefit more from hardware-based erasure commands.

---

## Disk Partitioning

### Create GPT Partition Table

```bash
parted -s /dev/nvme0n1 mklabel gpt
```

### Create Partitions

```bash
# EFI System Partition (1 GiB)
parted -s /dev/nvme0n1 mkpart ESP fat32 1MiB 1025MiB
parted -s /dev/nvme0n1 set 1 esp on
parted -s /dev/nvme0n1 set 1 boot on

# System partition (LUKS container)
parted -s /dev/nvme0n1 mkpart primary 1025MiB 70%

# VMs partition (LUKS container)
parted -s /dev/nvme0n1 mkpart primary 70% 100%
```

Verify the layout:

```bash
parted -s /dev/nvme0n1 print
lsblk /dev/nvme0n1
```

> [!note] 
> The ESP is sized at 1 GiB to accommodate multiple UKI images, bootloader updates, and diagnostic tools. The filesystem type flag in parted is metadata only; actual filesystem creation happens later.

---

## Encryption Setup

### LUKS2 Configuration

Create encrypted containers with strong parameters:

```bash
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/nvme0n1p2

cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/nvme0n1p3
```

> [!note]
> Parameters explained:
> - `aes-xts-plain64`: Industry-standard encryption mode for storage
> - `key-size 512`: 256-bit security for AES-XTS (512 bits total for XTS mode)
> - `argon2id`: Modern KDF resistant to GPU/ASIC attacks

### Open Encrypted Containers

```bash
cryptsetup open --persistent /dev/nvme0n1p2 cryptos
cryptsetup open --persistent /dev/nvme0n1p3 cryptvms
```

> [!note] 
> The `--persistent` flag stores the mapping in `/etc/crypttab.initramfs`, ensuring consistent device mapper names across reboots when handled by dracut.

---

## LVM Configuration

### Initialize Physical Volumes

```bash
pvcreate /dev/mapper/cryptos
pvcreate /dev/mapper/cryptvms
```

### Create Volume Groups

```bash
vgcreate leo-os /dev/mapper/cryptos
vgcreate leo-vms /dev/mapper/cryptvms
```

### Create Logical Volumes

**System volumes:**

```bash
lvcreate -L 38G -n swap leo-os
lvcreate -l 25%FREE -n root leo-os
lvcreate -l 20%FREE -n var leo-os
lvcreate -l 15%FREE -n home leo-os
lvcreate -l  5%FREE -n data leo-os
```

**VM storage:**

```bash
lvcreate -l 50%FREE -n vms leo-vms
```

Verify the layout:

```bash
vgdisplay
lvdisplay
lsblk
```

> [!tip] 
> Size allocations are percentages of remaining free space, calculated sequentially. Adjust percentages based on your storage requirements and usage patterns.

### Optional: Nested Encryption for User Data

Add an additional encryption layer for sensitive data:

```bash
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/mapper/leo--os-data

cryptsetup open --persistent /dev/mapper/leo--os-data cryptdata
```

> [!important] 
> This nested approach protects user data even if the system encryption is unlocked. The data volume requires separate authentication and won't auto-unlock with the system.

---

## Filesystem Creation

### Format Partitions

```bash
# ESP
mkfs.fat -F32 /dev/nvme0n1p1

# System volumes
mkfs.ext4 -L rootfs /dev/mapper/leo--os-root
mkfs.ext4 -L varfs  /dev/mapper/leo--os-var
mkfs.ext4 -L homefs /dev/mapper/leo--os-home

# Optional encrypted data
mkfs.ext4 -L datafs /dev/mapper/cryptdata

# VM storage (XFS optimized for large files)
mkfs.xfs -L vmsfs /dev/mapper/leo--vms-vms

# Swap
mkswap -L swapfs /dev/mapper/leo--os-swap
```

> [!note] 
> XFS is recommended for VM storage due to excellent performance with large files, efficient space allocation, and robust handling of concurrent I/O operations.

---

## Mount Filesystems

### Create Mount Hierarchy

```bash
# Root filesystem
mount /dev/mapper/leo--os-root /mnt

# System directories with security options
mount --mkdir -o noatime,lazytime,nodev,nosuid /dev/mapper/leo--os-var  /mnt/var
mount --mkdir -o noatime,lazytime,nodev,nosuid /dev/mapper/leo--os-home /mnt/home

# ESP with restrictive permissions
mount --mkdir -o noatime,nodev,nosuid,noexec,umask=0077 /dev/nvme0n1p1 /mnt/boot

# Activate swap
swapon /dev/mapper/leo--os-swap
```

> [!note] 
> Mount options explained:
> 
> - `noatime,lazytime`: Reduce write operations for better SSD longevity
> - `nodev`: Prevent device file interpretation
> - `nosuid`: Ignore setuid/setgid bits
> - `noexec`: Prevent direct execution (safe for ESP since UEFI handles execution)
> - `umask=0077`: Restrict ESP to root-only access

> [!tip] 
> The data and vms volumes should be mounted after first boot, not during installation. Add them to `/etc/fstab` manually with appropriate options:
> ```
> /dev/mapper/cryptdata    /data  ext4  noatime,nodev,nosuid,noexec  0 2
> /dev/mapper/leo--vms-vms /vms   xfs   noatime,discard=async        0 2
> ```

---

## Base System Installation

### Install Essential Packages

```bash
pacstrap -K /mnt \
  base base-devel linux linux-firmware linux-lts intel-ucode \
  busybox e2fsprogs xfsprogs cryptsetup lvm2 util-linux networkmanager iwd \
  vim nano man-db man-pages texinfo \
  dracut systemd-ukify \
  sbctl \
  nvidia nvidia-lts nvidia-utils \
  power-profiles-daemon
```

> [!note] 
> Package selection rationale:
> 
> - `linux` + `linux-lts`: Primary and fallback kernels
> - `intel-ucode`: CPU microcode updates
> - `dracut`: Modern initramfs with UKI support
> - `sbctl`: Secure Boot key management
> - `nvidia` + `nvidia-lts`: Graphics drivers for both kernel versions
> - `power-profiles-daemon`: Laptop power management

### Generate Filesystem Table

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Review and verify:

```bash
cat /mnt/etc/fstab
```

> [!warning] 
> Inspect `/mnt/etc/fstab` before proceeding. Ensure all partitions have correct UUIDs and mount options. Incorrect fstab entries can prevent system boot.

---

## System Configuration

### Enter Chroot Environment

```bash
arch-chroot /mnt
```

> [!important] 
> All subsequent commands are executed inside the chroot until explicitly noted otherwise.

### Timezone and Clock

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

### Locale Configuration

```bash
# Enable locale
sed -i 's/^#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen

# Set system locale
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

### Console Keymap

```bash
echo 'KEYMAP=uk' > /etc/vconsole.conf
```

> [!tip] 
> Verify available keymaps with `localectl list-keymaps | grep -i uk` if you need a different layout.

### Hostname Configuration

```bash
echo 'leo-os' > /etc/hostname
```

### User Management

**Set root password:**

```bash
passwd
```

**Create user account:**

```bash
useradd -m -G wheel -s /bin/bash leo
passwd leo
```

### Configure Sudo Access

```bash
# Enable wheel group
sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers

# Explicit user permission
echo 'leo ALL=(ALL:ALL) ALL' > /etc/sudoers.d/leo

# Enable sudo logging
echo 'Defaults logfile=/var/log/sudo.log' > /etc/sudoers.d/logging
```

> [!warning] 
> Always use `visudo` for manual sudoers edits to prevent syntax errors. The commands above are safe for automated configuration but verify with `visudo -c` afterward.

---

## Initramfs Configuration

### Dracut Setup

Create `/etc/dracut.conf.d/10-minimal.conf`:

```bash
install -Dm0644 /dev/stdin /etc/dracut.conf.d/00-global.conf <<'EOF'
hostonly="yes"
hostonly_mode="strict"
loglevel=3
EOF

install -Dm0644 /dev/stdin /etc/dracut.conf.d/05-compress.conf <<'EOF'
compress="zstd -T0 -3"
EOF

install -Dm0644 /dev/stdin /etc/dracut.conf.d/10-modules.conf <<'EOF'
add_dracutmodules+=" systemd crypt lvm resume busybox i18n "
EOF

install -Dm0644 /dev/stdin /etc/dracut.conf.d/20-drivers.conf <<'EOF'
add_drivers+=" nvme xhci_pci i915 nvidia nvidia_modeset nvidia_uvm nvidia_drm "
EOF

install -Dm0644 /dev/stdin /etc/dracut.conf.d/90-omit.conf <<'EOF'
omit_dracutmodules+=" network network-manager wicked nfs iscsi url-lib mdraid multipath btrfs zram dmsquash-live livenet plymouth "
EOF
```


> [!note] 
> Configuration explained:
> 
> **Core settings:**
> - `hostonly="yes"`: Include only drivers/modules for current hardware
> - `hostonly_mode="strict"`: Maximum reduction of initramfs size
> - `compress="zstd -T0 -3"`: Multi-threaded compression, balance between speed and size
> - `loglevel=3`: Reduce boot verbosity (errors and warnings only)
>
> **Required modules:**
> - `systemd`: Modern init system for initramfs
> - `crypt`: LUKS encryption support
> - `lvm`: LVM volume activation
> - `resume`: Hibernation/suspend-to-disk support
> - `busybox`: Minimal emergency shell and core utilities
>
> **Essential drivers:**
> - `nvme`: NVMe SSD controller
> - `xhci_pci`: USB 3.0+ host controller
> - `i915`: Intel integrated graphics (for early KMS)
> - `nvidia*`: NVIDIA GPU drivers for early modesetting

> [!note]
> Omitted modules and their purposes:
> 
> **Network-related (not needed for local boot):**
> - `network`: Generic network support
> - `network-manager`: NetworkManager integration
> - `wicked`: SUSE network manager
> - `nfs`: Network File System client
> - `iscsi`: Internet SCSI remote storage
> - `url-lib`: HTTP/FTP fetching capabilities
> 
> **Storage technologies (not in use):**
> - `mdraid`: Software RAID (md)
> - `multipath`: Multi-path device management
> - `btrfs`: Btrfs filesystem (using ext4/xfs)
> - `zram`: Compressed RAM block device
> - `dmsquash-live`: Live system support (squashfs)
> 
> **Boot environment features:**
> - `livenet`: Network-based live systems
> - `plymouth`: Graphical boot splash screen

> [!warning]
> If you later add dual-boot with network boot (PXE), encrypted network storage (iSCSI), or software RAID, you must remove the corresponding entries from `omit_dracutmodules` and regenerate the initramfs with `kernel-install`.

> [!tip] 
> Omitting unused modules reduces:
> - Attack surface (fewer code paths)
> - initramfs size (faster loading)
> - Boot time (fewer modules to process)
> - Memory footprint during early boot.
> 
> If you need network boot or mdraid later, remove those entries from `omit_dracutmodules`. You must also explicitly add it to `add_dracutmodules` to ensure it's included in the initramfs.

---

## Kernel Module Configuration

### Blacklist Nouveau

Create `/etc/modprobe.d/05-blacklist-nouveau.conf`:

```bash
install -Dm0644 /dev/stdin /etc/modprobe.d/05-blacklist-nouveau.conf <<'EOF'
blacklist nouveau
EOF
```

> [!note]
> **Why blacklist Nouveau?**
>You're installing proprietary NVIDIA drivers (nvidia, nvidia-lts, nvidia-utils). Without blacklisting Nouveau (the open-source NVIDIA driver), both drivers attempt to control the GPU simultaneously, causing:
> - Driver conflicts and initialization failures
> - System instability, black screens, or boot failures
> - Early KMS (kernel mode setting) breakage in initramfs
>
> **Skip this step if**:
> - Using only Nouveau (no proprietary drivers installed)
> - System has Intel/AMD graphics only (Nouveau never loads)
> - NVIDIA drivers not included in initramfs (less critical but still recommended for consistency)

### Configure NVIDIA KMS

Create `/etc/modprobe.d/10-nvidia-kms.conf`:

```bash
install -Dm0644 /dev/stdin /etc/modprobe.d/10-nvidia-kms.conf <<'EOF'
options nvidia_drm modeset=1 fbdev=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
EOF
```

> [!note] 
> NVIDIA options explained:
> 
> - `modeset=1`: Enable kernel mode setting for improved graphics
> - `fbdev=1`: Enable framebuffer device support
> - `NVreg_PreserveVideoMemoryAllocations=1`: Required for suspend/resume functionality
> - `NVreg_TemporaryFilePath=/var/tmp`: Alternative temporary file location

---

## Kernel Command Line Configuration

### Retrieve System UUID

```bash
SYSUUID="$(blkid -o value -s UUID /dev/nvme0n1p2)"
echo "System LUKS UUID: $SYSUUID"
```

> [!warning] 
> Verify this UUID corresponds to your system LUKS partition (nvme0n1p2). Incorrect UUID will cause boot failure.

### Create Kernel Command Line

Create `/etc/kernel/cmdline`:

```bash
cat > /etc/kernel/cmdline <<EOF
root=/dev/mapper/leo--os-root rd.luks.name=${SYSUUID}=cryptos rd.luks.options=cryptos=password echo=no,discard=0,timeout=20s,tries=3 rd.lvm.lv=leo-os/root loglevel=3 quiet
EOF
```

Verify the content:

```bash
cat /etc/kernel/cmdline
```

> [!important] 
> This must be a single line without line breaks. The example above should be entered as one continuous line. Parameters:
> 
> - `root=`: Root filesystem location
> - `rd.luks.name=`: LUKS device UUID and mapper name
> - `rd.luks.options=`: LUKS unlock options (no discard for security, 3 unlock attempts)
> - `rd.lvm.lv=`: LVM volumes to activate
> - `loglevel=3`: Reduced kernel verbosity
> - `quiet`: Minimal boot messages

> [!tip] 
> If you need SSD (Continuous) TRIM support within LUKS, change `discard=0` to `discard=1`. Note that this may leak information about filesystem usage patterns to attackers with physical access.

---

## Unified Kernel Image Setup

### Install Systemd-Boot

```bash
bootctl install
```

### Configure Kernel-Install

Create `/etc/kernel/install.conf`:

```bash
cat > /etc/kernel/install.conf <<'EOF'
layout=uki
initrd_generator=dracut
uki_generator=ukify
EOF
```

### Configure Systemd-Boot

Create `/boot/loader/loader.conf`:

```bash
cat > /boot/loader/loader.conf <<'EOF'
default @saved
auto-firmware no
timeout 10
console-mode max
editor no
EOF
```

> [!note] 
> Loader configuration explained:
> 
> - `default @saved`: Remember last boot choice
> - `auto-firmware no`: Don't show firmware settings entry
> - `timeout 3`: 3-second boot menu timeout
> - `console-mode max`: Use maximum console resolution
> - `editor no`: Disable kernel parameter editing (security)

### Generate Initial UKIs

```bash
for dir in /usr/lib/modules/*; do
    [[ -f $dir/vmlinuz ]] || continue
    kernel-install add "${dir##*/}" "$dir/vmlinuz"
done
```

Verify UKI creation:

```bash
ls -lh /boot/EFI/Linux/
```

> [!warning] 
> Do not manually run `dracut --uefi` for the same kernel version. The `kernel-install` command already handles UKI generation through dracut and ukify integration.

---

## Pacman Hooks: Automated Maintenance Hooks

### Create the shared rebuild script:

```bash
sudo install -Dm0755 /dev/stdin /usr/local/libexec/uki-rebuild-sign.sh <<'EOF'
#!/usr/bin/env sh
#!/usr/bin/env sh
set -eu
for dir in /usr/lib/modules/*; do
    [[ -f $dir/vmlinuz ]] || continue
    kernel-install add "${dir##*/}" "$dir/vmlinuz"
done
# sign UKIs and systemd-boot in one shot
find /boot/EFI/Linux   -name '*.efi' -exec sbctl sign -s {} +
find /usr/lib/systemd/boot/efi -name 'systemd-boot*.efi' -exec sbctl sign -s {} +
EOF
```

### UKI Rebuild and Signing Hook

Create `/etc/pacman.d/hooks/90-uki-build-sign.hook`:

```bash
install -Dm0644 /dev/stdin /etc/pacman.d/hooks/90-uki-build-sign.hook <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/modules/*/vmlinuz
Type = Package
Target = nvidia
Target = nvidia-lts

[Action]
Description = Re-building and signing UKIs…
When = PostTransaction
Exec = /usr/local/libexec/uki-rebuild-sign.sh
EOF
```

### Systemd-Boot Update Hook

Create `/etc/pacman.d/hooks/95-systemd-boot.hook`:

```bash
install -Dm0644 /dev/stdin /etc/pacman.d/hooks/95-systemd-boot.hook <<'EOF'
[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Updating systemd-boot…
When = PostTransaction
Exec = /usr/bin/bootctl update
EOF
```

### Secure Boot Signing Hook

Create `/etc/pacman.d/hooks/99-re-sign-after-keys.hook`:

```bash
install -Dm0644 /dev/stdin /etc/pacman.d/hooks/80-secureboot.hook <<'EOF'
[Trigger]
Type = Path
Operation = Install
Operation = Upgrade
Operation = Remove
Target = etc/secureboot/*
Target = usr/share/secureboot/*

[Action]
Description = Re-signing all EFI binaries after key change…
When = PostTransaction
Exec = /usr/local/libexec/uki-rebuild-sign.sh
EOF
```

> [!note] 
> These hooks automate critical maintenance tasks:
> - Rebuild UKIs when kernels are installed/upgraded
> - Sign all EFI binaries for Secure Boot compatibility
> - Update systemd-boot when systemd is upgraded

---

## Understanding Pacman Hooks

### Hook Anatomy and Syntax

Pacman hooks allow automatic execution of commands during package transactions. Let's break down the syntax and logic using the Secure Boot hook as an example.

#### The Install Command

```bash
install -Dm0644 /dev/stdin /etc/pacman.d/hooks/80-secureboot.hook <<'EOF'
```

**Command breakdown:**

- `install`: GNU coreutils command for copying files with specific permissions
- `-D`: Create parent directories if they don't exist
- `-m0644`: Set file permissions to `0644` (owner read/write, group/others read-only)
- `/dev/stdin`: Read content from standard input (the heredoc that follows)
- `/etc/pacman.d/hooks/80-secureboot.hook`: Destination file path

**Why use this method?**

- Creates the file and sets permissions in one atomic operation
- Eliminates need for separate `mkdir -p` and `chmod` commands
- Heredoc (`<<'EOF'`) allows multi-line content without escape issues

> [!note] 
> The single quotes in `<<'EOF'` prevent variable expansion within the heredoc content. This ensures shell variables like `$f` remain literal and are interpreted later during hook execution, not during hook creation.

#### Hook Structure

```bash
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/systemd/boot/efi/systemd-boot*.efi
```

**Trigger section explained:**

- `Operation = Install`: Hook activates when packages are installed
- `Operation = Upgrade`: Hook activates when packages are upgraded
- `Type = Path`: Monitor specific filesystem paths (alternative: `Type = Package`)
- `Target = usr/lib/systemd/boot/efi/systemd-boot*.efi`: Glob pattern matching the systemd-boot binary

> [!note] 
> The `Target` path is relative to the root filesystem (`/`). Pacman automatically prepends `/` when checking, so you write `usr/lib/...` not `/usr/lib/...`.

The asterisk wildcard (`*`) matches any systemd-boot variant (e.g., `systemd-bootx64.efi`, `systemd-bootaa64.efi`).

```bash
[Action]
Description = Signing systemd-boot EFI binary for Secure Boot
When = PostTransaction
Depends = sh
NeedsTargets
Exec = /bin/sh -c 'while read -r f; do sbctl sign -s -o "${f}.signed" "$f"; done'
```

**Action section explained:**

- `Description`: Human-readable text shown during package operations
- `When = PostTransaction`: Execute after the package transaction completes successfully
- `Depends = sh`: Ensure the `sh` shell is available (safety check)
- `NeedsTargets`: Pass triggered file paths to the command via standard input
- `Exec`: Command to execute

#### The Exec Command Logic

```bash
/bin/sh -c 'while read -r f; do sbctl sign -s -o "${f}.signed" "$f"; done'
```

**Step-by-step execution:**

1. **`/bin/sh -c`**: Execute the following command in a shell
2. **`while read -r f; do`**: Read each line from standard input into variable `f`
    - `-r`: Treat backslashes literally (don't interpret escape sequences)
3. **`sbctl sign -s -o " ${f}.signed" "$f"`**: Sign each file
    - `sbctl sign`: sbctl's signing command
    - `-s`: Save signature to database for verification
    - `-o " ${f}.signed"`: Output signed file with `.signed` suffix
    - `"$f"`: Input file (the triggered path)
4. **`done`**: End of while loop

**What `NeedsTargets` does:**

When pacman triggers this hook because `systemd-bootx64.efi` was updated, it pipes the full path to the Exec command's stdin:

```
/usr/lib/systemd/boot/efi/systemd-bootx64.efi
```

The while loop reads this path into `$f` and executes:

```bash
sbctl sign -s -o "/usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed" "/usr/lib/systemd/boot/efi/systemd-bootx64.efi"
```

### Why Multiple Hook Files?

The installation uses four separate hooks with different numbering:

```
80-secureboot.hook          # Signs systemd-boot
90-uki-build-sign.hook      # Rebuilds UKIs on kernel install
91-uki-rebuild-on-nvidia.hook  # Rebuilds UKIs on NVIDIA update
95-systemd-boot.hook        # Updates systemd-boot
```

**Execution order:**

- Hooks execute in alphanumeric order: `80` → `90` → `91` → `95`
- Lower numbers run first
- This ensures systemd-boot is signed (80) before being updated (95)

> [!tip] 
> Use consistent numbering schemes:
> - `00-19`: Pre-transaction preparation
> - `20-49`: During transaction modifications
> - `50-79`: Post-transaction file operations
> - `80-89`: Security operations (signing, verification)
> - `90-99`: Final cleanup and regeneration

---

## System Services

### Enable Essential Services

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
systemctl enable power-profiles-daemon
systemctl enable fstrim.timer
```
> [!note] 
> Services explained:
> - `NetworkManager`: Network connectivity management
> - `systemd-timesyncd`: NTP time synchronization
> - `power-profiles-daemon`: Power management for laptops
> - `fstrim.timer`: Weekly SSD (Periodic) TRIM operations for longevity

### Disable Automounting

```bash
systemctl disable autofs
```

---

## Secure Boot Configuration

### Generate and Enroll Keys

> [!warning] 
> Perform these steps before enabling Secure Boot in firmware. Once Secure Boot is enabled, only signed binaries will execute.

```bash
# Check current status
sbctl status

# Generate custom Secure Boot keys
sbctl create-keys

# Enroll keys (include Microsoft keys for dual-boot compatibility)
sbctl enroll-keys --microsoft

# Sign systemd-boot bootloader
sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi

sbctl sign -s -o /boot/EFI/boot/Bootx64.efi

# Regenerate the initramfs
pacman -S linux linux-lts

# Verify signing status
sbctl status

# Update bootloader
bootctl update
bootctl list
```

> [!important] 
> The `--microsoft` flag preserves Microsoft's keys, allowing Windows installation media and other Microsoft-signed binaries to boot. Omit this if you only boot Linux.

> [!tip] 
> After completing the installation and rebooting, enable Secure Boot in your firmware settings. On the next boot, verify with `sbctl status` that Secure Boot is active and all binaries are properly signed.

---

## Package Management Optimization

### Reflector Configuration

Install and configure reflector for optimal mirror selection:

```bash
pacman -S reflector

install -Dm0644 /dev/stdin /etc/xdg/reflector/reflector.conf <<'EOF'
--country France,Netherlands,Germany
--latest 20
--protocol https
--sort rate
--save /etc/pacman.d/mirrorlist
EOF

systemctl enable --now reflector.timer
```

> [!tip] 
> Adjust the `--country` list to include geographic regions closest to your location for optimal download speeds.

### Pacman Configuration

Edit `/etc/pacman.conf`:

```bash
# Enable colored output
sed -i 's/^#Color/Color/' /etc/pacman.conf

# Enable parallel downloads
sed -i 's/^#ParallelDownloads = 5/ParallelDownloads = 10/' /etc/pacman.conf

# Optional: Enable progress candy
sed -i '/^Color/a ILoveCandy' /etc/pacman.conf
```

Verify signature checking is enabled:

```bash
grep "^SigLevel" /etc/pacman.conf
```

Expected output:

```
SigLevel = Required DatabaseOptional
```

> [!important] 
> The `SigLevel = Required DatabaseOptional` setting ensures all packages are cryptographically verified, preventing tampering. Never disable signature verification.

---

## Finalization

### Exit Chroot and Unmount

```bash
# Exit chroot environment
exit

# Unmount all filesystems
umount -R /mnt
swapoff /dev/mapper/leo--os-swap

# Close encrypted volumes
cryptsetup close cryptdata

# Deactivate LVM
vgchange -an leo-os
vgchange -an leo-vms

cryptsetup close cryptos
cryptsetup close cryptvms

```

### Reboot

```bash
reboot
```

> [!tip] 
> Before rebooting, double-check that you've completed all configuration steps, especially Secure Boot key enrollment and UKI generation.

---

## Post-Installation Tasks

### Enable Secure Boot in Firmware

1. Access UEFI firmware settings during boot
2. Navigate to Secure Boot configuration
3. Enable Secure Boot
4. Save and exit

### Verify System Configuration

After first boot, verify everything is working:

```bash
# Check Secure Boot status
sbctl status

# Verify encryption
lsblk
cryptsetup status cryptos
cryptsetup status cryptvms

# Check LVM volumes
lvdisplay

# Verify mounted filesystems
df -h
mount | grep mapper

# Verify swap
findmnt /tmp
Expected: FSTYPE=tmpfs with nosuid,nodev,relatime.

# Test hibernation
systemctl hibernate
```

### Mount Additional Volumes

If using nested encryption or VM storage:

**Encrypted data volume:**

```bash
cryptsetup open /dev/mapper/leo--os-data cryptdata
mount /dev/mapper/cryptdata /data
```

**VM storage:**

```bash
mount /dev/mapper/leo--vms-vms /vms
```

Add to `/etc/fstab` for automatic mounting:

```
/dev/mapper/cryptdata    /data  ext4  noatime,nodev,nosuid,noexec  0 2
/dev/mapper/leo--vms-vms /vms   xfs   noatime,discard=async        0 2
```

> [!warning] 
> The nested encrypted data volume requires manual unlocking. Consider creating a systemd service or script if automatic unlocking is desired.

---

## Maintenance and Recovery

### Routine Kernel Updates

Updates are handled automatically by pacman hooks, but you can manually verify:

```bash
# Check installed kernels
ls /usr/lib/modules/

# Verify UKIs exist
ls /boot/EFI/Linux/

# Check signing status
sbctl verify

# Manual rebuild if needed
for k in /usr/lib/modules/*/vmlinuz; do
  [ -e "$k" ] || continue
  v="${k%/vmlinuz}"
  v="${v##*/}"
  kernel-install add "$v" "$k"
done

# Sign all UKIs
find /boot/EFI/Linux -type f -name "*.efi" -exec sbctl sign -s {} \;
```

### Systemd-Boot Maintenance

Bootloader updates are automated, but manual updates:

```bash
# Update bootloader
bootctl update

# List boot entries
bootctl list

# Check status
bootctl status
```

### Secure Boot Troubleshooting

If Secure Boot blocks booting:

1. Temporarily disable Secure Boot in firmware
2. Boot the system
3. Check sbctl status:
```bash
sbctl status
sbctl verify
```
4. Re-sign all binaries:
```bash
find /boot/EFI/Linux -type f -name "*.efi" -exec sbctl sign -s {} \;
sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
bootctl update
```
5. Re-enable Secure Boot in firmware

### LUKS Key Management

**Add additional key slot:**

```bash
cryptsetup luksAddKey /dev/nvme0n1p2
```

**Remove key slot:**

```bash
cryptsetup luksRemoveKey /dev/nvme0n1p2
```

**View key slots:**

```bash
cryptsetup luksDump /dev/nvme0n1p2 | grep "Key Slot"
```

### Backup Critical Data

**Export Secure Boot keys:**

```bash
sbctl export-keys /root/sbkeys
```

**Backup LUKS headers:**

```bash
cryptsetup luksHeaderBackup /dev/nvme0n1p2 --header-backup-file /root/cryptos.header
cryptsetup luksHeaderBackup /dev/nvme0n1p3 --header-backup-file /root/cryptvms.header
```

> [!warning] 
> Store these backups on external media in a secure location. LUKS header corruption without backup means permanent data loss.

**Restore LUKS header:**

```bash
cryptsetup luksHeaderRestore /dev/nvme0n1p2 --header-backup-file /root/cryptos.header
```

### Emergency Recovery

If unable to boot:

1. Boot from Arch ISO
2. Unlock encrypted volumes:
```bash
cryptsetup open /dev/nvme0n1p2 cryptos
vgchange -ay leo-os
mount /dev/mapper/leo--os-root /mnt
mount /dev/nvme0n1p1 /mnt/boot
```
3. Chroot and repair:
```bash
arch-chroot /mnt
# Perform repairs
```

---

## Troubleshooting Common Issues

### System Won't Boot

**Check boot entries:**

```bash
bootctl list
```

**Verify Secure Boot signature:**

```bash
sbctl verify
```

**Check UKI exists:**

```bash
ls -l /boot/EFI/Linux/
```

### LUKS Unlock Fails

**Check LUKS header integrity:**

```bash
cryptsetup luksDump /dev/nvme0n1p2
```

**Test password:**

```bash
cryptsetup open --test-passphrase /dev/nvme0n1p2
```

### Graphics Issues

**Check NVIDIA driver loading:**

```bash
lsmod | grep nvidia
dmesg | grep -i nvidia
```

**Verify KMS:**

```bash
cat /sys/module/nvidia_drm/parameters/modeset
```

Expected output: `Y`

### Hibernation Not Working

**Check resume parameter:**

```bash
cat /proc/cmdline | grep resume
```

**Verify swap size:**

```bash
free -h
```

Swap should be at least equal to RAM size for hibernation.

---

## Additional Resources

### Documentation References

- [Arch Wiki: Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki: dm-crypt](https://wiki.archlinux.org/title/Dm-crypt)
- [Arch Wiki: Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [Arch Wiki: Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [dracut Documentation](https://man.archlinux.org/man/dracut.8)
- [systemd-boot Documentation](https://man.archlinux.org/man/systemd-boot.7)
- [zram Documentation](https://wiki.archlinux.org/title/Zram)

## License

Documentation provided as-is for educational purposes.
