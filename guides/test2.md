## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Installation Setup](#pre-installation-setup)
3. [Disk Layout Planning](#disk-layout-planning)
4. [Disk Preparation](#disk-preparation)
5. [Partition Creation](#partition-creation)
6. [Encryption Setup](#encryption-setup)
7. [LVM Configuration](#lvm-configuration)
8. [Filesystem Creation](#filesystem-creation)
9. [Mounting Filesystems](#mounting-filesystems)
10. [Base System Installation](#base-system-installation)
11. [System Configuration](#system-configuration)
12. [Boot Configuration](#boot-configuration)
13. [Secure Boot Setup](#secure-boot-setup)
14. [Package Management](#package-managment)
15. [Pacman Hooks](#pacman-hooks)
16. [Secure Boot Setup](#secure-boot-setup)
17. [System Services](#system-services)
18. [Finalization](#finalization)
19. [Post-Installation Tasks](#post-installation-tasks)
20. [Maintenance](#maintenance)
21. [Resources](#ressources)

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
/dev/nvme0n1 (Windows disk - DO NOT MODIFY)
├─ nvme0n1p1    Windows ESP
├─ nvme0n1p2    Windows Recovery
├─ nvme0n1p3    Windows System (C:)
└─ nvme0n1p4    Windows Data (optional)

/dev/nvme1n1 (Linux disk - our target)
├─ nvme1n1p1    ESP (1 GiB, FAT32)
├─ nvme1n1p2    LUKS2 → cryptos → LVM (leo-os)
│                └─ root (50G), var (30G), home (100G), swap (32G), data (50G)
└─ nvme1n1p3    LUKS2 → cryptvms → LVM (leo-vms) [OPTIONAL]
                 └─ vms (remaining space)
```

>[!NOTE]
>**Partition purposes**:
>- **ESP (nvme1n1p1)**: Linux bootloader and boot components (separate from Windows ESP)
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
>- Skip `nvme1n1p3` if you don't use VMs - expand `nvme1n1p2` to 100%
>- Skip `data` volume if you don't need nested encryption
>- Adjust volume sizes based on your usage patterns

### Why This Layout?

**Security**:
- Everything except ESP is encrypted with LUKS2
- Nested encryption for `data` adds extra protection layer
- Complete physical separation from Windows disk

**Flexibility**:
- LVM allows resizing volumes without repartitioning
- Independent backup strategies for system vs user data
- Easy to add new volumes later

**Performance**:
- VMs on separate partition reduce I/O contention with system
- Dedicated swap improves hibernation reliability
- No disk sharing with Windows (eliminates cross-OS performance impact)

## Disk Preparation

>[!WARNING]
>The following operations are **irreversible**. Triple-check you're targeting `/dev/nvme1n1` (Linux disk), NOT `/dev/nvme0n1` (Windows disk).

### Verify Target Drive

```bash
lsblk -d | grep disk
fdisk -l /dev/nvme0n1  # Windows disk - should show Windows partitions
fdisk -l /dev/nvme1n1  # Linux disk - this is our target
```

Confirm:
- `/dev/nvme0n1` contains Windows partitions (DO NOT TOUCH)
- `/dev/nvme1n1` is your intended Linux target

>[!IMPORTANT]
>If your disks are named differently (e.g., `/dev/sda` and `/dev/nvme0n1`), adjust all subsequent commands accordingly. Use `lsblk -d` to identify the correct device names.

### Secure Erasure

**For the Linux disk only**:

```bash
# Fastest method for SSDs
blkdiscard -f /dev/nvme1n1
```

**Alternative: Hardware sanitization (most secure)**:

```bash
# Check sanitize capabilities
nvme id-ctrl /dev/nvme1n1 | grep -i sanitize

# Cryptographic erase (if supported)
nvme format /dev/nvme1n1 --ses=1

# OR block erase
nvme sanitize /dev/nvme1n1 --sanact=1
```

>[!WARNING]
>Hardware sanitization can take hours and requires a power cycle. The drive is inaccessible during this process.

## Partition Creation

### Create Linux Disk Partitions

```bash
# Create GPT partition table on Linux disk
parted -s /dev/nvme1n1 mklabel gpt

# Create ESP (1GB)
parted -s /dev/nvme1n1 mkpart ESP fat32 1MiB 1025MiB
parted -s /dev/nvme1n1 set 1 esp on
parted -s /dev/nvme1n1 set 1 boot on

# Create Linux system partition (adjust size as needed)
parted -s /dev/nvme1n1 mkpart primary 1025MiB 70%

# Create VM partition (optional, use remaining space)
parted -s /dev/nvme1n1 mkpart primary 70% 100%

# Verify
parted /dev/nvme1n1 print
lsblk /dev/nvme1n1
```

>[!IMPORTANT]
>All partition commands use `/dev/nvme1n1` (Linux disk). Never execute partition commands on `/dev/nvme0n1` (Windows disk).

## Encryption Setup

### Configure LUKS2 for System Partition

```bash
# Encrypt system partition on Linux disk
cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  /dev/nvme1n1p2

# Type YES and enter a strong passphrase
```

>[!IMPORTANT]
>**Choose a strong passphrase**:
>- Minimum 20 characters recommended
>- Mix letters, numbers, symbols
>- Store backup securely offline
>- You'll type this on every boot

**Open encrypted container**:

```bash
cryptsetup open --persistent /dev/nvme0n1p2 cryptos
```

>[!note]
>The `--persistent` flag stores the mapping in `/etc/crypttab.initramfs`, ensuring consistent device mapper names across reboots when handled by dracut.

### Configure LUKS2 for VM Partition (Optional)

```bash
# Encrypt VM partition on Linux disk
cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  /dev/nvme1n1p3

# Open encrypted container
cryptsetup open --persistent /dev/nvme1n1p3 cryptvms
```

>[!TIP]
>You can use the same passphrase for both containers or different ones. Same passphrase is more convenient, different passphrases provide better security isolation.

**Encryption Options Explained**
- `--type luks2`: Modern LUKS version with better security
- `--cipher aes-xts-plain64`: Industry-standard encryption
- `--key-size 512`: Maximum security for AES-XTS
- `--hash sha512`: Strong hashing algorithm
- `--pbkdf argon2id`: Memory-hard key derivation (resists brute-force)

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

**System volumes**:

```bash
# Swap
lvcreate -L 8G -n swap leo-os

# Root filesystem
lvcreate -l 25%FREE -n root leo-os

# Var for logs and caches
lvcreate -l 20%FREE -n var leo-os

# Home for user home
lvcreate -l 15%FREE -n home leo-os

# Encrypted data volume
lvcreate -l 5%FREE -n data leo-os
```

**VM volume (optional)**:

```bash
# Use all remaining space
lvcreate -l 100%FREE -n vms leo-vms
```

>[!TIP]
>Size allocations are percentages of remaining free space, calculated sequentially. Adjust percentages based on your storage requirements and usage patterns.

### Verify LVM structure:

```bash
pvdisplay
vgdisplay
lvdisplay
```

### Optional: Nested Encryption for Data Volume

```bash
# Encrypt the data LV
cryptsetup luksFormat --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --hash sha512 \
  --pbkdf argon2id \
  /dev/mapper/leo--os-data

# Open encrypted data volume
cryptsetup open --persistent /dev/mapper/leo--os-data cryptdata
```

>[!NOTE]
>**Why nested encryption?**
>- Provides additional security layer for highly sensitive data
>- Different passphrase from system encryption
>- Can remain locked when not actively needed
>- Adds minimal performance overhead on modern CPUs with AES-NI

## Filesystem Creation

### Format Partitions

```bash
# ESP on Linux disk
mkfs.fat -F32 /dev/nvme1n1p1

# System volumes
mkfs.ext4 -L rootfs /dev/mapper/leo--os-root
mkfs.ext4 -L varfs  /dev/mapper/leo--os-var
mkfs.ext4 -L homefs /dev/mapper/leo--os-home

# Nested encrypted data
mkfs.ext4 -L datafs /dev/mapper/cryptdata

# VM storage with XFS
mkfs.xfs -L vmsfs /dev/mapper/leo--vms-vms

# Swap
mkswap -L swapfs /dev/mapper/leo--os-swap
```

>[!NOTE]
>**Filesystem choices**:
>- **ext4**: Reliable, mature, excellent for system and user data
>- **XFS**: Superior for large files and concurrent I/O (ideal for VMs)
>- **FAT32**: Required for ESP (UEFI firmware compatibility)

## Mounting Filesystems

### Mount in Correct Order

```bash
# Mount root first
mount /dev/mapper/leo--os-root /mnt

# Create and mount system directories
mount --mkdir /dev/mapper/leo--os-var /mnt/var -o noatime,lazytime,nodev,nosuid
mount --mkdir /dev/mapper/leo--os-home /mnt/home -o noatime,lazytime,nodev,nosuid

# Mount ESP from Linux disk
mount --mkdir /dev/nvme1n1p1 /mnt/boot -o noatime,nodev,nosuid,noexec,umask=0077

# Activate swap
swapon /dev/mapper/leo--os-swap
```

>[!TIP]
>**Mount options explained**:
>- `noatime`: Reduces writes (better SSD lifespan)
>- `nodev`: Prevents device file interpretation
>- `nosuid`: Ignores setuid/setgid bits (security)
>- `noexec`: Prevents direct execution (safe for ESP)
>- `umask=0077`: Restricts ESP to root-only access

>[!IMPORTANT]
>Do **NOT mount data and vms volumes now**. They should be manually mounted after first boot with appropriate security options.

### Verify Mount Structure

```bash
lsblk -f
mount | grep /mnt
```

Expected output:

```bash
/dev/mapper/leo--os-root on /mnt type ext4
/dev/mapper/leo--os-var on /mnt/var type ext4
/dev/mapper/leo--os-home on /mnt/home type ext4
/dev/nvme0n1p1 on /mnt/boot type vfat
```

## Base System Installation

### Install Essential Packages

```bash
pacstrap -K /mnt \
  base base-devel linux linux-firmware linux-lts intel-ucode \
  busybox e2fsprogs xfsprogs cryptsetup lvm2 util-linux \
  networkmanager iwd dhcpcd \
  vim nano man-db man-pages texinfo \
  dracut systemd-ukify \
  sbctl \
  nvidia nvidia-lts nvidia-utils \
  power-profiles-daemon
```

>[!NOTE]
>**Package selection rationale**:
>- **Kernels**: linux (latest) + linux-lts (stable fallback)
>- **Microcode**: intel-ucode for CPU security updates
>- **Encryption**: cryptsetup + lvm2 for LUKS and LVM
>- **Network**: networkmanager for easy connectivity
>- **Boot**: dracut + systemd-ukify for UKI creation
>- **Secure Boot**: sbctl for key management
>- **Graphics**: nvidia drivers for RTX 4070
>- **Power**: power-profiles-daemon for laptop efficiency

### Generate Filesystem Table

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## System Configuration

### Enter Chroot Environment

```bash
arch-chroot /mnt
```

>[!NOTE]
>You're now inside your new system. All subsequent commands run here until you `exit`.

### Timezone and Clock

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

### Locale Configuration

```bash
# Enable English locale

sed -i 's/^#en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen

# Set system locale
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

### Console Keymap

```bash
echo 'KEYMAP=uk' > /etc/vconsole.conf
```

>[!TIP]
>Find your keymap: `localectl list-keymaps | grep -i <your-region>`

### Hostname

```bash
echo 'leo-os' > /etc/hostname
```

### User Management

**Set root password**:

```bash
passwd
```

**Create your user**:

```bash
useradd -m -G wheel -s /bin/bash leo
passwd leo
```

### Configure Sudo

```bash
# Enable wheel group
sed -i 's/^# %wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/' /etc/sudoers

# Add explicit user permission
install -Dm0440 /dev/stdin /etc/sudoers.d/leo <<'EOF'
leo ALL=(ALL:ALL) ALL

# Enable sudo logging
install -Dm0440 /dev/stdin /etc/sudoers.d/logging <<'EOF'
Defaults logfile=/var/log/sudo.log
EOF
```

>[!WARNING]
>Always verify sudoers syntax: `visudo -c`. Syntax errors can lock you out of sudo.

## Boot Configuration

### Initramfs with Dracut

**Create dracut configurations**:

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

>[!NOTE]
>**Dracut configuration explained**:
>- `hostonly="yes"`: Include only current hardware drivers (smaller initramfs)
>- `systemd`: Modern init for initramfs
>- `crypt`: LUKS decryption support
>- `lvm`: LVM activation
>- `nvidia*`: Early GPU initialization


### Kernel Modules Configuration

**Blacklist Nouveau**:

```bash
install -Dm0644 /dev/stdin /etc/modprobe.d/05-blacklist-nouveau.conf <<'EOF'
blacklist nouveau
EOF
```

>[!NOTE]
>**Why blacklist Nouveau?** The proprietary NVIDIA driver conflicts with the open-source Nouveau driver. Both cannot control the GPU simultaneously.

**Configure NVIDIA KMS**:

```bash
install -Dm0644 /dev/stdin /etc/modprobe.d/10-nvidia-kms.conf <<'EOF'
options nvidia_drm modeset=1 fbdev=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1 NVreg_TemporaryFilePath=/var/tmp
EOF
```

>[!NOTE]
>**NVIDIA options**:
>- `modeset=1`: Enable kernel mode setting (better graphics)
>- `fbdev=1`: Framebuffer device support
>- `NVreg_PreserveVideoMemoryAllocations=1`: Required for suspend/resume
>- `NVreg_TemporaryFilePath=/var/tmp`: Alternative temp location

### Kernel Command Line

**Get system LUKS UUID**:

```bash
SYSUUID=$(blkid -o value -s UUID /dev/nvme1n1p2)
echo "System LUKS UUID: $SYSUUID"
```

>[!WARNING]
>Verify this UUID matches your system partition (**nvme1n1p2** on Linux disk). Wrong UUID = boot failure.

**Create kernel command line**:

```bash
install -Dm0644 /dev/stdin /etc/kernel/cmdline <<EOF
root=/dev/mapper/leo--os-root rd.luks.name=${SYSUUID}=cryptos rd.luks.options=cryptos=password echo=no,discard=0,timeout=20s,tries=3 rd.lvm.lv=leo-os/root loglevel=3 quiet
EOF
```

**Verify (must be single line)**:

```bash
cat /etc/kernel/cmdline
```

>[!IMPORTANT]
>**Kernel parameters explained**:
>- `root=`: Root filesystem location
>- `rd.luks.name=`: LUKS UUID and mapper name
>- `rd.luks.options=`: Unlock timeout and retry attempts
>- `rd.lvm.lv=`: LVM volume to activate
>- `nvidia_drm.modeset=1`: Enable NVIDIA KMS

### Unified Kernel Images (UKI)

**Install systemd-boot**:

```bash
bootctl install
```

**Configure kernel-install**:

```bash
install -Dm0644 /dev/stdin /etc/kernel/install.conf <<'EOF'
layout=uki
initrd_generator=dracut
uki_generator=ukify
EOF
```

**Configure systemd-boot**:

```bash
install -Dm0644 /dev/stdin /boot/loader/loader.conf <<'EOF'
default @saved
timeout 3
console-mode max
editor no
EOF
```

>[!NOTE]
>- `default @saved`: Boot last-used entry
>- `timeout 3`: 3 second menu delay
>- `editor no`: Disable kernel parameter editing (security)


**Generate UKIs for both kernels**:

```bash
for dir in /usr/lib/modules/*; do
    [[ -f $dir/vmlinuz ]] || continue
    kernel-install add "${dir##*/}" "$dir/vmlinuz"
done
```

**Verify UKI creation**:

```bash
ls -lh /boot/EFI/Linux/
bootctl list
```

You should see:
- `arch-linux.efi`
- `arch-linux-lts.efi`

>[!warning]
>Do not manually run `dracut --uefi` for the same kernel version. `The kernel-install` command already handles UKI generation through dracut and ukify integration.

## Pacman Hooks

Automated maintenance hooks ensure your system stays bootable after updates.

### Shared Rebuild Script

```bash
install -Dm0755 /dev/stdin /usr/local/bin/uki-rebuild-sign.sh <<'EOF'
#!/usr/bin/env sh
set -eu
# rebuild all UKIs
for dir in /usr/lib/modules/*; do
    [[ -f $dir/vmlinuz ]] || continue
    kernel-install add "${dir##*/}" "$dir/vmlinuz"
done
# sign everything
find /boot/EFI/Linux   -name '*.efi' -exec sbctl sign -s {} +
find /usr/lib/systemd/boot/efi -name 'systemd-boot*.efi' -exec sbctl sign -s {} +

# ---------- friendly names ----------
cd /boot/EFI/Linux
for k in /usr/lib/modules/*/vmlinuz; do
    [[ -f $k ]] || continue
    ver=${k%/vmlinuz}
    ver=${ver##*/}
    case $ver in
        *-lts) title="Arch Linux (LTS) (${ver})" ;;
        *)     title="Arch Linux (Main) (${ver})" ;;
    esac
    safe=${title// /-}.efi
    ln -f "${ver}.efi" "$safe" 2>/dev/null || cp "${ver}.efi" "$safe"
    printf '%s' "$title" > "$safe.splash"
done
# keep only the pretty-named UKIs
rm -f [0-9a-f]*-*.efi        # deletes raw *.efi files
EOF
```

### UKI Rebuild Hook (Kernel Updates)

```bash
install -Dm0644 /dev/stdin /etc/pacman.d/hooks/90-uki-build-sign.hook <<'EOF'
[Trigger]
Operation = Install
Operation = Upgrade
Type = Path
Target = usr/lib/modules/*/vmlinuz

[Trigger]
Operation = Install
Operation = Upgrade
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

>[!NOTE]
>**Hook execution order**:
>- 90: Rebuild UKIs when kernel updates
>- 91: Rebuild UKIs when NVIDIA updates
>- 95: Update systemd-boot itself
>- 99: Sign everything for Secure Boot
>
>Lower numbers run first. This ensures components are built before signing.

### Package Management

## Pacman Configuration

```bash
# Enable colors and parallel downloads
sed -i 's/^#Color/Color/' /etc/pacman.conf
sed -i 's/^#ParallelDownloads = 5/ParallelDownloads = 10/' /etc/pacman.conf
sed -i '/^Color/a ILoveCandy' /etc/pacman.conf

# Verify signature checking (should already be enabled)
grep "^SigLevel" /etc/pacman.conf
```

Expected output: `SigLevel = Required DatabaseOptional`

>[!IMPORTANT]
>Never disable signature verification. It prevents package tampering and ensures authenticity.

## Secure Boot Setup

### Generate Secure Boot Keys
```bash
# Check current status
sbctl status

# Create custom keys
sbctl create-keys

# Enroll keys (include Microsoft for Windows compatibility)
sbctl enroll-keys --microsoft
```

>[!NOTE]
>`--microsoft` preserves Microsoft keys, allowing Windows and Windows Boot Manager to boot. Required for dual-boot systems.

### Sign Boot Components

**Sign systemd-boot**:

```bash
sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi   # fallback file
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
```

**Sign both kernel UKIs**:

```bash
# Regenerate the initramfs
pacman -S linux linux-lts
```

**Verify signatures**:

```bash
sbctl verify
```

All files should show "Signed."

### Windows Boot Manager Setup

Since Windows is on a separate disk with its own ESP, we need to access it:

```bash
# Create temporary mount point
mkdir -p /mnt/win-esp

# Mount Windows ESP (verify partition number first with lsblk)
mount /dev/nvme0n1p1 /mnt/win-esp

# Check Windows bootloader exists
ls -l /mnt/win-esp/EFI/Microsoft/Boot/

# Copy Windows bootloader to Linux ESP
cp -a /mnt/win-esp/EFI/Microsoft /boot/EFI/

# Sign Windows bootloader for Secure Boot
sbctl sign -s /boot/EFI/Microsoft/Boot/bootmgfw.efi

# Clean up
umount /mnt/win-esp
rmdir /mnt/win-esp
```

>[!NOTE]
>**Why copy Windows bootloader**?
>- Your firmware will default to the disk you select in boot menu
>- Having both bootloaders on Linux ESP allows systemd-boot to chain-load Windows
>- This gives you a unified boot menu instead of using firmware boot selection

>[!TIP]
>After copying, you can boot Windows either through:
>1. systemd-boot menu (if it auto-detected the Windows entry)
>2. Firmware boot menu (F8/F12/Del during boot)
>3. Manual EFI boot entry (see troubleshooting section)

## System Services

### Enable Essential Services

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
systemctl enable power-profiles-daemon
systemctl enable fstrim.timer
```

>[!NOTE]
>**Services explained**:
>`NetworkManager`: Network connectivity
>`systemd-timesyncd`: Time synchronization
>`power-profiles-daemon`: Laptop power management
>`fstrim.timer`: Weekly SSD TRIM for longevity

## Finalization

### Exit Chroot

```bash
exit
```

### Unmount Filesystems

```bash
umount -R /mnt
swapoff /dev/mapper/leo--os-swap
```

### Close Encrypted Volumes

```bash
# Close nested encryption
cryptsetup close cryptdata

# Deactivate LVM
vgchange -an leo-os
vgchange -an leo-vms

# Close LUKS containers
cryptsetup close cryptos
cryptsetup close cryptvms
```

### Reboot

```bash
reboot
```

>[!IMPORTANT]
>Remove the installation media. The system should boot to the LUKS password prompt.

## Post-Installation Tasks

### Mount Additional Volumes

If using nested encryption or VM storage:

**Encrypted data volume**:

```bash
cryptsetup open /dev/mapper/leo--os-data cryptdata
mount --mkdir /dev/mapper/cryptdata /data -o noatime,nodev,nosuid,noexec
```

**VM storage**:

```bash
cryptsetup open /dev/nvme0n1p3 cryptvms
mount --mkdir /dev/mapper/leo--vms-vms /vms -o noatime,discard=async
```

Add to `/etc/fstab` for automatic mounting:

```bash
/dev/mapper/cryptdata    /data  ext4  noatime,nodev,nosuid,noexec  0 2
/dev/mapper/leo--vms-vms /vms   xfs   noatime,discard=async        0 2
```

>[!warning]
>The nested encrypted data volume requires manual unlocking. Consider creating a systemd service or script if automatic unlocking is desired.

## Maintenance

### LUKS Key Management

**For Linux disk partitions**:

**Add recovery key to system partition**:

```bash
sudo cryptsetup luksAddKey /dev/nvme1n1p2
```

**Add recovery key to VM partition**:

```bash
sudo cryptsetup luksAddKey /dev/nvme1n1p3
```

**List key slots**:

```bash
sudo cryptsetup luksDump /dev/nvme1n1p2 | grep "Key Slot"
```

**Remove key from slot**:

```bash
sudo cryptsetup luksKillSlot /dev/nvme1n1p2 1
```

**Change passphrase**:

```bash
sudo cryptsetup luksChangeKey /dev/nvme1n1p2
```

### Backup Critical Data

**LUKS headers from Linux disk**:

```bash
sudo cryptsetup luksHeaderBackup /dev/nvme1n1p2 --header-backup-file ~/cryptos.header
sudo cryptsetup luksHeaderBackup /dev/nvme1n1p3 --header-backup-file ~/cryptvms.header
```

>[!warning]
>Store these backups on external media in a secure location. LUKS header corruption without backup means permanent data loss.

**Restore LUKS header**:

```bash
cryptsetup luksHeaderRestore /dev/nvme1n1p2 --header-backup-file /root/cryptos.header
cryptsetup luksHeaderRestore /dev/nvme1n1p3 --header-backup-file /root/cryptvms.header
```

## Resources

### Official Documentation

- [Arch Wiki: Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki: dm-crypt](https://wiki.archlinux.org/title/Dm-crypt)
- [Arch Wiki: Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [Arch Wiki: Unified Extensible Firmware Interface/Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Arch Wiki: NVIDIA](https://wiki.archlinux.org/title/NVIDIA)

### Tools Documentation

- [dracut Manual](https://man.archlinux.org/man/dracut.8)
- [systemd-boot Manual](https://man.archlinux.org/man/systemd-boot.7)
- [cryptsetup Manual](https://man.archlinux.org/man/cryptsetup.8)
- [sbctl GitHub](https://github.com/Foxboron/sbctl)

### Related Guides

- [Arch Wiki: Dual Boot with Windows](https://wiki.archlinux.org/title/Dual_boot_with_Windows)
- [Arch Wiki: Power Management](https://wiki.archlinux.org/title/Power_management)
- [Arch Wiki: Laptop](https://wiki.archlinux.org/title/Laptop)


License
Documentation provided as-is for educational purposes.
