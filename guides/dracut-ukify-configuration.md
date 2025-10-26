## Table of Contents

1. [Installation](#installation)
2. [Configuration Overview](#configuration-overview)
3. [Configuration File Setup](#configuration-file-setup)
4. [Hook Management](#hook-management)
5. [Initramfs Generation](#initramfs-generation)

## Installation

### Install dracut-ukify Package

```bash
yay -S dracut-ukify
```

>[!NOTE]
>The dracut-ukify package is the modern approach for generating unified kernel images using systemd-ukify. It automatically creates EFI-executable initramfs images that systemd-boot can detect without additional boot entries.

### Configuration Overview

**Key features**:
- Generates unified kernel images (UKI) as single EFI executables
- Integrates kernel, initramfs, and command line into one file
- Supports Secure Boot signing
- Automatic systemd-boot integration
- No separate boot loader entries needed

**Generated UKI location**:

```bash
/boot/EFI/Linux/linux-<version>-<machine_id>-<build_id>.efi
```

## Configuration File Setup

### Create Main Configuration

**Get system UUID first**:

```bash
blkid -o value -s UUID /dev/nvme1n1p2
```

**Edit configuration file**:

```bash
sudo vim /etc/dracut-ukify.conf
```

```
# Configuration file for dracut-ukify package

# ============================================================================
# Output Configuration
# ============================================================================

# Colorize output (auto, true, or false)
colorize=auto

# ============================================================================
# Boot Entry Configuration
# ============================================================================

# Set default kernel package in systemd-boot
# Uncomment and set to your preferred kernel (linux, linux-lts, linux-g14)
#default_kernel_package='linux-lts'

# ============================================================================
# Kernel Command Line
# ============================================================================

# IMPORTANT: Replace UUID_HERE with actual UUID from blkid command above
# Kernel command line is specified here via --cmdline flag
# Do NOT create /etc/kernel/cmdline as it would conflict

ukify_global_args+=(--cmdline "rd.luks.name=UUID_HERE=cryptos rd.lvm.lv=leo-os/root root=/dev/mapper/leo--os-root rootfstype=ext4 rd.luks.options=discard,timeout=30,tries=3 nvidia_drm.modeset=1 loglevel=3 quiet")

# ============================================================================
# Secure Boot Configuration
# ============================================================================

# Sign UKI images for UEFI Secure Boot
# Requires sbctl keys to exist at these paths
# Verify with: ls -l /var/lib/sbctl/keys/db/
# If keys don't exist, run: sbctl create-keys

ukify_global_args+=(--sign-kernel --secureboot-private-key /var/lib/sbctl/keys/db/db.key --secureboot-certificate /var/lib/sbctl/keys/db/db.pem)

# ============================================================================
# Optional Features
# ============================================================================

# Add splash screen (BMP format only)
#ukify_global_args+=(--splash /usr/share/systemd/bootctl/splash-arch.bmp)

# Use systemd-sbsign (requires systemd>=257)
#ukify_global_args+=(--signtool=systemd-sbsign --no-sign-kernel)

# ============================================================================
# Build Variants
# ============================================================================

# Define image variants
# Key "default" is special - omitted from resulting image name
# Example with fallback:
# ukify_variants=(
#   [default]="--hostonly"
#   [fallback]="--no-hostonly"
# )

ukify_variants=(
  [default]="--hostonly"
)

# ============================================================================
# Install Path Configuration
# ============================================================================

# UKI installation path for each variant
# Available variables:
#   ${name}       - Package name (linux, linux-lts, linux-zen, etc)
#   ${version}    - Package version
#   ${machine_id} - Machine ID from /etc/machine-id
#   ${build_id}   - Build ID from /etc/os-release (ArchLinux: 'rolling')
#   ${id}         - OS ID from /etc/os-release (ArchLinux: 'arch')
#   ${esp}        - ESP partition mount path
#   ${boot}       - XBOOTLDR partition mount path, or ESP if none exists

ukify_install_path=(
  [default]='EFI/Linux/${id}-${name}-${version}.efi'
)

# ============================================================================
# Per-Variant Command Line Override
# ============================================================================

# Override kernel cmdline per variant (optional)
# Example:
#ukify_cmdline=(
#  [default]='root=/dev/mapper/leo--os-root quiet'
#  [fallback]='root=/dev/mapper/leo--os-root'
#)
```

>[!IMPORTANT]
>You **must** replace UUID_HERE with the actual UUID from your encrypted partition. Get it with:
>```bash
>blkid -o value -s UUID /dev/nvme1n1p2
>```

## Hook Management

When using `dracut-ukify`, the traditional `90-dracut-install` hook may still generate non-UKI initramfs images in `/boot/`, such as `initramfs-linux.img`. These files:

- Are not used by systemd-boot when UKI images exist
- Waste space on the ESP partition
- Create confusion about which initramfs is actually being used
- Are redundant since UKI images already contain the initramfs

### Disable Traditional dracut-ukify Hook

Simply run:

```bash
sudo touch /etc/pacman.d/hooks/90-dracut-install.hook
```

### Remove Conflicting Configuration Files

**Clean up old kernel configuration**:

```bash
[[ -f /etc/kernel/cmdline ]] && sudo rm /etc/kernel/cmdline
[[ -f /etc/kernel/install.conf ]] && sudo rm /etc/kernel/install.conf
```

>[!TIP]
>The `--cmdline` flag in `/etc/dracut-ukify.conf` takes precedence over `etc/kernel/cmdline`. Removing old files prevents confusion but isn't strictly necessary.

## Initramfs Generation

### Trigger UKI Generation

**Reinstall kernel packages to trigger hooks**:

```bash
sudo pacman -S linux-g14 linux-lts
```

### Verify UKI Creation

**Check generated UKI files**:

```bash
ls -lh /boot/EFI/Linux/
```

**Expected output**:

```bash
arch-linux-g14-<version>.efi
arch-linux-lts-<version>.efi
```

**Verify boot entries**:

```bash
bootctl list
```

>[!NOTE]
>Each UKI is a complete, self-contained bootable image. No separate initramfs files, no boot loader configuration files, and no command line stored separately. Everything is embedded in one signed EFI executable.

## Resources

### Official Documentation

- [Arch Wiki: Unified Kernel Image](https://wiki.archlinux.org/title/Unified_kernel_image)
- [dracut-ukify AUR Package](https://aur.archlinux.org/packages/dracut-ukify)
- [systemd-ukify Manual](https://man.archlinux.org/man/ukify.1)
- [dracut Manual](https://man.archlinux.org/man/dracut.8)

### Related Guides

- [Arch Wiki: Secure Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot)
- [Arch Wiki: systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)
