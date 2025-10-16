# Performance Tuning Guide

## Overview

This guide provides post-installation performance optimizations for the ASUS ROG Zephyrus G16 (GU605MI) with 32GB RAM and RTX 4070. These configurations enhance system responsiveness, reduce disk wear on SSDs, and optimize memory management.

> [!NOTE]
> **Prerequisites**: Complete the [Installation Guide](./Arch%20Linux%20Installation%20Guide%20with%20Full-Disk%20Encryption%20and%20Secure%20Boot.md) before applying these optimizations.

> [!TIP]
> **Quick Start**: If you want tested defaults without deep-diving into configuration details, jump to the Quick Start Configuration section.

## Table of Contents

1. [Understanding Memory Management](#understanding-memory-management)
2. [Quick Start Configuration](#quick-start-configuration)
3. [Essential Optimizations](#essential-optimizations)
   - [Zram Configuration](#zram-configuration)
   - [Swap Priority and Hibernation](#swap-priority-and-hibernation)
   - [Tmpfs Configuration](#tmpfs-configuration)
4. [Optional Optimizations](#optional-optimizations)
   - [Advanced Sysctl Tuning](#advanced-sysctl-tuning)
   - [SSD Optimization](#ssd-optimization)
   - [NVIDIA Power Management](#nvidia-power-management)
5. [Verification and Monitoring](#verification-and-monitoring)
6. [Rollback Procedures](#rollback-procedures)
7. [Troubleshooting](#troubleshooting)

## Understanding Memory Management

Before configuring, understand these core concepts:

**Zram** - Compressed RAM block device used as swap space:
- Lives entirely in RAM (no disk I/O)
- Compresses data 2-3x on average
- Faster than disk swap but uses CPU cycles
- Data is lost on reboot (volatile)

**Disk Swap** - Traditional swap partition on storage:
- Persistent across reboots (required for hibernation)
- Slower than RAM but doesn't consume memory
- Causes SSD wear with frequent writes
- Essential for suspend-to-disk functionality

**Tmpfs** - RAM-based temporary filesystem:
- Used for `/tmp` by default in systemd
- Automatically sized at 50% of RAM unless configured
- Faster than disk for temporary files
- Contents lost on reboot

**The Strategy**:
1. Use zram as primary swap (fast, protects SSD)
2. Keep disk swap as backup and for hibernation
3. Configure tmpfs with reasonable limits
4. Tune kernel parameters for responsive behavior

> [!IMPORTANT]
> **Hibernation Compatibility**: This guide maintains hibernation support by keeping disk swap active at lower priority. If you don't use hibernation, you can disable disk swap entirely after verifying zram stability.

## Quick Start Configuration

For users who want optimized defaults without customization, execute these commands in sequence:

```bash
# 1. Disable zswap (if not already in kernel cmdline)
# Check current status:
cat /proc/cmdline | grep zswap

# If not present, add to kernel cmdline (already done in Installation Guide Section 13)
# Otherwise, verify: zswap.enabled=0

# 2. Configure zram
sudo tee /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
[zram0]
zram-size = 20480
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF

# 3. Adjust disk swap priority for hibernation compatibility
sudo cp /etc/fstab /etc/fstab.backup
sudo sed -i 's/$ .*swap.*defaults $ /\1,pri=10/' /etc/fstab

# Verify the change:
grep swap /etc/fstab

# 4. Configure tmpfs limits
sudo tee -a /etc/fstab > /dev/null <<'EOF'
tmpfs /tmp tmpfs mode=1777,strictatime,noexec,nosuid,nodev,size=4G 0 0
EOF

# 5. Set conservative swappiness
sudo tee /etc/sysctl.d/99-zram.conf > /dev/null <<'EOF'
vm.swappiness = 100
vm.page-cluster = 0
EOF

# 6. Apply configurations
sudo sysctl -p /etc/sysctl.d/99-zram.conf
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service

# 7. Verify everything is working
swapon --show
zramctl
free -h
```
>[!TIP]
>**Reboot Recommended**: While these changes activate immediately, reboot to ensure all configurations persist across boot cycles: `sudo reboot`

## Essential Optimizations

### Zram Configuration

**Purpose**: Primary swap device using compressed RAM for better performance and reduced SSD wear.

**Configuration File**: `/etc/systemd/zram-generator.conf`

```bash
sudo tee /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
[zram0]
zram-size = 20480
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF
```

**Parameter Explanation**:

| Parameter | Value | Reasoning |
| --- | --- | --- |
| zram-size | 20480 (20GB) | ~62% of 32GB RAM - balanced capacity without over-commitment |
| compression-algorithm | zstd | Best balance of compression ratio (2.5-3x) and speed |
| swap-priority | 100 | Higher than disk swap (10) to prefer zram |
| fs-type | swap | Explicitly declare as swap device |

>[!NOTE]
>**Size Calculation**: The 20GB allocation provides comfortable headroom for typical workloads. With zstd compression averaging 2.5-3x, this effectively provides 50-60GB of swap space.

**Alternative Compression Algorithms**:

```bash
# For maximum compression (slower CPU):
compression-algorithm = zstd

# For fastest compression (lower ratio):
compression-algorithm = lz4

# For balanced performance:
compression-algorithm = lzo
```

**Activate Zram**:

```bash
sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service
sudo systemctl enable systemd-zram-setup@zram0.service
```

```bash
Verify Configuration:

# Check zram device
zramctl

# Expected output:
# NAME       ALGORITHM DISKSIZE DATA COMPR TOTAL STREAMS MOUNTPOINT
# /dev/zram0 zstd         20G   4K   74B   12K       8 [SWAP]

# Verify swap priority
swapon --show

# Expected output:
# NAME       TYPE      SIZE USED PRIO
# /dev/zram0 partition  20G   0B  100
# /dev/dm-X  partition  XG    0B   10
```

Swap Priority and Hibernation

**Purpose**: Configure swap device priorities to prefer zram while maintaining hibernation capability.

**Background**: Linux uses swap devices in priority order. Higher priority values are used first. Our configuration:
- Zram: Priority 100 (primary swap)
- Disk swap: Priority 10 (hibernation and overflow)

**Modify Disk Swap Entry**:

```bash
# Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# View current swap entry
grep swap /etc/fstab

# Add priority to disk swap
sudo sed -i 's/$ .*swap.*defaults $ /\1,pri=10/' /etc/fstab

# Verify modification
grep swap /etc/fstab
# Expected: /dev/mapper/leo--os-swap none swap defaults,pri=10 0 0
```

>[!WARNING]
>**Hibernation Requirement**: Disk swap must remain enabled and be at least equal to your RAM size (32GB) for hibernation to work.
>Do not disable disk swap if you use systemctl hibernate.

Reactivate Swap:

# Turn off all swap
sudo swapoff -a

# Reactivate with new priorities
sudo swapon -a

# Verify priorities
swapon --show

Tmpfs Configuration

Purpose: Configure /tmp with explicit limits and security options to prevent memory exhaustion.

Default Behavior: Systemd mounts /tmp as tmpfs with 50% RAM allocation (16GB on your system). This is usually sufficient but lacks security hardening.

Recommended Configuration:

Add this line to /etc/fstab:

tmpfs /tmp tmpfs mode=1777,strictatime,noexec,nosuid,nodev,size=4G 0 0

Mount Option Explanation:
Option 	Purpose
mode=1777 	Preserves sticky bit (only file owner can delete)
strictatime 	Updates access times (filesystem consistency)
noexec 	Prevents execution of binaries from /tmp (security)
nosuid 	Ignores setuid/setgid bits (security)
nodev 	Prevents device file creation (security)
size=4G 	Maximum allocation (demand-allocated, not reserved)

    [!NOTE]
    Memory Allocation: The size=4G is a ceiling, not a reservation. Tmpfs only consumes RAM for actual file content. Empty /tmp uses negligible memory.

Apply Configuration:

# Remount with new options
sudo mount -o remount /tmp

# Verify mount options
mount | grep /tmp

# Expected output:
# tmpfs on /tmp type tmpfs (rw,nosuid,nodev,noexec,relatime,size=4194304k)

    [!TIP]
    Size Tuning: If you compile large projects in /tmp or use RAM-intensive applications, increase to size=8G. For minimal systems, size=2G suffices.

Optional Optimizations
Advanced Sysctl Tuning

Purpose: Fine-tune kernel memory management behavior for responsive performance with zram.

Configuration File: /etc/sysctl.d/99-zram.conf

sudo tee /etc/sysctl.d/99-zram.conf > /dev/null <<'EOF'
# Aggressive swapping to zram (fast compressed RAM)
vm.swappiness = 100

# Disable swap readahead (inefficient with compression)
vm.page-cluster = 0
EOF

Parameter Explanation:

vm.swappiness = 100

    Controls how aggressively kernel swaps memory to swap space
    Range: 0-200 (default: 60)
    Value 100: Treat swap and RAM equally
    Rationale: Since zram is compressed RAM (fast), aggressive swapping increases effective memory capacity without performance penalty

vm.page-cluster = 0

    Controls swap readahead page count
    Range: 0-8 (default: 3, reads 2^3 = 8 pages ahead)
    Value 0: Disable readahead
    Rationale: Readahead optimizes disk seeks but adds overhead with zram compression. Disable for zram-primary configurations.

    [!IMPORTANT]
    Use Case Specific: These aggressive settings benefit zram configurations. If you primarily use disk swap, revert to conservative values: vm.swappiness=10 and vm.page-cluster=3.

Apply Sysctl Configuration:

sudo sysctl -p /etc/sysctl.d/99-zram.conf

# Verify active values
sysctl vm.swappiness
sysctl vm.page-cluster

Alternative Profiles:

# Conservative (prefer RAM, minimal swap):
vm.swappiness = 10
vm.page-cluster = 3

# Balanced (moderate swap usage):
vm.swappiness = 60
vm.page-cluster = 3

# Aggressive for zram (current recommendation):
vm.swappiness = 100
vm.page-cluster = 0

SSD Optimization

Purpose: Reduce write amplification and extend SSD lifespan with periodic TRIM operations.

    [!NOTE]
    Already Configured: If you followed the Installation Guide Section 17, fstrim.timer is already enabled. This section is for verification and troubleshooting.

Verify TRIM Support:

# Check device TRIM capabilities
lsblk --discard

# Look for non-zero DISC-GRAN and DISC-MAX values
# Expected output for NVMe SSD:
# NAME        DISC-GRAN DISC-MAX
# nvme0n1           512B       2G

Check TRIM Timer Status:

# Verify timer is active
systemctl status fstrim.timer

# Check last execution
systemctl list-timers fstrim.timer

# View execution history
journalctl -u fstrim.service

Manual TRIM Execution (for testing):

# Run TRIM on all mounted filesystems
sudo fstrim -av

# Expected output:
# /boot: 800 MiB (838860800 bytes) trimmed on /dev/nvme0n1p1
# /: 45 GiB (48318382080 bytes) trimmed on /dev/dm-1
# /home: 180 GiB (193273528320 bytes) trimmed on /dev/dm-3

    [!WARNING]
    Continuous TRIM: The Installation Guide uses fstrim.timer (periodic TRIM) instead of discard mount option (continuous TRIM) because periodic TRIM is more reliable and reduces write overhead.

NVIDIA Power Management

Purpose: Optimize power consumption on laptops with NVIDIA Optimus graphics.

Check Available Power Profiles:

# List profiles
powerprofilesctl list

# Expected output:
# * performance:
#     Driver:     placeholder
# 
#   balanced:
#     Driver:     placeholder
# 
#   power-saver:
#     Driver:     placeholder

Set Power Profile:

# For battery life (recommended on battery):
powerprofilesctl set power-saver

# For maximum performance (recommended when plugged in):
powerprofilesctl set performance

# For balanced usage:
powerprofilesctl set balanced

# Check current profile:
powerprofilesctl get

Automatic Profile Switching:

The power-profiles-daemon (enabled in Installation Guide Section 17) automatically switches profiles based on power source. No additional configuration needed.

Verify NVIDIA Power State:

# Check NVIDIA GPU status
nvidia-smi

# Monitor power consumption
watch -n 1 nvidia-smi --query-gpu=power.draw --format=csv

    [!TIP]
    Prime Offload: For Optimus laptops, use prime-run <application> to explicitly run applications on NVIDIA GPU. Most applications use Intel integrated graphics by default for power savings.

Verification and Monitoring
Comprehensive System Check

Execute these commands after applying configurations:

# 1. Verify zram is active and being used
echo "=== Zram Status ==="
zramctl
swapon --show

# 2. Check memory usage and compression ratio
echo -e "\n=== Memory Overview ==="
free -h

# 3. View zram statistics
echo -e "\n=== Zram Statistics ==="
cat /sys/block/zram0/mm_stat
# Column meanings:
# 1: orig_data_size (uncompressed data)
# 2: compr_data_size (compressed data)
# 3: mem_used_total (total memory used)
# 4: mem_limit (memory limit)
# 5: mem_used_max (peak memory usage)
# 6: same_pages (deduplicated pages)
# 7: pages_compacted (compacted pages)
# 8: huge_pages (huge pages count)

# 4. Calculate compression ratio
echo -e "\n=== Compression Ratio ==="
awk '{printf "Ratio: %.2fx\n",  $ 1/ $ 2}' /sys/block/zram0/mm_stat

# 5. Verify tmpfs configuration
echo -e "\n=== Tmpfs Mounts ==="
mount | grep tmpfs

# 6. Check sysctl values
echo -e "\n=== Sysctl Configuration ==="
sysctl vm.swappiness vm.page-cluster

# 7. Verify TRIM timer
echo -e "\n=== TRIM Status ==="
systemctl status fstrim.timer --no-pager

# 8. Check swap usage over time
echo -e "\n=== Swap Usage ==="
watch -n 5 'free -h && echo "" && swapon --show'

Performance Monitoring Commands

Real-time Memory Monitoring:

# Continuous memory and swap monitoring
watch -n 1 'free -h; echo ""; swapon --show'

# Detailed zram statistics
watch -n 1 'zramctl; echo ""; cat /sys/block/zram0/mm_stat'

Historical Analysis:

# System resource usage over time (requires sysstat package)
sudo pacman -S sysstat
sudo systemctl enable --now sysstat

# View historical memory usage
sar -r 1 10  # 10 samples at 1-second intervals

# View swap activity
sar -S 1 10

Disk I/O Monitoring (verify reduced swap writes):

# Monitor disk writes (should see minimal swap partition writes)
sudo iotop -o

# Check disk write statistics
iostat -x 2  # Update every 2 seconds

Expected Outcomes

After successful configuration, you should observe:

Memory Efficiency:

    Compression ratio: 2.5-3.0x with zstd
    Effective swap capacity: ~50-60GB from 20GB zram
    Minimal disk swap usage unless memory pressure is extreme

Performance Indicators:

    Faster application switching (no disk swap delays)
    Reduced SSD write operations (verify with iostat)
    Consistent system responsiveness under memory pressure

Swap Priority Verification:

swapon --show
# Expected output:
# NAME           TYPE      SIZE   USED PRIO
# /dev/zram0     partition  20G    2G  100  <- Used first
# /dev/dm-3      partition  32G    0B   10  <- Hibernation backup

    [!TIP]
    Baseline Testing: Before applying optimizations, run sysbench or similar benchmarks to establish baseline performance. Compare after configuration to quantify improvements.

Rollback Procedures

If you experience issues or want to revert to default configuration:
Complete Rollback

# 1. Stop and disable zram
sudo systemctl stop systemd-zram-setup@zram0.service
sudo systemctl disable systemd-zram-setup@zram0.service

# 2. Remove zram configuration
sudo rm /etc/systemd/zram-generator.conf

# 3. Restore original fstab
sudo cp /etc/fstab.backup /etc/fstab

# 4. Remove custom sysctl configuration
sudo rm /etc/sysctl.d/99-zram.conf

# 5. Restore default sysctl values
sudo sysctl vm.swappiness=60
sudo sysctl vm.page-cluster=3

# 6. Reactivate swap
sudo swapoff -a
sudo swapon -a

# 7. Verify rollback
swapon --show
sysctl vm.swappiness vm.page-cluster

# 8. Reboot to ensure clean state
sudo reboot

Partial Rollback (Keep Zram, Revert Tuning)

# Revert to conservative swappiness while keeping zram
sudo tee /etc/sysctl.d/99-zram.conf > /dev/null <<'EOF'
vm.swappiness = 10
vm.page-cluster = 3
EOF

sudo sysctl -p /etc/sysctl.d/99-zram.conf

Rollback Tmpfs Only

# Remove custom tmpfs entry from fstab
sudo sed -i '/tmpfs.*\/tmp/d' /etc/fstab

# Remount with systemd defaults
sudo systemctl daemon-reload
sudo mount -o remount /tmp

    [!IMPORTANT]
    Backup Reminder: Always keep /etc/fstab.backup until you verify new configuration works across multiple reboots. Delete old backups after confirming stability.

Troubleshooting
High Memory Usage After Configuration

Symptom: System uses more swap than expected with zram enabled.

Diagnosis:

# Check what's consuming zram
sudo zramctl --output-all

# Identify memory-heavy processes
ps aux --sort=-%mem | head -20

Solutions:

# Reduce swappiness to be less aggressive
sudo sysctl vm.swappiness=60

# Increase zram size if compression ratio is poor
sudo tee -a /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
zram-size = 24576
EOF

sudo systemctl restart systemd-zram-setup@zram0.service

System Freeze Under Memory Pressure

Symptom: System becomes unresponsive when memory is exhausted.

Cause: Over-aggressive swapping with insufficient zram capacity.

Solutions:

# Reduce swappiness to prevent excessive swapping
sudo sysctl vm.swappiness=40

# Ensure disk swap is active as overflow
swapon --show  # Verify both zram and disk swap are present

# Enable memory pressure monitoring
sudo tee /etc/systemd/system/low-memory-monitor.service > /dev/null <<'EOF'
[Unit]
Description=Low Memory Monitor

[Service]
Type=simple
ExecStart=/usr/bin/systemd-oomd

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now low-memory-monitor.service

Hibernation Fails

Symptom: systemctl hibernate fails or system doesn't resume properly.

Diagnosis:

# Verify resume parameter in kernel cmdline
cat /proc/cmdline | grep resume

# Check disk swap size
swapon --show | grep dm
# Must be >= 32GB for 32GB RAM system

# Test hibernation manually
sudo systemctl hibernate

Solutions:

# Ensure disk swap has higher capacity than RAM
# Resize swap LV if needed (requires reinstall or complex resize)

# Verify disk swap priority is lower than zram
sudo sed -i 's/pri=[0-9]*/pri=10/' /etc/fstab

# Ensure swap is properly configured in fstab
grep swap /etc/fstab
# Expected: /dev/mapper/leo--os-swap none swap defaults,pri=10 0 0

# Rebuild initramfs to include resume hook (already done in Installation Guide)

Zram Not Activating on Boot

Symptom: After reboot, zramctl shows no devices.

Diagnosis:

# Check service status
systemctl status systemd-zram-setup@zram0.service

# View service logs
journalctl -u systemd-zram-setup@zram0.service

Solutions:

# Verify configuration file exists and is valid
cat /etc/systemd/zram-generator.conf

# Check for syntax errors
systemd-zram-generator --verify

# Manually enable service
sudo systemctl enable systemd-zram-setup@zram0.service

# Test activation
sudo systemctl start systemd-zram-setup@zram0.service

Poor Zram Compression Ratio

Symptom: Compression ratio below 2.0x, indicating inefficient memory use.

Diagnosis:

# Check current compression ratio
awk '{printf "Compression Ratio: %.2fx\n",  $ 1/ $ 2}' /sys/block/zram0/mm_stat

# Verify compression algorithm
zramctl | grep ALGORITHM

Solutions:

# Try different compression algorithm
sudo tee /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
[zram0]
zram-size = 20480
compression-algorithm = lz4
swap-priority = 100
EOF

sudo systemctl restart systemd-zram-setup@zram0.service

# Compare algorithms:
# zstd: Best ratio (~2.5-3x), moderate CPU
# lzo: Balanced ratio (~2.2x), lower CPU
# lz4: Fast compression (~2x), lowest CPU

Excessive CPU Usage from Compression

Symptom: High CPU load attributed to zram compression/decompression.

Diagnosis:

# Monitor CPU usage
top
# Look for [zswap] or compression-related kernel threads

# Check zram statistics
cat /sys/block/zram0/io_stat

Solutions:

# Switch to faster compression algorithm
sudo tee /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
[zram0]
zram-size = 20480
compression-algorithm = lz4
swap-priority = 100
EOF

# Reduce zram size to swap less frequently
sudo tee /etc/systemd/zram-generator.conf > /dev/null <<'EOF'
[zram0]
zram-size = 16384
compression-algorithm = lz4
swap-priority = 100
EOF

sudo systemctl restart systemd-zram-setup@zram0.service

Tmpfs Filling Up /tmp

Symptom: Applications fail with "No space left on device" errors for /tmp.

Diagnosis:

# Check /tmp usage
df -h /tmp

# Identify large files
sudo du -sh /tmp/* | sort -h

Solutions:

# Increase tmpfs size
sudo sed -i 's/size=[0-9]*G/size=8G/' /etc/fstab

# Remount with new size
sudo mount -o remount /tmp

# Verify new size
df -h /tmp

# Alternative: Use disk-based /tmp (revert tmpfs)
sudo sed -i '/tmpfs.*\/tmp/d' /etc/fstab
sudo systemctl daemon-reload
sudo reboot

    [!TIP]
    Log Monitoring: Keep an eye on system logs for memory-related warnings: journalctl -p warning -b | grep -i mem
