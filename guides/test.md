Table of Contents
Part I: Planning and Preparation

    Document Overview
    Prerequisites and System Requirements
    Understanding the Storage Layout
    Security Considerations

Part II: Installation Process
Phase 1: Environment Preparation

    Boot Environment Verification
    Network Configuration
    Time Synchronization

Phase 2: Storage Configuration

    Secure Drive Erasure
    Disk Partitioning
    Encryption Setup
    LVM Configuration
    Filesystem Creation
    Mount Filesystems

Phase 3: System Installation

    Base System Installation
    Generate Filesystem Table
    Enter Chroot Environment

Phase 4: System Configuration

    Regional Settings
    Network and User Management
    Initramfs Configuration
    Kernel Module Configuration
    Kernel Command Line Setup

Phase 5: Boot Configuration

    Unified Kernel Image Setup
    Systemd-Boot Installation
    Automated Maintenance Hooks
    Understanding Pacman Hooks

Phase 6: Security and Finalization

    System Services Configuration
    Secure Boot Setup
    Package Management Optimization
    Final Verification
    Exit and Reboot

Part III: Post-Installation

    First Boot Procedures
    Enable Secure Boot in Firmware
    System Verification
    Additional Configuration

Part IV: Operations and Maintenance

    Routine Maintenance
    Update Procedures
    Backup Strategies
    Troubleshooting Guide
    Emergency Recovery

Part V: Reference

    Command Quick Reference
    Configuration Templates
    Additional Resources

Part I: Planning and Preparation
Document Overview

This guide provides a complete walkthrough for installing Arch Linux with enterprise-grade security features. The installation implements full-disk encryption using LUKS2, Logical Volume Management for flexible storage, Unified Kernel Images for improved boot security, and Secure Boot for firmware-level protection.

Target Audience: Users with intermediate Linux knowledge seeking a secure, production-ready Arch Linux installation.

Estimated Time: 2-4 hours for first-time installation.

Skill Level: Intermediate to Advanced.
Prerequisites and System Requirements
Hardware Requirements

    UEFI firmware (Legacy/CSM mode not supported)
    NVMe SSD (guide uses /dev/nvme0n1 as example)
    Minimum 8GB RAM (16GB+ recommended for VMs)
    Minimum 100GB storage (200GB+ recommended)
    Active internet connection
    NVIDIA GPU (optional, guide includes driver setup)
    Intel CPU (guide uses intel-ucode, substitute amd-ucode for AMD)

Software Requirements

    Arch Linux installation ISO (latest version)
    USB drive for installation media (2GB minimum)

Knowledge Prerequisites

    Basic understanding of Linux command line
    Familiarity with disk partitioning concepts
    Understanding of encryption fundamentals
    Ability to access UEFI firmware settings

    [!WARNING]
    Data Loss Warning

    This procedure will completely erase all data on the target drive. This action is irreversible.

    Before proceeding:

        Verify target drive with lsblk -d | grep disk
        Backup all important data
        Disconnect other storage devices to prevent accidental erasure
        Double-check all commands before executing

    [!NOTE]
    Pre-Flight Checklist

    Complete these verifications before starting:

    System Checks:

        System boots in UEFI mode: [ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
        Target drive identified: lsblk -d | grep disk
        Sufficient time allocated (2-4 hours)

    Network Checks:

        Internet connection active: ping -c 3 archlinux.org
        DNS resolution working: nslookup archlinux.org

    Preparation Checks:

        LUKS passphrase prepared (20+ characters recommended)
        Root password decided
        Username decided
        All important data backed up

    Documentation:

        This guide accessible during installation
        Alternative device available for reference

    If any check fails, resolve the issue before proceeding.

Understanding the Storage Layout

This guide implements a three-partition layout optimized for security, flexibility, and performance.
Partition Structure

/dev/nvme0n1
├─ nvme0n1p1    ESP (1 GiB, FAT32)
├─ nvme0n1p2    LUKS2 → cryptos → LVM (leo-os)
│                └─ root, var, home, swap, data
└─ nvme0n1p3    LUKS2 → cryptvms → LVM (leo-vms)
                 └─ vms

Partition Purposes

ESP - EFI System Partition (nvme0n1p1)

Purpose: Stores UEFI bootloader and signed boot components

Technical Details:

    Size: 1 GiB (generous allocation for multiple kernels and fallback images)
    Filesystem: FAT32 (required by UEFI specification)
    Mount Point: /boot
    Contents: systemd-boot bootloader, UKI files, Secure Boot signatures

Why This Size: Standard recommendations suggest 512MB, but 1GB provides:

    Space for multiple kernel versions
    Room for kernel backups
    Adequate space for large initramfs images
    Buffer for future updates

System Partition (nvme0n1p2)

Purpose: Encrypted container for entire operating system

Technical Details:

    Encryption: LUKS2 with AES-XTS-256
    Volume Management: LVM (volume group: leo-os)
    Contains: Operating system, user data, swap

Logical Volumes:

    root (50GB): Core system files
        Contains: /, /usr, /lib, /bin, /sbin, /etc
        Why 50GB: Accommodates base system, development tools, and growth

    var (30GB): Variable system data
        Contains: /var (logs, caches, package databases)
        Why 30GB: Prevents log files from filling root, isolates package cache

    home (100GB): User personal data
        Contains: /home (user files, configurations, documents)
        Why 100GB: Adjust based on your needs, main data storage area

    swap (16GB): Memory overflow and hibernation
        Why equal to RAM: Required for hibernation support
        Adjust to match your system RAM

    data (remaining): Additional encrypted storage (optional)
        Purpose: Example of nested encryption for sensitive documents
        Can be omitted or resized based on needs

VM Partition (nvme0n1p3)

Purpose: Separate encrypted container for virtual machine storage

Technical Details:

    Encryption: LUKS2 with AES-XTS-256
    Volume Management: LVM (volume group: leo-vms)
    Contains: Virtual machine disk images

Why Separate:

    Isolates VM storage from system storage
    Allows independent backup strategies
    Reduces I/O contention on system partition
    Simplifies VM migration
    Can be omitted if not using virtualization

    [!NOTE]
    Personal Layout Disclosure

    This storage layout reflects my personal workflow and security requirements. The data and vms volumes are optional components specific to my needs.

    What's Optional:

        data volume: Additional encrypted storage for sensitive documents (demonstrates nested encryption)
        nvme0n1p3 + vms volume: Dedicated VM storage (only needed if using virtualization)

    Customization Examples:

    Single-user laptop without VMs:

/dev/nvme0n1
├─ nvme0n1p1    ESP (1 GiB)
└─ nvme0n1p2    LUKS2 → cryptos → LVM
                 ├─ root (50GB)
                 ├─ var (30GB)
                 ├─ home (remaining)
                 └─ swap (RAM size)

Dual-boot configuration:

/dev/nvme0n1
├─ nvme0n1p1    ESP (1 GiB, shared)
├─ nvme0n1p2    LUKS2 → Arch Linux (150GB)
├─ nvme0n1p3    Windows NTFS (150GB)
└─ nvme0n1p4    Shared Data NTFS (remaining)

Server deployment:

/dev/nvme0n1
├─ nvme0n1p1    ESP (1 GiB)
└─ nvme0n1p2    LUKS2 → LVM
                 ├─ root (30GB)
                 ├─ var (50GB, larger for logs)
                 ├─ srv (100GB, web content)
                 ├─ opt (30GB, applications)
                 └─ swap (RAM size)

Desktop gaming system:

    /dev/nvme0n1
    ├─ nvme0n1p1    ESP (1 GiB)
    └─ nvme0n1p2    LUKS2 → LVM
                     ├─ root (50GB)
                     ├─ var (30GB)
                     ├─ home (300GB, large for games)
                     └─ swap (RAM size)

Benefits of This Architecture

Security:

    Full-disk encryption: Everything except ESP is LUKS2-encrypted
    Passphrase required before system boots
    Protects against physical theft
    Optional nested encryption for highly sensitive data

Flexibility:

    LVM allows resizing volumes without repartitioning
    Easy to add new volumes for new purposes
    Snapshots possible (with additional setup)
    Volumes can be moved between systems

Performance:

    Separate var prevents log files from impacting system performance
    Dedicated swap partition optimized for swap operations
    VM storage isolation reduces I/O contention
    SSD TRIM support maintains drive performance

Maintainability:

    Logical separation simplifies backup strategies
    Individual volumes can be backed up independently
    System recovery easier with separated user data
    Clear organization of data types

Security Considerations
Encryption Strength

This guide implements military-grade encryption:

Algorithm: AES-XTS-256

    AES: Advanced Encryption Standard (NIST FIPS 197)
    XTS: XEX-based tweaked-codebook mode with ciphertext stealing
    256-bit: Key size provides 2^256 possible keys

Key Derivation: Argon2id

    Winner of Password Hashing Competition (2015)
    Memory-hard algorithm resists GPU/ASIC attacks
    Hybrid approach combines Argon2i and Argon2d benefits
    Default: 2-second calibrated iteration count

Attack Resistance:

    Brute force: Computationally infeasible with strong passphrase
    Cold boot: Data encrypted at rest
    Evil maid: Secure Boot prevents bootloader tampering
    Physical access: LUKS2 protects data even if drive removed

    [!TIP]
    Passphrase Security Best Practices

    Your LUKS passphrase is the only thing protecting your data. Make it strong.

    Minimum Requirements:

        Length: 20+ characters
        Complexity: Mix of uppercase, lowercase, numbers, symbols
        Uniqueness: Not used elsewhere
        Memorability: Must be remembered (cannot be recovered if lost)

    Recommended Approach - Diceware:
    Generate a passphrase using random words:

correct-horse-battery-staple-mountain-river-7

    6-7 random words from Diceware list
    Add numbers/symbols
    Easy to remember, hard to crack

What to Avoid:

    Dictionary words alone
    Personal information (names, dates)
    Common phrases or quotes
    Keyboard patterns (qwerty, asdf)
    Short passphrases (under 15 characters)

Passphrase Strength Examples:

    Weak: password123 (seconds to crack)
    Medium: MyP@ssw0rd2024 (hours to crack)
    Strong: correct-horse-battery-staple (centuries to crack)
    Very Strong: 7-random-diceware-words-with-numbers-42 (beyond current computing)

Testing Passphrase Strength:
Use pwqcheck or online tools (never enter actual passphrase online):

    echo "your-test-passphrase" | pwqcheck

    [!WARNING]
    Critical Security Warnings

    Passphrase Loss = Data Loss:

        No backdoor exists in LUKS2 encryption
        No "forgot password" recovery mechanism
        No way to recover data without passphrase
        Not even the NSA can help you

    Mitigation:

        Write passphrase down and store securely (safe deposit box)
        Use a password manager with secure backup
        Consider storing encrypted backup of LUKS header offline

    LUKS Header Vulnerability:

        Header corruption = data loss even with correct passphrase
        Can occur from: drive failure, bit rot, accidental overwrite

    Mitigation:

        Backup LUKS headers immediately after creation
        Store header backups on multiple external devices
        Test header backups periodically

    Physical Security:

        Encryption protects powered-off systems only
        Running system is vulnerable if attacker gains access
        Hibernation writes unencrypted memory to disk

    Mitigation:

        Use strong user passwords
        Enable automatic screen lock
        Shut down (don't suspend) when leaving system unattended
        Consider encrypted swap for hibernation

Secure Boot Implementation

Secure Boot prevents unauthorized code execution during boot:

What It Protects Against:

    Bootkit malware
    Evil maid attacks
    Unauthorized bootloader modifications
    Rootkit installation

How It Works:

    Firmware verifies bootloader signature
    Bootloader verifies kernel signature
    Kernel verifies module signatures
    Chain of trust from firmware to kernel

This Guide's Implementation:

    Custom Secure Boot keys (not manufacturer keys)
    Self-signed UKI (Unified Kernel Image) files
    Automatic signing via pacman hooks
    Microsoft keys enrolled for dual-boot compatibility

    [!NOTE]
    Secure Boot Key Management

    This guide uses custom Secure Boot keys instead of manufacturer keys:

    Advantages:

        Complete control over what boots on your system
        Protection against attacks using stolen manufacturer keys
        Ability to sign custom kernels and modules

    Trade-offs:

        Requires manual key enrollment
        Microsoft-signed media (Windows install) won't boot unless Microsoft keys enrolled
        Recovery requires disabling Secure Boot temporarily

    Key Types:

        PK (Platform Key): Top-level key, controls key enrollment
        KEK (Key Exchange Key): Signs database updates
        db (Signature Database): Allowed signatures
        dbx (Forbidden Signature Database): Revoked signatures

    Microsoft Keys:
    This guide includes --microsoft flag during enrollment:

        Allows booting Windows installation media
        Permits Microsoft-signed hardware drivers
        Can be omitted for Linux-only systems
        Doesn't compromise security (only adds trust)

Part II: Installation Process
Phase 1: Environment Preparation
Boot Environment Verification

After booting from Arch Linux installation media, verify you're in UEFI mode:

# Verify UEFI boot mode
[ -d /sys/firmware/efi ] && echo "UEFI mode confirmed" || echo "ERROR: Not in UEFI mode"

Expected output: UEFI mode confirmed

    [!WARNING]
    UEFI Mode Required

    If the output shows "ERROR: Not in UEFI mode", stop here.

    This guide requires UEFI mode. Legacy/BIOS mode is not supported.

    To enable UEFI mode:

        Reboot into firmware settings
        Disable Legacy/CSM boot mode
        Enable UEFI boot mode
        Set UEFI as first boot priority
        Save and reboot from USB again

Load Keyboard Layout (Optional)

If you use a non-US keyboard layout:

# List available layouts
localectl list-keymaps | grep -i <your-language>

# Load your layout (example: French)
loadkeys fr

# Verify by typing special characters

Network Configuration
Ethernet Connection

For wired connections, the network should work automatically. Verify:

# Check network interfaces
ip link

# Test connectivity
ping -c 3 archlinux.org

If you see successful ping responses, skip to Time Synchronization.
Ethernet with Static IP

If DHCP isn't available:

# Identify your interface
ip link

# Example: interface is enp0s31f6
# Set static IP (adjust values for your network)
ip addr add 192.168.1.100/24 dev enp0s31f6
ip route add default via 192.168.1.1 dev enp0s31f6

# Set DNS server
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# Test connectivity
ping -c 3 archlinux.org

    [!NOTE]
    Network Configuration Variables

    Replace these values with your network details:

        192.168.1.100/24: Your desired IP address and subnet mask
        192.168.1.1: Your gateway (usually your router's IP)
        enp0s31f6: Your network interface name (from ip link output)
        1.1.1.1: DNS server (alternatives: 8.8.8.8, 9.9.9.9, your ISP's DNS)

Wireless Connection

For wireless networks, use iwctl:

# Enter iwctl interactive mode
iwctl

# Inside iwctl prompt:
[iwd]# device list
# Note your wireless device name (usually wlan0)

[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
# Note your network SSID

[iwd]# station wlan0 connect "Your-Network-SSID"
# Enter password when prompted

[iwd]# exit

# Verify connection
ping -c 3 archlinux.org

    [!TIP]
    Wireless Troubleshooting

    Device not found:

# Check if wireless device is blocked
rfkill list

# Unblock all wireless devices
rfkill unblock all

# Check if driver loaded
lsmod | grep iwl

Can't connect to network:

    Verify correct SSID and password
    Check signal strength: iwctl station wlan0 show
    Try forgetting and reconnecting:

    iwctl
    [iwd]# known-networks list
    [iwd]# known-networks "Your-SSID" forget
    [iwd]# station wlan0 connect "Your-SSID"

Connection drops:

    Disable power management:

        iw dev wlan0 set power_save off

Time Synchronization

Accurate time is critical for HTTPS certificate validation and proper filesystem timestamps:

# Enable NTP synchronization
timedatectl set-ntp true

# Set your timezone (example: Europe/Paris)
timedatectl set-timezone Europe/Paris

# Verify time settings
timedatectl status

Expected output should show:

    System clock synchronized: yes
    NTP service: active
    Correct timezone and time

    [!NOTE]
    Finding Your Timezone

    List available timezones:

    timedatectl list-timezones | grep -i <your-city>

    Common timezones:

        US East Coast: America/New_York
        US West Coast: America/Los_Angeles
        UK: Europe/London
        Central Europe: Europe/Paris, Europe/Berlin
        Japan: Asia/Tokyo
        Australia: Australia/Sydney

    [!TIP]
    Verify Network Time Sync

    Ensure time is actually synchronizing:

    # Check NTP sync status
    timedatectl timesync-status

    # If not syncing, restart service
    systemctl restart systemd-timesyncd

    # Wait a few seconds and check again
    timedatectl timesync-status

Phase 2: Storage Configuration
Secure Drive Erasure

Before creating partitions, securely erase the drive to eliminate any previous data.

    [!WARNING]
    Final Warning Before Erasure

    The following commands will permanently destroy all data on /dev/nvme0n1.

    Before proceeding:

        Verify target drive:

        lsblk -d | grep disk

        Confirm it shows your intended target
        Disconnect any other storage devices
        Triple-check the drive path in commands below

    Point of no return: Once executed, data cannot be recovered.

Method 1: Quick Erasure (Fastest - Recommended for SSDs)

# Quick secure erase using TRIM
blkdiscard -f /dev/nvme0n1

This completes in seconds and is sufficient for most users.

Method 2: Hardware-Based Sanitization (Most Secure)

NVMe drives support hardware-level sanitization that's more thorough than software methods.

First, check if your drive supports sanitization:

# Check sanitization capabilities
nvme id-ctrl /dev/nvme0n1 | grep -i sanitize

If supported, choose a sanitization method:

# Option A: Cryptographic erase (fastest hardware method)
# Deletes encryption key, making data unrecoverable
nvme format /dev/nvme0n1 --ses=1

# Option B: Block erase (most thorough)
# Physically erases all blocks
nvme sanitize /dev/nvme0n1 --sanact=1

# Option C: Overwrite pattern (most time-consuming)
# Writes pattern to every sector
nvme sanitize /dev/nvme0n1 --sanact=2

Check sanitization progress:

# Monitor sanitization status
nvme sanitize-log /dev/nvme0n1

    [!WARNING]
    NVMe Sanitization Warnings

    Time Requirements:

        Cryptographic erase: 1-2 seconds
        Block erase: 30 minutes to 2+ hours
        Overwrite: Several hours depending on drive size

    System Behavior:

        Drive is completely inaccessible during sanitization
        System may appear frozen (this is normal)
        Do not interrupt power during operation
        Some drives require power cycle after completion

    If Sanitization Hangs:

        Wait at least 1 hour before assuming failure
        Check status: nvme sanitize-log /dev/nvme0n1
        If stuck, power cycle the system
        After reboot, verify completion
        If failed, use blkdiscard instead

    [!TIP]
    Which Erasure Method to Use

    For most users: Use blkdiscard (Method 1)

        Fast (seconds)
        Sufficient for personal use
        Secure against casual recovery

    For high-security needs: Use cryptographic erase (Method 2, Option A)

        Extremely fast (1-2 seconds)
        Meets enterprise security standards
        Effective against sophisticated recovery

    For paranoid security: Use block erase (Method 2, Option B)

        Time-consuming
        Meets military standards
        Overkill for most scenarios

    Don't bother with overwrite (Method 2, Option C) unless required by policy:

        Extremely slow
        No practical security advantage over block erase
        Wears out SSD unnecessarily

Disk Partitioning

Create the partition table and partitions for our layout.

    [!INFO]
    Partition Scheme Overview

    We're creating three partitions:

        nvme0n1p1 (1GB): EFI System Partition
            Type: EF00 (EFI System)
            Format: FAT32
            Purpose: Bootloader and UKI storage

        nvme0n1p2 (200GB): System partition
            Type: 8309 (Linux LUKS)
            Encryption: LUKS2
            Contents: Operating system and user data

        nvme0n1p3 (remaining): VM partition
            Type: 8309 (Linux LUKS)
            Encryption: LUKS2
            Contents: Virtual machine storage

    Adjust sizes based on your drive capacity and needs.

# Destroy existing partition table
sgdisk --zap-all /dev/nvme0n1

# Clear any remnants
sgdisk --clear /dev/nvme0n1

# Create partition 1: ESP (1GB)
sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI System" /dev/nvme0n1

# Create partition 2: System (200GB) - adjust size as needed
sgdisk -n 2:0:+200G -t 2:8309 -c 2:"Linux LUKS" /dev/nvme0n1

# Create partition 3: VMs (remaining space)
sgdisk -n 3:0:0 -t 3:8309 -c 3:"VM Storage" /dev/nvme0n1

# Verify partition layout
sgdisk -p /dev/nvme0n1

    [!TIP]
    Verify Partition Creation

    Check that partitions were created correctly:

lsblk -o NAME,SIZE,TYPE,FSTYPE /dev/nvme0n1

Expected output:

    NAME        SIZE TYPE FSTYPE
    nvme0n1     512G disk
    ├─nvme0n1p1   1G part
    ├─nvme0n1p2 200G part
    └─nvme0n1p3 311G part

    If incorrect:

        Run sgdisk --zap-all /dev/nvme0n1 to start over
        Adjust size values in the commands above
        Re-run partition creation commands

    [!NOTE]
    Customizing Partition Sizes

    To modify sizes, change the -n parameters:

    Syntax: sgdisk -n PARTITION:START:END

        0 for START: Begin immediately after previous partition
        +SIZE for END: Size in GB (e.g., +100G)
        0 for END: Use all remaining space

    Examples:

    Smaller system partition (100GB):

sgdisk -n 2:0:+100G -t 2:8309 -c 2:"Linux LUKS" /dev/nvme0n1

No VM partition (system uses all space):

sgdisk -n 1:0:+1G -t 1:ef00 -c 1:"EFI System" /dev/nvme0n1
sgdisk -n 2:0:0 -t 2:8309 -c 2:"Linux LUKS" /dev/nvme0n1

Dual-boot (leave 150GB unallocated):

    sgdisk -n 2:0:+200G -t 2:8309 -c 2:"Linux LUKS" /dev/nvme0n1
    # Don't create partition 3, leaving space for Windows

Encryption Setup

Configure LUKS2 encryption on the data partitions with strong security parameters.

    [!NOTE]
    LUKS2 Encryption Parameters

    We're using these encryption settings:

    Cipher: aes-xts-plain64

        AES: Advanced Encryption Standard (military-grade)
        XTS: Mode specifically designed for storage encryption
        plain64: Initialization vector calculation method
        Industry standard for full-disk encryption

    Key Size: 512 bits

        XTS mode uses two keys (256-bit each)
        Provides 256-bit security level
        Optimal balance of security and performance

    PBKDF: argon2id

        Modern password-based key derivation function
        Winner of 2015 Password Hashing Competition
        Memory-hard algorithm resists GPU/ASIC cracking
        Hybrid design combines security of argon2i and argon2d
        Automatically calibrated to 2-second unlock time

    Security Level: These parameters provide security equivalent to:

        AES-256 encryption strength
        Resistance against quantum computing attacks (current generation)
        Protection against brute-force attempts for centuries with strong passphrase

Create encrypted containers on both partitions:

# Encrypt system partition (nvme0n1p2)
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/nvme0n1p2

# You'll be prompted: "Are you sure? (Type 'yes' in capital letters):"
# Type: YES
# Enter your LUKS passphrase twice

# Encrypt VM partition (nvme0n1p3)
cryptsetup luksFormat \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --key-size 512 \
  --pbkdf argon2id \
  /dev/nvme0n1p3

# Type: YES
# Enter your LUKS passphrase twice (can be same or different)

    [!TIP]
    Same or Different Passphrases?

    Same passphrase for both partitions:

        Pros: Only one passphrase to remember, quicker unlock
        Cons: Single point of failure
        Recommended for: Most users, simpler setup

    Different passphrases:

        Pros: Compartmentalized security, can grant VM access without system access
        Cons: Two passphrases to manage, slower unlock
        Recommended for: High-security environments, multi-user systems

    For this guide, we'll assume same passphrase for both partitions.

    [!WARNING]
    Encryption Parameter Warnings

    Higher security = slower unlock:

    Default unlock time is calibrated to 2 seconds. To increase security at cost of unlock time:

    cryptsetup luksFormat \
      --type luks2 \
      --cipher aes-xts-plain64 \
      --key-size 512 \
      --pbkdf argon2id \
      --pbkdf-force-iterations 10 \
      --pbkdf-memory 1048576 \
      /dev/nvme0n1p2

    This increases unlock time to ~10 seconds but dramatically increases brute-force resistance.

    Not recommended unless you have specific high-security requirements:

        Government/military systems
        Whistleblower protection
        Storing classified information

    For 99% of users, default parameters are more than sufficient.

Open the encrypted containers:

# Open system partition
cryptsetup open /dev/nvme0n1p2 cryptos

# Open VM partition
cryptsetup open /dev/nvme0n1p3 cryptvms

# Verify they're opened
ls /dev/mapper/

Expected output should show: control, cryptos, cryptvms

    [!TIP]
    Verify Encryption

    Confirm LUKS containers are properly configured:

    # Check LUKS header for system partition
    cryptsetup luksDump /dev/nvme0n1p2

    # Verify key slot 0 is used
    # Confirm cipher shows: aes-xts-plain64
    # Confirm PBKDF shows: argon2id

    What to look for:

        Version: 2
        Cipher: aes-xts-plain64
        Key size: 512 bits
        PBKDF: argon2id
        Keyslots: Key slot 0 should show as ENABLED

Backup LUKS headers immediately:

This is critical. Header corruption without backup means permanent data loss.

# Create backup directory
mkdir -p /root/luks-backups

# Backup system partition header
cryptsetup luksHeaderBackup /dev/nvme0n1p2 \
  --header-backup-file /root/luks-backups/cryptos-header.img

# Backup VM partition header
cryptsetup luksHeaderBackup /dev/nvme0n1p3 \
  --header-backup-file /root/luks-backups/cryptvms-header.img

# Verify backups exist
ls -lh /root/luks-backups/

    [!WARNING]
    LUKS Header Backup Critical Instructions

    These backups are as sensitive as your passphrase. Anyone with the header backup and your passphrase can decrypt your data.

    Immediate actions required:

        Copy header backups to USB drive:

    # Insert USB drive and mount it
    mkdir /mnt/usb
    mount /dev/sdX1 /mnt/usb  # Replace sdX1 with your USB device
    cp /root/luks-backups/* /mnt/usb/
    umount /mnt/usb

    Store USB drive in secure location:
        Safe deposit box
        Home safe
        Locked drawer
        NOT in the same location as your laptop

    Consider second backup copy on different media

Header restoration (only if header corrupted):

# Boot from Arch ISO
cryptsetup luksHeaderRestore /dev/nvme0n1p2 \
  --header-backup-file /path/to/cryptos-header.img

Test backup (optional but recommended):

    # Close container
    cryptsetup close cryptos

    # Restore from backup (this is safe, overwrites with identical header)
    cryptsetup luksHeaderRestore /dev/nvme0n1p2 \
      --header-backup-file /root/luks-backups/cryptos-header.img

    # Re-open container
    cryptsetup open /dev/nvme0n1p2 cryptos

LVM Configuration

Set up Logical Volume Management for flexible storage allocation.

Initialize Physical Volumes:

# Initialize PV on system partition
pvcreate /dev/mapper/cryptos

# Initialize PV on VM partition
pvcreate /dev/mapper/cryptvms

# Verify PVs
pvs

Expected output:

PV                 VG   Fmt  Attr PSize   PFree
/dev/mapper/cryptos           lvm2 ---  200.00g 200.00g
/dev/mapper/cryptvms          lvm2 ---  311.00g 311.00g

Create Volume Groups:

# Create VG for system
vgcreate leo-os /dev/mapper/cryptos

# Create VG for VMs
vgcreate leo-vms /dev/mapper/cryptvms

# Verify VGs
vgs

Expected output:

VG      #PV #LV #SN Attr   VSize   VFree
leo-os    1   0   0 wz--n- 200.00g 200.00g
leo-vms   1   0   0 wz--n- 311.00g 311.00g

    [!NOTE]
    Volume Group Naming

    Volume group names leo-os and leo-vms are personal choices. You can use any name:

        vg-system and vg-vms
        Your hostname: myhost-os
        Descriptive names: main-system

    Change commands accordingly:

    vgcreate myhost-os /dev/mapper/cryptos

Create Logical Volumes:

    [!INFO]
    Logical Volume Sizing Strategy

    These sizes are examples for a 200GB system partition. Adjust based on your needs:

    root (50GB):

        Purpose: Core system files (/, /usr, /lib, /bin, /etc)
        Rationale: Base system ~10GB, development tools ~15GB, growth buffer ~25GB
        Adjust: Minimal system 30GB, heavy development 70GB

    var (30GB):

        Purpose: Variable data (/var: logs, caches, package databases)
        Rationale: Package cache ~10GB, logs ~5GB, databases ~5GB, buffer ~10GB
        Adjust: Server with extensive logging 50-100GB

    home (100GB):

        Purpose: User personal data and configurations
        Rationale: Documents, downloads, personal files
        Adjust: Gaming system 300GB+, minimal workstation 50GB

    swap (RAM size):

        Purpose: Hibernation support and memory overflow
        Rationale: Must equal RAM for hibernation
        Adjust: Match your system RAM exactly for hibernation

    data (remaining):

        Purpose: Optional additional encrypted storage
        Rationale: Example of nested encryption, sensitive documents
        Adjust: Omit if not needed, or size for specific use

    vms (all):

        Purpose: Virtual machine disk images
        Rationale: Isolated VM storage
        Adjust: Size based on number and size of VMs

# Get your system's RAM size
free -h | grep Mem | awk '{print $2}'
# Note this value for swap size

# Create logical volumes for system
lvcreate -L 50G -n root leo-os
lvcreate -L 30G -n var leo-os
lvcreate -L 100G -n home leo-os
lvcreate -L 16G -n swap leo-os     # Replace 16G with your RAM size
lvcreate -l 100%FREE -n data leo-os  # Uses remaining space

# Create logical volume for VMs
lvcreate -l 100%FREE -n vms leo-vms

# Verify logical volumes
lvs -o lv_name,vg_name,lv_size,lv_path

Expected output:

LV   VG      LSize   Path
data leo-os    4.00g /dev/leo-os/data
home leo-os  100.00g /dev/leo-os/home
root leo-os   50.00g /dev/leo-os/root
swap leo-os   16.00g /dev/leo-os/swap
var  leo-os   30.00g /dev/leo-os/var
vms  leo-vms 311.00g /dev/leo-vms/vms

    [!TIP]
    Verify Volume Group Free Space

    Before creating volumes, check available space:

vgdisplay leo-os | grep Free

Make sure total of all LV sizes doesn't exceed VG free space.

If you over-allocated:

    # Remove all LVs and start over
    lvremove /dev/leo-os/data
    lvremove /dev/leo-os/home
    lvremove /dev/leo-os/var
    lvremove /dev/leo-os/swap
    lvremove /dev/leo-os/root

    # Recreate with adjusted sizes

    [!NOTE]
    Optional: Nested Encryption for Data Volume

    For extra security on sensitive data, you can add another layer of encryption to the data volume:

    # Format data volume with LUKS2
    cryptsetup luksFormat \
      --type luks2 \
      --cipher aes-xts-plain64 \
      --key-size 512 \
      --pbkdf argon2id \
      /dev/mapper/leo--os-data

    # Open nested encryption
    cryptsetup open /dev/mapper/leo--os-data cryptdata

    # Format with filesystem
    mkfs.ext4 -L data /dev/mapper/cryptdata

    Benefits:

        Two passphrases required (system + data)
        Can share system access without data access
        Extra protection for highly sensitive files

    Drawbacks:

        Additional passphrase to manage
        Slight performance overhead
        More complex unlock procedure

    When to use:

        Storing classified/confidential documents
        Multi-user shared system
        Whistleblower/activist protection

    For most users, the base LUKS2 encryption on the partition is sufficient. Skip this nested encryption unless you have specific high-security requirements.

Filesystem Creation

Format all partitions and volumes with appropriate filesystems.

# Format ESP with FAT32
mkfs.fat -F 32 -n ESP /dev/nvme0n1p1

# Format root volume
mkfs.ext4 -L root /dev/mapper/leo--os-root

# Format var volume
mkfs.ext4 -L var /dev/mapper/leo--os-var

# Format home volume
mkfs.ext4 -L home /dev/mapper/leo--os-home

# Format data volume (skip if using nested encryption)
mkfs.ext4 -L data /dev/mapper/leo--os-data

# Format VM volume
mkfs.ext4 -L vms /dev/mapper/leo--vms-vms

# Initialize swap
mkswap -L swap /dev/mapper/leo--os-swap

    [!NOTE]
    Filesystem Choice: ext4

    This guide uses ext4 for all Linux filesystems:

    Why ext4:

        Mature and stable (production-ready since 2008)
        Excellent performance
        Reliable journaling
        Good TRIM support for SSDs
        Universal compatibility

    Alternative: Btrfs
    Replace mkfs.ext4 with mkfs.btrfs if you want:

        Built-in snapshots
        Subvolumes
        Compression
        Self-healing features

mkfs.btrfs -L root /dev/mapper/leo--os-root

Alternative: XFS
Replace mkfs.ext4 with mkfs.xfs for:

    Better performance with large files
    Excellent for databases
    Superior parallel I/O

    mkfs.xfs -L root /dev/mapper/leo--os-root

    For most users, ext4 is the best choice.

    [!TIP]
    Verify Filesystem Creation

    Check that all filesystems were created successfully:

    lsblk -f /dev/nvme0n1

    Expected output should show:

        nvme0n1p1: vfat (ESP label)
        leo-os-root: ext4 (root label)
        leo-os-var: ext4 (var label)
        leo-os-home: ext4 (home label)
        leo-os-data: ext4 (data label)
        leo-os-swap: swap (swap label)
        leo-vms-vms: ext4 (vms label)

    If any filesystem missing or wrong type:
    Re-run the appropriate mkfs command for that volume.

Mount Filesystems

Mount all filesystems in the correct order for installation.

    [!INFO]
    Mount Order Matters

    Always mount parent directories before child directories:

        Root (/) first
        Boot, var, home after root
        Any nested directories last

    Mounting in wrong order will cause errors or hidden mounts.

# Mount root partition first
mount /dev/mapper/leo--os-root /mnt

# Create mount point directories
mkdir -p /mnt/{boot,var,home,data}

# Mount boot partition (ESP)
mount /dev/nvme0n1p1 /mnt/boot

# Mount var partition
mount /dev/mapper/leo--os-var /mnt/var

# Mount home partition
mount /dev/mapper/leo--os-home /mnt/home

# Mount data partition
mount /dev/mapper/leo--os-data /mnt/data

# Enable swap
swapon /dev/mapper/leo--os-swap

# Verify all mounts
mount | grep /mnt
lsblk -o NAME,MOUNTPOINT /dev/nvme0n1

Expected output should show all filesystems properly mounted under /mnt.

    [!TIP]
    Verify Mount Structure

    Confirm mount hierarchy is correct:

findmnt /mnt

Should show tree structure:

TARGET SOURCE                     FSTYPE
/mnt   /dev/mapper/leo--os-root   ext4
├─/mnt/boot /dev/nvme0n1p1        vfat
├─/mnt/var  /dev/mapper/leo--os-var  ext4
├─/mnt/home /dev/mapper/leo--os-home ext4
└─/mnt/data /dev/mapper/leo--os-data ext4

If mount missing or wrong location:

    # Unmount incorrectly mounted filesystem
    umount /path/to/mount

    # Remount correctly
    mount /dev/mapper/leo--os-xxx /mnt/xxx

    [!WARNING]
    Do Not Proceed If Mounts Are Incorrect

    If any mount is missing or in the wrong location:

        Do NOT proceed to next phase
        Unmount all: umount -R /mnt
        Turn off swap: swapoff /dev/mapper/leo--os-swap
        Review and correct mount commands
        Re-mount everything correctly

    Incorrect mounts will result in failed installation or boot issues.

Phase 3: System Installation
Base System Installation

Install the base Arch Linux system and essential packages.

    [!INFO]
    Package Selection Explanation

    Base System:

        base: Core Arch Linux packages
        linux: Latest mainline kernel
        linux-firmware: Device firmware
        linux-lts: Long-term support kernel (fallback)

    CPU Microcode:

        intel-ucode: Intel CPU updates (use amd-ucode for AMD)

    Encryption & LVM:

        lvm2: Logical volume management
        cryptsetup: LUKS encryption tools

    Network:

        networkmanager: Network connection management

    Editors & Documentation:

        vim: Advanced text editor
        nano: Simple text editor
        man-db man-pages texinfo: System documentation

    Boot Infrastructure:

        dracut: Modern initramfs generator
        systemd-ukify: UKI creation tool
        dracut-ukify: Dracut integration for UKI

    Secure Boot:

        sbctl: Secure Boot key management

    Graphics (optional, remove if not using NVIDIA):

        nvidia nvidia-lts: NVIDIA drivers for both kernels
        nvidia-utils: NVIDIA utilities
        nvidia-prime: Laptop GPU switching (optional)

    Power Management (optional, remove if desktop):

        power-profiles-daemon: Laptop power profiles

# Install base system and packages
pacstrap -K /mnt \
  base linux linux-firmware linux-lts \
  intel-ucode \
  lvm2 cryptsetup \
  networkmanager \
  vim nano man-db man-pages texinfo \
  dracut systemd-ukify dracut-ukify \
  sbctl \
  nvidia nvidia-lts nvidia-utils nvidia-prime \
  power-profiles-daemon

This will take 5-15 minutes depending on internet speed.

    [!TIP]
    Package Installation Customization

    For AMD CPU:
    Replace intel-ucode with amd-ucode:

pacstrap -K /mnt \
  base linux linux-firmware linux-lts \
  amd-ucode \
  ...

For AMD/Intel integrated graphics:
Remove NVIDIA packages:

pacstrap -K /mnt \
  base linux linux-firmware linux-lts \
  intel-ucode \
  lvm2 cryptsetup \
  networkmanager \
  vim nano man-db man-pages texinfo \
  dracut systemd-ukify dracut-ukify \
  sbctl \
  power-profiles-daemon

For desktop (not laptop):
Remove power management:

# Remove this line from pacstrap:
# power-profiles-daemon

Additional useful packages (optional):

    pacstrap /mnt \
      git wget curl \           # Development tools
      htop btop \               # System monitors
      tmux \                    # Terminal multiplexer
      zsh \                     # Alternative shell
      sudo \                    # If not included in base
      openssh                   # Remote access

    [!WARNING]
    Package Installation Errors

    Error: "failed to retrieve file":

        Check internet connection: ping archlinux.org
        Update package database: pacman -Sy
        Try different mirror: edit /etc/pacman.d/mirrorlist

    Error: "signature is unknown trust":

    # Refresh keys
    pacman-key --refresh-keys

    # Re-run pacstrap

    Error: "insufficient space":

        Check /mnt has space: df -h /mnt
        Root partition might be too small
        Remove some packages and install later

    Installation interrupted:

        Safe to re-run pacstrap command
        Will continue where it left off

Generate Filesystem Table

Create /etc/fstab with correct mount options.

# Generate fstab with UUIDs
genfstab -U /mnt >> /mnt/etc/fstab

# Review generated fstab
cat /mnt/etc/fstab

    [!TIP]
    Verify Fstab Entries

    Check that /mnt/etc/fstab contains entries for all your filesystems:

    Expected entries:

        /dev/mapper/leo--os-root → / (ext4)
        /dev/nvme0n1p1 → /boot (vfat)
        /dev/mapper/leo--os-var → /var (ext4)
        /dev/mapper/leo--os-home → /home (ext4)
        /dev/mapper/leo--os-data → /data (ext4)
        /dev/mapper/leo--os-swap → none (swap)

    Each entry should have:

        UUID (not device path)
        Mount point
        Filesystem type
        Options (defaults or specific)
        Dump and pass numbers

    If entries missing or incorrect:

    # Edit manually
    vim /mnt/etc/fstab

    # Or regenerate (will append, remove duplicates manually)
    genfstab -U /mnt >> /mnt/etc/fstab

    [!WARNING]
    Critical: Verify Before Proceeding

    Incorrect fstab will prevent system from booting.

    Must verify:

        All mount points present
        All using UUIDs (not /dev/sdX paths)
        Correct filesystem types
        No duplicate entries

    Do not proceed if fstab looks incorrect. Fix it now.

Enter Chroot Environment

Change root into the new system to configure it.

    [!INFO]
    What is Chroot?

    Chroot (change root) makes the new system (currently at /mnt) appear as the root filesystem. This allows us to configure the new system as if we had booted into it.

    After entering chroot:

        Commands run in the new system context
        / refers to what was /mnt
        Can install packages, edit configs, etc.
        Exit with exit command

# Enter chroot environment
arch-chroot /mnt

You're now inside the new system. Your prompt might change.

    [!TIP]
    Verify Chroot Success

    After entering chroot, verify you're in the correct environment:

    # Should show /
    pwd

    # Should show chroot environment
    echo $PS1

    # Check critical mount points exist
    ls /boot /var /home

    If any command fails, you may not be properly chrooted. Exit and try again.

Phase 4: System Configuration
Regional Settings

Configure timezone, locale, and keyboard layout.

Set Timezone:

# Set timezone (example: Europe/Paris)
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime

# Generate /etc/adjtime
hwclock --systohc

# Verify
timedatectl status

    [!NOTE]
    Finding Your Timezone

    List available timezones:

    ls /usr/share/zoneinfo/
    ls /usr/share/zoneinfo/Europe/
    ls /usr/share/zoneinfo/America/

    Common timezones:

        US Eastern: America/New_York
        US Pacific: America/Los_Angeles
        US Central: America/Chicago
        UK: Europe/London
        Central Europe: Europe/Paris, Europe/Berlin, Europe/Amsterdam
        Japan: Asia/Tokyo
        Australia: Australia/Sydney
        India: Asia/Kolkata

Configure Locale:

# Edit locale.gen
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen

# Add additional locales if needed (example: French)
echo "fr_FR.UTF-8 UTF-8" >> /etc/locale.gen

# Generate locales
locale-gen

# Set system locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf

    [!NOTE]
    Locale Configuration Explained

    LANG: Default language for all categories
    LC_MESSAGES: System messages language
    LC_TIME: Date and time format
    LC_NUMERIC: Number format
    LC_MONETARY: Currency format

    Most users only need to set LANG. The system will use it for all categories unless overridden.

    Multiple Languages:
    To use different locales for different purposes:

    # In /etc/locale.conf
    LANG=en_US.UTF-8
    LC_TIME=fr_FR.UTF-8
    LC_MONETARY=fr_FR.UTF-8

Set Console Keymap (Optional):

Only needed if using non-US keyboard layout:

# Set keymap (example: French)
echo "KEYMAP=fr" > /etc/vconsole.conf

Common keymaps: us, uk, fr, de, es, it
Network and User Management

Configure hostname, users, and sudo access.

Set Hostname:

# Choose a hostname for your system
echo "your-hostname" > /etc/hostname

# Configure hosts file
cat > /etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   your-hostname.localdomain your-hostname
EOF

Replace your-hostname with your chosen hostname (single word, lowercase, no spaces).

Set Root Password:

# Set root password
passwd

Enter a strong password for the root account. You'll need this for system recovery.

    [!WARNING]
    Root Password Security

    Choose a different password from your LUKS passphrase:

        Passphrase compromise doesn't compromise system
        System compromise doesn't help with encrypted drive

    Root password requirements:

        Minimum 16 characters
        Mix of character types
        Not related to LUKS passphrase
        Must be memorable (needed for recovery)

Create Regular User:

# Create user (replace 'yourusername' with desired username)
useradd -m -G wheel -s /bin/bash yourusername

# Set user password
passwd yourusername

    [!NOTE]
    User Creation Options Explained

    -m: Create home directory (/home/yourusername)
    -G wheel: Add user to wheel group (for sudo access)
    -s /bin/bash: Set default shell to bash

    Alternative shells:

        /bin/zsh: Zsh (more features, better completion)
        /bin/fish: Fish (very user-friendly)

    Additional groups (add with -G group1,group2,wheel):

        audio: Access audio devices
        video: Access video devices
        storage: Access removable storage
        optical: Access optical drives
        docker: Docker management (if using Docker)

    Example with multiple groups:

    useradd -m -G wheel,audio,video,storage -s /bin/bash yourusername

Configure Sudo Access:

# Install sudo if not already installed
pacman -S --noconfirm sudo

# Edit sudoers file
EDITOR=vim visudo

In the editor, uncomment this line (remove the #):

%wheel ALL=(ALL:ALL) ALL

Save and exit (in vim: press Esc, type :wq, press Enter).
