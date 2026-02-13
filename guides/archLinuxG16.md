# Chapter I - Drive Preparation

## Part I - Drive Identification

Before invoking any destructive commands, we must confirm with absolute certainty the physical path of the Linux target drive. Since both installed NVMe drives share the same 2TB capacity, size alone cannot serve as a differentiator. We must rely on partition table inspection and filesystem type analysis.

> [!WARNING] 
> **Critical Safety Check - Irreversible Operation Ahead**
> 
> The commands in Part III of this chapter will **permanently destroy all data** on the targeted controller. Misidentifying the drive means wiping your Windows installation with zero possibility of recovery.
> 
> The typical mapping is:
> 
> - `/dev/nvme0n1` — First M.2 slot (usually Windows).
> - `/dev/nvme1n1` — Second M.2 slot (usually the target).
> 
> **Do not trust this assumption.** You must verify it empirically using the commands below.

Execute the following to map out your current drive topology:

```bash
lsblk -o NAME,MODEL,SIZE,TYPE,FSTYPE
```

Then inspect the partition tables of each controller individually:

```bash
fdisk -l /dev/nvme0n1
fdisk -l /dev/nvme1n1
```

> [!NOTE] 
> **How to Read the Output**
> 
| Indicator Found on Drive                                                 | Conclusion                                                    |
| ------------------------------------------------------------------------ | ------------------------------------------------------------- |
| `Microsoft Basic Data`, `NTFS`, `EFI System` with a Windows Boot Manager | This is your **Windows drive**. Do not touch this controller. |
| Empty table, unknown filesystem, or a previous Linux installation        | This is your **target drive**.                                |

For the remainder of this guide, we assume the target is **`/dev/nvme1`** (the controller) and **`/dev/nvme1n1`** (the namespace). Substitute accordingly if your verification reveals a different mapping.

> [!TIP] 
> **Controller vs. Namespace - Terminology Clarification**
> 
> NVMe uses a hierarchical addressing model. The **controller** (`/dev/nvme1`) is the physical hardware device. The **namespace** (`/dev/nvme1n1`) is a logical storage unit exposed by that controller. Most consumer SSDs expose a single namespace per controller.
> 
> - Sanitize commands target the **controller** (`/dev/nvme1`) — they wipe everything the hardware manages, including over-provisioned and hidden areas.
> - Partition and format commands target the **namespace** (`/dev/nvme1n1`) — they operate only on the visible logical address space.
> 
> This distinction matters in Part III.

## Part II - Capability Analysis

Before erasing, we must query the NVMe controller's firmware to determine which sanitize methods it supports. Not all controllers implement every method defined in the NVMe specification.

**Crypto Erase** is the preferred method: it destroys the internal media encryption key, rendering all stored data cryptographically irrecoverable in a matter of seconds. **Block Erase** physically overwrites every block and can take significantly longer.

Run the following against your **target controller**:

```bash
nvme id-ctrl /dev/nvme1 -H | grep -E 'Sanitize|Crypto'
```

**Interpret the output against this table:**

|Field|Desired Value|Implication|
|---|---|---|
|`Crypto Erase Sanitize Operation Supported`|**Yes**|Preferred — executes in seconds.|
|`Block Erase Sanitize Operation Supported`|Yes|Fallback — may take minutes to hours depending on drive capacity.|
|`Overwrite Sanitize Operation Supported`|Yes|Last resort — slowest method, writes a fixed pattern across all blocks.|

> [!NOTE] 
> **Why Crypto Erase Is Instantaneous**
> 
> Modern NVMe SSDs encrypt all data at rest using an internal AES key, regardless of whether the user has enabled software encryption. This is called **Self-Encrypting Drive (SED)** functionality, defined in the TCG Opal specification. Crypto Erase does not overwrite data — it destroys the AES key. Without the key, the stored ciphertext is computationally indistinguishable from random noise.
> 
> _Reference: NVMe Specification, Revision 2.0, Section 5.23 — Sanitize Operations._

> [!WARNING] 
> If **neither** Crypto Erase nor Block Erase shows `Supported`, your controller's firmware does not implement the `sanitize` command set. Proceed directly to **Option C** (Format with Secure Erase) in Part III.

## Part III - Execution of Sanitize

> [!Abstract] 
> **Sanitize vs. Format — Understanding the Scope**
> 
> The NVMe specification defines two distinct erasure mechanisms:
> 
> 
| Characteristic      | `sanitize`                                                                       | `format --ses`                                               |
| ------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **Scope**           | Entire controller — all namespaces, all caches, all over-provisioned area.       | Single namespace only.                                       |
| **Survives reboot** | Yes — the controller resumes the operation autonomously after power restoration. | No — interruption may leave the drive in an undefined state. |
| **Guarantees**      | Data is irrecoverable per NIST SP 800-88 "Purge" level.                          | Data is irrecoverable per NIST SP 800-88 "Clear" level.      |

### Option A - Crypto Erase (Fastest and Secure)

If the capability check in Part II confirmed support for Crypto Erase:

```bash
nvme sanitize /dev/nvme1 -a start-crypto-erase
```

> [!TIP] 
> This operation completes in under 5 seconds on most modern NVMe controllers. If the command returns immediately with no error, it has likely already finished. Verify with the sanitize log in Part IV.

### Option B - Block Erase (Slower)

If Crypto Erase is unavailable but Block Erase is supported:

```bash
nvme sanitize /dev/nvme1 -a start-block-erase
```

> [!NOTE]
> On a 2TB drive, Block Erase may take anywhere from 2 to 30 minutes depending on the controller's internal write speed. Do not interrupt the process.

### Option C - Fallback to Format

If `sanitize` is entirely unsupported by the firmware, fall back to the Format command with Secure Erase Settings (`ses=1`):

```bash
nvme format /dev/nvme1n1 --ses=1
```

> [!WARNING] 
> Note the target difference: this command targets the **namespace** (`/dev/nvme1n1`), not the controller. The `format` command does not guarantee erasure of over-provisioned or controller-cached data. It is acceptable for our purposes (we are layering LUKS2 encryption on top), but it is the weakest of the three options from a forensic standpoint.

## Part IV - Verification

Monitor the sanitize operation's progress. **Do not power off the machine during this step.**

```bash
nvme sanitize-log /dev/nvme1
```

**Interpret the output:**

|Field|Completion Value|Meaning|
|---|---|---|
|`Sanitize Status (SSTAT)`|`0x0101`|Sanitize completed successfully.|
|`Sanitize Progress (SPROG)`|`65535`|100% complete (value is reported on a 0–65535 scale).|

> [!TIP] 
> If `SPROG` shows a value less than `65535`, the operation is still in progress. Re-run the `sanitize-log` command after a short interval. You can loop it for convenience:
> 
> ```bash
> watch -n 5 nvme sanitize-log /dev/nvme1
> ```
> 
> Press `Ctrl+C` to exit once `SPROG` reaches `65535`.

Once the sanitize operation reports completion, the kernel's block device layer may still cache the old partition table. Force a rescan:

```bash
nvme list
lsblk
```

> [!NOTE] 
> **Expected Post-Sanitize State**
> 
> The drive `/dev/nvme1n1` should appear in `lsblk` with its full 2TB capacity and **no child partitions** listed beneath it. The `FSTYPE` column should be blank. If you still see old partitions, run:

```bash
partprobe /dev/nvme1n1
```

This forces the kernel to re-read the partition table from disk. If partitions persist after this, the sanitize operation may not have completed successfully — return to the sanitize log and investigate.

# Chapter II - Live Environment Setup

## Part I - Network Configuration (Wi-Fi)

The Arch Linux ISO ships with `iwd` (Interactive Wireless Daemon) as its default wireless management tool. Your ASUS ROG Zephyrus G16 is equipped with an Intel Wi-Fi 6E AX211 chipset, which has mature kernel support via the `iwlwifi` driver included in the `linux-firmware` package on the ISO.

> [!Abstract] 
> **Why iwd and Not wpa_supplicant?**
> 
> The Arch ISO chose `iwd` as its default because it is a modern, self-contained daemon that handles scanning, authentication (WPA2/WPA3), and DHCP in a single process. Unlike `wpa_supplicant`, it does not require a separate DHCP client to be configured. This makes it ideal for the ephemeral live environment.
> 
> Note that once the system is installed, we will switch to `NetworkManager` with `iwd` as its backend — a more feature-complete stack for daily use.
> 
> _Reference: Arch Wiki — iwd ([https://wiki.archlinux.org/title/Iwd](https://wiki.archlinux.org/title/Iwd))_

### A - Initialize the Interface

Enter the `iwd` interactive prompt:

```bash
iwctl
```

Your shell prompt will change to `[iwd]#`, indicating you are inside the `iwd` control interface. All subsequent commands in this Part (B through E) are executed within this prompt, not in your regular shell.

### B - Identify the Wireless Device

List the physical wireless devices detected by the kernel:

```bash
device list
```

> [!NOTE] 
> **Expected Output**
> 
> You should see a single wireless adapter, typically named `wlan0`. The important columns to verify are:
> 
> 
| Column    | Expected Value                                  |
| --------- | ----------------------------------------------- |
| `Name`    | `wlan0` (or `wlp0s20f3` on some configurations) |
| `Mode`    | `station`                                       |
| `Powered` | `on`                                            |

### C - Scan for Networks

Trigger an active scan on your wireless interface. Replace `wlan0` if your device name differs:

```bash
station wlan0 scan
```

> [!NOTE] 
> This command produces no visible output on success. It silently instructs the radio to sweep all channels and collect beacon frames from nearby access points. The results are retrieved in the next step.

### D - List Available SSIDs

Display the networks discovered during the scan:

```bash
station wlan0 get-networks
```

Your home network's SSID should appear in the list. Note the `Security` column — it will typically show `psk` for WPA2/WPA3-Personal networks.

### E - Connect

Connect to your access point. You will be prompted interactively for the passphrase:

```bash
station wlan0 connect "YOUR_SSID"
```

Replace `"YOUR_SSID"` with the exact name of your network as displayed in step D. If the SSID contains spaces, the quotes are mandatory.

> [!TIP] 
> **WPA3 Networks**
> 
> If your router is configured for WPA3-only (SAE authentication), `iwd` handles this transparently — no additional flags are required. The negotiation is automatic based on the access point's advertised capabilities.

### F - Exit and Verify

Type `exit` to leave the `iwd` interactive prompt and return to your regular shell.

Verify that your interface has obtained an IP address via DHCP:

```bash
ip addr show wlan0
```

Look for an `inet` line showing a valid IPv4 address (e.g., `192.168.x.x/24`). If you only see `inet6` with a `fe80::` prefix, DHCP did not succeed.

Confirm end-to-end internet connectivity:

```bash
ping -c 3 archlinux.org
```

> [!WARNING] 
> **If `ping` Fails with `Temporary failure in name resolution`**
> 
> This indicates DNS is not configured. The Arch ISO should auto-configure DNS via DHCP, but if it has not, you can set a temporary resolver manually:
> 
> ```bash
> echo "nameserver 1.1.1.1" > /etc/resolv.conf
> ```
> 
> Then retry the `ping` command. This is a temporary fix for the live environment only — the installed system will use `NetworkManager`'s DNS handling.

> [!TIP] 
> **If `device list` Shows Your Adapter as "Powered Off"**
> 
> This is a common issue on laptops where a hardware or software RF kill switch is active. Run the following from your regular shell (not inside `iwctl`):
>```bash
>rfkill list all
>```
>
>If the output shows `Soft blocked: yes` for your wireless device:
>```bash
>rfkill unblock wifi
>```

 If it shows `Hard blocked: yes`, you have a physical toggle or BIOS setting disabling the radio. Check your ASUS BIOS under **Advanced > Wireless** and ensure the adapter is enabled. Then retry from step A.

## Part II - Time Synchronization

Accurate system time is a non-negotiable prerequisite before proceeding. Two critical processes depend on it:

> [!Abstract] 
> **Why Time Accuracy Matters at This Stage**
> 
> 
| Process                         | Consequence of Clock Skew                                                                                                                                                                                     |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pacman` mirror synchronization | HTTPS/TLS certificate validation will fail if the system clock deviates by more than a few minutes. `pacstrap` will refuse to download packages, producing opaque SSL errors.                                 |
| Filesystem metadata             | `ext4`, `btrfs`, and `xfs` all embed timestamps in their superblocks and journal entries. A wildly incorrect clock creates forensic confusion and can interfere with incremental backup tools like `snapper`. |
| LUKS2 header metadata           | The LUKS2 header records a creation timestamp. While not functionally critical, an incorrect date complicates future auditing.                                                                                |

### A - Enable Network Time Protocol (NTP)

Instruct `systemd-timesyncd` to synchronize the hardware clock against upstream NTP servers:

```bash
timedatectl set-ntp true
```

> [!NOTE] 
> The Arch ISO pre-configures `systemd-timesyncd` to query the `0.arch.pool.ntp.org` through `3.arch.pool.ntp.org` server pool. This command activates the daemon — it does not install anything new.

### B - Verify Synchronization

Query the synchronization state:

```bash
timedatectl status
```

**Validate the following fields:**

|Field|Required Value|Meaning|
|---|---|---|
|`System clock synchronized`|`yes`|The daemon has successfully received at least one NTP response.|
|`NTP service`|`active`|The `systemd-timesyncd` daemon is running.|
|`RTC in local TZ`|`no`|The hardware clock is stored in UTC, which is correct for Linux.|

> [!WARNING] 
> **If `System clock synchronized` Reads `no`**
> 
> Wait 10–15 seconds and re-run `timedatectl status`. NTP synchronization requires a round-trip to an external server. If it still reads `no` after 30 seconds, verify your internet connectivity (Part I, step F). Without a working network path, NTP cannot function.

### C - Set Timezone

Setting the timezone in the live environment is optional but recommended. It ensures that all log entries generated during the installation carry human-readable local timestamps, which simplifies debugging if something fails later.

List available timezones matching your region:

```bash
timedatectl list-timezones | grep Paris
```

Set your specific zone:

```bash
timedatectl set-timezone Europe/Paris
```

> [!TIP] 
> **Common Timezone Identifiers**
> 
|Location|Identifier|
|---|---|
|Paris, France|`Europe/Paris`|
|New York, USA|`America/New_York`|
|London, UK|`Europe/London`|
|Tokyo, Japan|`Asia/Tokyo`|

The full list follows the IANA Time Zone Database naming convention: `Region/City`. If unsure, pipe the full list through `less`:

```bash
timedatectl list-timezones | less
```

> [!NOTE] 
> **This Timezone Setting Is Ephemeral**
> 
> The timezone configured here applies only to the live ISO session. In Chapter VI (System Configuration), we will set the permanent timezone for the installed system via a symlink to `/usr/share/zoneinfo/`. The two configurations are independent.

# Chapter III - Disk Layout & Encryption

## Part I - Partitioning with Parted

We will now create a GPT partition structure on the sanitized drive using `parted`. The layout follows a three-partition architecture designed for dual-purpose isolation: the operating system and personal data live on one encrypted volume, while virtual machine storage occupies a second, independently encrypted volume.

> [!Abstract] 
> **Partition Layout Overview**
>
> 
| Partition   | Label      | Size          | Purpose                                                              |
| ----------- | ---------- | ------------- | -------------------------------------------------------------------- |
| `nvme1n1p1` | `ESP`      | 2 GiB         | EFI System Partition — stores bootloader, UKI images, and microcode. |
| `nvme1n1p2` | `cryptOs`  | ~1.3 TB (70%) | LUKS2-encrypted container for the Arch Linux root, home, var and swap.   |
| `nvme1n1p3` | `cryptVMs` | ~570 GB (30%) | LUKS2-encrypted container for virtual machine disk images.           |

### A - Create the Partition Table

Initialize the drive with a GUID Partition Table (GPT). This is mandatory for UEFI booting:

```bash
parted -s /dev/nvme1n1 mklabel gpt
```

> [!WARNING] 
> **Point of No Return for Drive Structure**
> 
> This command destroys any residual partition table metadata on the drive. If the sanitize operation in Chapter I completed successfully, this is redundant but harmless. If you skipped Chapter I for any reason, this is your last chance to reconsider — `mklabel` will render any existing partitions unrecoverable without forensic tools.

### B - Create the EFI System Partition (ESP)

We allocate 2 GiB to the ESP. This is deliberately generous:

```bash
parted -s /dev/nvme1n1 mkpart "ESP" fat32 1MiB 2049MiB
parted -s /dev/nvme1n1 set 1 esp on
```

> [!NOTE] 
> **Why 2 GiB for the ESP?**
> 
> The UEFI specification requires a minimum of 100 MiB. Many guides recommend 512 MiB. We allocate 2 GiB for a specific reason: this system will store **Unified Kernel Images** (UKIs), which bundle the kernel, initramfs, microcode, and command line into a single `.efi` file. Each UKI typically weighs 60–100 MB. With two kernels (`linux-lts` and `linux-g14`), plus their fallback images and signed copies, space consumption adds up quickly.
> 
> 2 GiB provides ample headroom for kernel updates, rollback images, and future additions without ever needing to resize the ESP.
> 
> _Reference: Arch Wiki — EFI system partition ([https://wiki.archlinux.org/title/EFI_system_partition](https://wiki.archlinux.org/title/EFI_system_partition))_

> [!TIP] 
> **Why Start at 1 MiB, Not 0?**
> 
> The first 1 MiB of the disk is reserved for the GPT protective MBR and partition table headers. Starting at `1MiB` ensures the first partition is aligned to the physical erase block boundary of the SSD (typically 1 MiB or 4 MiB). Misalignment at this stage would propagate performance penalties to every I/O operation for the life of the partition.

### C - Create the System Partition (70%)

This partition will hold the LUKS2 container for your entire Arch Linux installation — root filesystem, home directory, variable data, snapshots, and swap:

```bash
parted -s /dev/nvme1n1 mkpart "cryptOs" 2049MiB 70%
```

### D - Create the VM Partition (Remaining 30%)

The remaining space is dedicated to an isolated LUKS2 container for virtual machine disk images:

```bash
parted -s /dev/nvme1n1 mkpart "cryptVMs" 70% 100%
```

> [!NOTE] 
> **Why Separate the VM Storage?**
> 
> There are three reasons for this architectural decision:
> 
> **A - Filesystem Isolation.** The OS partition will use Btrfs (optimized for snapshots and compression). VM disk images perform poorly on Copy-on-Write filesystems due to fragmentation. The VM partition will use XFS, which is the industry standard for large, sequentially-written files.
> 
> **B - Security Compartmentalization.** Each LUKS2 container can use a different passphrase. If you handle client data inside VMs, this provides a cryptographic boundary — unlocking the OS does not automatically expose VM contents.
> 
> **C - Snapshot Hygiene.** Btrfs snapshots of the root filesystem will not inadvertently include 200+ GB of VM images, keeping snapshot sizes manageable and rollback operations fast.

### E - Verification and Alignment Check

Display the resulting partition table:

```bash
parted /dev/nvme1n1 print
```

**Expected output structure:**

|Number|Start|End|Size|File system|Name|Flags|
|---|---|---|---|---|---|---|
|1|1049kB|2147MB|2147MB|fat32|ESP|boot, esp|
|2|2147MB|~1.4TB|~1.3TB||cryptOs||
|3|~1.4TB|2.0TB|~573GB||cryptVMs||

Verify that partitions 2 and 3 are optimally aligned to the SSD's physical sector boundaries:

```bash
parted /dev/nvme1n1 align-check optimal 2
parted /dev/nvme1n1 align-check optimal 3
```

Both commands must return `aligned`. If either reports misalignment, delete the offending partition and recreate it — the performance penalty of misaligned writes on NVMe is severe and permanent for the life of that partition.

> [!TIP] 
> **Partition 1 Alignment**
> 
> You can also verify partition 1 with `align-check optimal 1`, but since we explicitly started it at `1MiB` (a power-of-two boundary), it is guaranteed to be aligned. Partitions 2 and 3 warrant verification because `parted` internally converts the `70%` and `100%` values to byte offsets, which could theoretically land on a non-aligned boundary depending on the drive's total capacity.

## Part II - LUKS2 Encryption (Dual Containers)

We will now encrypt Partition 2 and Partition 3 as independent LUKS2 volumes. Each container is self-contained: it has its own header, its own master key, and its own passphrase.

> [!Abstract] 
> **LUKS2 Parameter Rationale**
> 
> Every flag passed to `cryptsetup luksFormat` below serves a specific purpose. The following table explains the full parameter set:
> 
| Parameter       | Value             | Rationale                                                                                                                                                                                                                     |
| --------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--type`        | `luks2`           | LUKS2 supports `argon2id`, metadata redundancy, and 4096-byte sectors. LUKS1 does not.                                                                                                                                        |
| `--cipher`      | `aes-xts-plain64` | AES-XTS is the standard mode for disk encryption. Your Intel CPU includes AES-NI hardware acceleration, making this effectively free in terms of performance overhead.                                                        |
| `--key-size`    | `512`             | In XTS mode, the key is split into two 256-bit halves (one for encryption, one for tweak). This yields AES-256 effective security.                                                                                            |
| `--hash`        | `sha512`          | Hash algorithm used for key derivation. SHA-512 is faster than SHA-256 on 64-bit CPUs due to its wider internal state.                                                                                                        |
| `--pbkdf`       | `argon2id`        | Password-Based Key Derivation Function. Argon2id is memory-hard, making brute-force attacks via GPU or ASIC economically infeasible. It is the winner of the Password Hashing Competition (2015) and is recommended by OWASP. |
| `--sector-size` | `4096`            | Matches the physical sector size of your NVMe SSD. See the note below.                                                                                                                                                        |
_Reference: cryptsetup(8) man page; NIST SP 800-38E (XTS-AES); RFC 9106 (Argon2)_

> [!WARNING] 
> **The `--sector-size 4096` Parameter Is Non-Negotiable**
> 
> Modern NVMe SSDs operate with 4096-byte (4K) physical sectors internally, even if they report 512-byte logical sectors to the OS for backward compatibility. If LUKS is configured with the default 512-byte sector size, every 4K physical write requires the SSD controller to:
> 
> A - Read the existing 4K block. 
> B - Modify the 512-byte slice within it. 
> C - Write the entire 4K block back.
> 
> This "Read-Modify-Write" amplification can degrade sequential write performance by up to 75% and significantly increases write amplification, reducing the SSD's lifespan. Setting `--sector-size 4096` ensures a 1:1 mapping between LUKS sectors and physical sectors, eliminating this overhead entirely.

### A - Encrypt the System Partition

```bash
cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --pbkdf argon2id \
    --sector-size 4096 \
    --label "cryptos" \
    /dev/nvme1n1p2
```

You will be prompted to type `YES` (in uppercase) to confirm, then to enter and verify your **System Passphrase**.

> [!TIP] 
> **Passphrase Strength Guidance**
> 
> The Argon2id PBKDF provides strong protection against brute-force attacks, but it cannot compensate for a weak passphrase. Aim for a minimum of 6 random words (Diceware method) or 20+ characters of mixed complexity. Avoid dictionary words, personal dates, or patterns.
> 
> You can verify the PBKDF parameters that `cryptsetup` chose automatically by running:
> ```bash
> cryptsetup luksDump /dev/nvme1n1p2 | grep -A 10 "PBKDF"
> ```

 Look for `Memory` (should be several hundred MB) and `Iterations`. Higher values mean stronger resistance to brute-force, at the cost of a slightly longer unlock time during boot.

### B - Encrypt the VM Partition

```bash
cryptsetup luksFormat --type luks2 \
    --cipher aes-xts-plain64 \
    --key-size 512 \
    --hash sha512 \
    --pbkdf argon2id \
    --sector-size 4096 \
    --label "cryptvms" \
    /dev/nvme1n1p3
```

Type `YES` and set your **VM Passphrase**.

> [!NOTE] 
> **Same or Different Passphrase?**
> 
> You may use the same passphrase for convenience or a different one for compartmentalization. Using a different passphrase means that unlocking your OS at boot does **not** automatically grant access to VM data — you would need to unlock `cryptvms` separately (manually or via a keyfile). This is relevant if:
> 
> - You store client data inside VMs and need to demonstrate access controls for compliance (GDPR, contractual obligations).
> - You want the ability to keep VMs locked during normal desktop use and only unlock them when actively working with virtual machines.
> 
> For most personal setups, using the same passphrase is perfectly acceptable.

### C - Open the Containers

Map the encrypted partitions to logical device names under `/dev/mapper/`:

```bash
cryptsetup open /dev/nvme1n1p2 cryptos
cryptsetup open /dev/nvme1n1p3 cryptvms
```

You will be prompted for the passphrase of each container. Upon success, the decrypted block devices become available at:

- `/dev/mapper/cryptos` — the system volume.
- `/dev/mapper/cryptvms` — the VM storage volume.

> [!NOTE] 
> **What Happens During `cryptsetup open`**
> 
> The `open` command does not decrypt the entire drive into RAM. It uses the provided passphrase to unlock the **master key** stored in the LUKS2 header, then creates a virtual block device (`/dev/mapper/cryptos`) that transparently encrypts and decrypts data on the fly as it is written to and read from the underlying physical partition. The performance impact on a CPU with AES-NI is negligible — typically less than 2% overhead on sequential I/O.

## Part III - Filesystem Creation

With the encrypted containers open and mapped, we now create the filesystems on top of them. Each filesystem is chosen to match its specific workload.

### A - Format the ESP (FAT32)

The EFI System Partition must be FAT32 per the UEFI specification:

```bash
mkfs.fat -F 32 -n EFI /dev/nvme1n1p1
```

> [!NOTE] 
> Notice that we format `nvme1n1p1` (the raw partition), **not** a mapper device. The ESP is unencrypted — the UEFI firmware must be able to read it directly before any OS or decryption layer is loaded. This is by design. The ESP contains only the bootloader and signed UKI images, not personal data.

### B - Format the System Volume (Btrfs)

```bash
mkfs.btrfs -L leoOs /dev/mapper/cryptos
```

> [!Abstract] 
> **Why Btrfs for the System Partition?**
> 
> Btrfs (B-tree File System) is selected here for three capabilities that are critical to this system's architecture:
> 
> **A - Instant Snapshots.** Btrfs snapshots are metadata-only operations that complete in milliseconds regardless of filesystem size. Before every system update, a snapshot of the root subvolume can be taken, providing an instant rollback point if an update breaks the system. Tools like `snapper` automate this.
> 
> **B - Transparent Compression.** Btrfs supports per-filesystem compression (we will use `zstd` at mount time). On an NVMe drive, the CPU can decompress data faster than the SSD can deliver uncompressed data, resulting in a net read speed _increase_. It also reduces write amplification, extending SSD lifespan.
> 
> **C - Subvolume Isolation.** Btrfs subvolumes act as independent filesystem trees within a single partition. We will create separate subvolumes for `/`, `/home`, `/var`, and snapshots in Chapter IV. This allows us to snapshot the root without including the home directory, or exclude volatile log data from backups.
> 
> _Reference: Arch Wiki — Btrfs ([https://wiki.archlinux.org/title/Btrfs](https://wiki.archlinux.org/title/Btrfs)); Kernel.org — Btrfs documentation_

### C - Format the VM Storage Volume (XFS)

```bash
mkfs.xfs -L leoVMs /dev/mapper/cryptvms
```

> [!TIP] 
> **Why XFS Instead of Btrfs for Virtual Machines?**
> 
> Btrfs uses Copy-on-Write (CoW) semantics: when a block is modified, Btrfs writes the new data to a _new_ location on disk and updates the metadata to point to it. The old block is retained until no snapshot references it.
> 
> This is excellent for system snapshots, but catastrophic for VM disk images. A QCOW2 or raw disk image undergoes thousands of random in-place writes per second during normal guest operation. Under CoW, each of these writes:
> 
> A - Allocates a new block elsewhere on disk. 
> B - Writes the modified data to the new location. 
> C - Updates the B-tree metadata. 
> D - Eventually frees the old block.
> 
> The result is severe fragmentation — a VM image that should be contiguous becomes scattered across the entire filesystem within hours of use. This can degrade VM I/O performance by 50–80%.
> 
> **XFS** uses a traditional in-place write model with extent-based allocation, making it the standard choice for large, frequently-modified files. Red Hat, which is the primary commercial backer of libvirt/KVM, uses XFS as the default filesystem for its virtualization storage pools.
> 
> _Reference: Red Hat Documentation — Recommended Filesystems for Virtualization; XFS.org_

> [!WARNING] 
> **Do Not Attempt to "Fix" CoW on Btrfs for VMs**
> 
> Some guides suggest using `chattr +C` (no-copy-on-write attribute) on a Btrfs directory to disable CoW for VM images. While this technically works, it also disables checksumming and snapshot inclusion for those files, negating the primary benefits of Btrfs. Using a dedicated XFS partition is the architecturally clean solution — it avoids the workaround entirely and provides the correct I/O semantics from the start.

# Chapter IV - Subvolume Layout & Mounting

## Part I - Create Btrfs Subvolumes

Before we can mount the system into its final hierarchy, we must create the Btrfs subvolume structure inside the encrypted system volume. Subvolumes are the mechanism by which we logically partition a single Btrfs filesystem into independently manageable trees — each can be snapshotted, mounted, or excluded from backups without affecting the others.

> [!Abstract] 
> **Subvolume Architecture Overview**
> 
> |Subvolume|Mount Point|Purpose|
> |---|---|---|
> |`@`|`/`|Root filesystem — the core OS (binaries, libraries, configuration).|
> |`@home`|`/home`|User data — documents, dotfiles, application state.|
> |`@var`|`/var`|Variable data — package caches, logs, databases. Excluded from root snapshots.|
> |`@snapshots`|`/.snapshots`|Snapshot storage directory (used by `snapper` or manual `btrfs subvolume snapshot`).|
> |`@swap`|`/swap`|Dedicated subvolume for the swapfile. No compression, no snapshots.|
> 
> The `@` prefix is a widely adopted naming convention (originating from Ubuntu and openSUSE) that visually distinguishes subvolumes from regular directories. It carries no technical meaning to Btrfs itself.

Mount the raw top-level Btrfs volume (subvolume ID 5) to a temporary location so we can create the subvolume structure:

```bash
mount /dev/mapper/cryptos /mnt
```

> [!NOTE] 
> **What Is the "Top-Level" Subvolume?**
> 
> Every Btrfs filesystem has an implicit root subvolume with ID 5 (also called the "top-level" or "default" subvolume). When you mount a Btrfs filesystem without specifying `subvol=`, you mount this top-level. We mount it now solely to create child subvolumes inside it — we will never mount the top-level directly in the final system. Instead, we mount `@` as `/`, which gives us the freedom to snapshot and replace the root without touching the top-level tree.

Create the subvolumes:

```bash
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@var
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@swap
```

> [!TIP] 
> **Why Separate `@var` From `@`?**
> 
> The `/var` directory contains package manager caches (`/var/cache/pacman/pkg/`), system logs (`/var/log/`), and temporary databases that change constantly. If `@var` were part of `@`, every root snapshot would include gigabytes of volatile cache data that is irrelevant to system state — inflating snapshot sizes and slowing down rollback operations.
> 
> By isolating `@var` into its own subvolume, a root snapshot captures only the OS state (installed packages, configuration files, system binaries) without the noise. This is the same principle Red Hat and SUSE apply in their enterprise Btrfs layouts.

> [!NOTE] 
> **Why a Dedicated `@swap` Subvolume?**
> 
> Btrfs imposes strict requirements on swapfiles: they must reside on a subvolume with Copy-on-Write disabled, they cannot be compressed, and they cannot be included in snapshots. A dedicated subvolume ensures these constraints are enforced structurally rather than relying on per-file attributes that could be accidentally reset.

Verify that all five subvolumes were created:

```bash
btrfs subvolume list /mnt
```

**Expected output** (IDs will vary):

```
ID 256 gen 8 top level 5 path @
ID 257 gen 9 top level 5 path @home
ID 258 gen 10 top level 5 path @var
ID 259 gen 11 top level 5 path @snapshots
ID 260 gen 12 top level 5 path @swap
```

All five subvolumes should appear with `top level 5`, confirming they are direct children of the Btrfs top-level volume.

Unmount the top-level — we are done with it:

```bash
umount /mnt
```

## Part II - Mount the Structure

We now mount each subvolume to its designated location in the filesystem hierarchy, applying performance-optimized mount flags tailored to NVMe SSDs.

> [!Abstract] 
> **Mount Flag Reference**
> 
> The following flags will be applied to all Btrfs mounts (with noted exceptions):
> 
> |Flag|Effect|Rationale|
> |---|---|---|
> |`noatime`|Disables access-time updates on file reads.|Without this, every `read()` syscall triggers a metadata _write_ to record the timestamp. On an SSD, this is pure write amplification with no practical benefit. `noatime` implicitly includes `nodiratime`.|
> |`compress=zstd`|Enables transparent Zstandard compression.|On NVMe drives, the CPU can decompress data faster than the SSD can deliver raw bytes, resulting in a net increase in effective read throughput. Additionally, compression reduces the total bytes written to flash, directly extending SSD lifespan. Zstd offers the best ratio of compression speed to compression ratio of any currently supported algorithm.|
> |`discard=async`|Queues TRIM (discard) commands asynchronously.|When files are deleted, the filesystem must inform the SSD that the underlying blocks are free (TRIM). Synchronous TRIM (`discard` without `async`) blocks the deletion I/O path until the SSD processes the TRIM, causing visible freezes during large deletions. Asynchronous mode batches TRIM commands and sends them during idle periods, eliminating this stall.|
> 
> _Reference: Arch Wiki — Btrfs#Mount options ([https://wiki.archlinux.org/title/Btrfs#Mount_options](https://wiki.archlinux.org/title/Btrfs#Mount_options)); Kernel.org — Btrfs mount options_

### A - Mount Root (`@`)

```bash
mount -o noatime,compress=zstd,discard=async,subvol=@ /dev/mapper/cryptos /mnt
```

> [!NOTE] 
> The `subvol=@` flag tells Btrfs to mount the `@` subvolume as the root of `/mnt` instead of the top-level volume. This is the cornerstone of the entire subvolume strategy — the kernel sees `@` as `/`, and all other subvolumes are mounted at paths within it.

### B - Create Mount Points

The directories must exist before we can mount subvolumes onto them:

```bash
mkdir -p /mnt/{home,var,boot,.snapshots,swap,vms}
```

> [!TIP] 
> The brace expansion `{home,var,boot,.snapshots,swap,vms}` is a Bash feature that expands into six separate `mkdir` arguments. The `-p` flag creates parent directories as needed and suppresses errors if a directory already exists.

### C - Mount the Remaining Subvolumes

```bash
mount -o noatime,compress=zstd,discard=async,subvol=@home /dev/mapper/cryptos /mnt/home
mount -o noatime,compress=zstd,discard=async,subvol=@var /dev/mapper/cryptos /mnt/var
mount -o noatime,compress=zstd,discard=async,subvol=@snapshots /dev/mapper/cryptos /mnt/.snapshots
mount -o noatime,subvol=@swap /dev/mapper/cryptos /mnt/swap
```

> [!WARNING] 
> **No Compression on `@swap`**
> 
> Notice that the `@swap` mount omits `compress=zstd`. This is deliberate and critical. Swap data is, by nature, memory pages that the kernel has already determined cannot remain in RAM. Attempting to compress swap introduces two problems:
> 
> **A - Deadlock Risk.** If the system is under memory pressure severe enough to trigger swapping, the `zstd` compression algorithm itself requires memory to operate. This creates a circular dependency: the kernel needs to free memory by writing to swap, but writing to swap requires memory for compression. The result can be a hard system deadlock.
> 
> **B - Futility.** Memory pages being swapped out are often already compressed (by `zswap` or `zram` if enabled) or contain high-entropy data (encrypted buffers, random state). Attempting to compress them again yields negligible size reduction at non-negligible CPU cost.
> 
> The `discard=async` flag is also omitted from the swap mount because the swapfile is a fixed-size, pre-allocated file — there are no blocks to discard.

### D - Mount the ESP (Boot)

```bash
mount -o noatime,nosuid,noexec,nodev,umask=0077 /dev/nvme1n1p1 /mnt/boot
```

> [!NOTE] 
> **ESP Security Flags Explained**
> 
> The ESP is a FAT32 partition readable by the UEFI firmware. It contains executable code (the bootloader, UKI images). We apply restrictive mount options to harden it at the OS level:
> 
> |Flag|Effect|
> |---|---|
> |`nosuid`|Ignores SUID/SGID bits on any file. Prevents privilege escalation via binaries placed on the ESP.|
> |`noexec`|Prevents direct execution of files from this partition by the Linux kernel. (UEFI firmware can still execute `.efi` files — this restriction applies only to the running OS.)|
> |`nodev`|Prevents interpretation of block/character special devices on this partition.|
> |`umask=0077`|Sets file permissions so that only `root` can read, write, or traverse the ESP contents. Other users see an empty directory.|
> 
> These flags do not affect the UEFI firmware's ability to load the bootloader or UKI images at pre-boot time. They protect the ESP only while Linux is running.

### E - Mount the VM Storage (XFS)

```bash
mount -o noatime,discard /dev/mapper/cryptvms /mnt/vms
```

> [!NOTE] 
> **XFS Mount Options**
> 
> XFS does not support `compress=zstd` — compression is a Btrfs-specific feature. The only performance-relevant flags here are:
> 
> - `noatime` — same rationale as above.
> - `discard` — XFS supports online discard (TRIM). Unlike Btrfs, XFS does not offer an `async` variant of discard at the mount option level; however, the XFS allocator batches extent frees internally, so the practical impact is similar.
> 
> The `/mnt/vms` path is a staging location. After installation, you can symlink or configure `libvirt` to point its default storage pool to `/vms` (or whichever path you prefer).

## Part III - Swap File Setup

We create a swapfile inside the dedicated `@swap` subvolume using the native Btrfs swapfile command, which handles all the internal requirements automatically.

> [!Abstract] 
> **Why a Swapfile Instead of a Swap Partition?**
> 
> On a Btrfs filesystem, a dedicated swap _partition_ would require repartitioning the drive and carving out space from the LUKS2 container — a rigid allocation that cannot be resized without reformatting. A swapfile inside a subvolume can be created, resized, or removed at will.
> 
> Historically, Btrfs did not support swapfiles at all (due to CoW complications). Since kernel 5.0, Btrfs natively supports swapfiles on subvolumes with CoW disabled. The `btrfs filesystem mkswapfile` command (introduced in `btrfs-progs` 6.1) automates the entire process: it creates the file, sets the `NOCOW` attribute, pre-allocates contiguous extents, and formats it with swap headers — all in one step.

### A - Create the Swapfile

For a system with 32 GB of RAM, 8 GB of swap is a reasonable baseline — enough to handle memory pressure spikes without dedicating excessive disk space:

```bash
btrfs filesystem mkswapfile --size 8g --uuid clear /mnt/swap/swapfile
```

> [!NOTE] 
> **Flag Breakdown**
> 
> |Flag|Purpose|
> |---|---|
> |`--size 8g`|Allocates an 8 GiB swapfile.|
> |`--uuid clear`|Clears the UUID field in the swap header. This prevents `genfstab` from generating a UUID-based fstab entry for the swapfile, which would be incorrect — swapfiles must be referenced by path, not UUID.|
> 
> This single command replaces the manual multi-step process (`truncate` → `chattr +C` → `fallocate` → `mkswap`) that older guides require.

> [!TIP] 
> **Swap Sizing Guidance**
> 
> |RAM|Swap (no hibernation)|Swap (with hibernation)|
> |---|---|---|
> |16 GB|4–8 GB|20 GB (RAM + margin)|
> |32 GB|8 GB|36 GB (RAM + margin)|
> |64 GB|8 GB|72 GB (RAM + margin)|
> 
> If you intend to use hibernation (suspend-to-disk) in the future, the swapfile must be at least as large as your total RAM plus a safety margin. Hibernation on Btrfs with LUKS requires additional configuration (resume offset calculation) that is beyond the scope of this chapter but is fully documented on the Arch Wiki.

### B - Activate the Swapfile

```bash
swapon /mnt/swap/swapfile
```

> [!NOTE] 
> This activates swap for the current live environment session. Permanent activation at boot will be handled via the `/etc/fstab` entry generated in Chapter V.

## Part IV - Verification

This is a critical checkpoint. Before proceeding to the base system installation, verify that every component of the filesystem hierarchy is correctly stacked.

```bash
lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINTS
```

**Expected output structure:**

|Device|Type|Mount Point(s)|
|---|---|---|
|`nvme1n1p1`|`vfat`|`/mnt/boot`|
|`nvme1n1p2` → `cryptos`|`crypto_LUKS` → `btrfs`|`/mnt`, `/mnt/home`, `/mnt/var`, `/mnt/.snapshots`, `/mnt/swap`|
|`nvme1n1p3` → `cryptvms`|`crypto_LUKS` → `xfs`|`/mnt/vms`|
|`[SWAP]`|`swap`|—|

> [!WARNING] 
> **Do Not Proceed If Any Mount Is Missing**
> 
> If `lsblk` does not show all five Btrfs mount points, the ESP mount, the XFS mount, **and** the swap entry, stop here and diagnose the issue. Running `pacstrap` (Chapter V) against an incomplete mount tree will scatter files across the wrong subvolumes, resulting in an unbootable system that must be rebuilt from scratch.

> [!TIP] 
> **Additional Verification Commands**
> 
> For a more detailed view, you can also inspect the mount table directly:
> 
> ```bash
> findmnt --target /mnt --real
> ```
> 
> This shows the full mount tree rooted at `/mnt`, including all mount options. Verify that:
> 
> - All Btrfs mounts show `compress=zstd` (except `@swap`).
> - The ESP shows `nosuid,noexec,nodev,umask=0077`.
> - The swap mount shows `noatime` and no `compress` flag.
> 
> You can also confirm the swap is active and correctly sized:
> 
> ```bash
> swapon --show
> ```
> 
> Expected output: one entry pointing to `/mnt/swap/swapfile` with `SIZE` of `8G` and `TYPE` of `file`.


# Chapter V - Base System Installation

## Part I - Select the Package Set

We now install the base system into the mounted hierarchy at `/mnt` using `pacstrap`. The package selection is not arbitrary — every package listed below addresses a specific requirement imposed by our hardware (ASUS ROG G16, Intel CPU, NVIDIA GPU), our filesystem architecture (Btrfs + LUKS2 + XFS), or our boot strategy (Unified Kernel Images with Secure Boot).

```bash
pacstrap -K /mnt \
    base base-devel \
    linux-firmware linux-lts linux-lts-headers \
    sof-firmware \
    intel-ucode \
    busybox \
    btrfs-progs xfsprogs \
    cryptsetup \
    networkmanager iwd \
    dracut systemd-ukify \
    sbctl sbsigntools \
    neovim git \
    man-db man-pages texinfo \
    nvidia-dkms nvidia-utils \
    power-profiles-daemon
```

> [!NOTE] 
> **The `-K` Flag**
> 
> The `-K` option initializes a fresh pacman keyring inside the target system (`/mnt`). Without it, `pacstrap` would copy the live ISO's keyring — which may be outdated if your ISO is not recent. A fresh keyring ensures all package signature verifications use current, trusted keys.

> [!Abstract] 
> **Package Selection Rationale**
> 
> Each package or group serves a precise role within the system architecture. The table below maps every entry to its function:
> 
> |Package(s)|Category|Role|
> |---|---|---|
> |`base`|Core|Minimal Arch Linux system — `systemd`, `glibc`, `bash`, `coreutils`, filesystem layout. This is the skeleton.|
> |`base-devel`|Core|Compiler toolchain (`gcc`, `make`, `patch`, `fakeroot`). Required for building AUR packages and DKMS kernel modules.|
> |`linux-firmware`|Firmware|Binary firmware blobs for hardware controllers — Wi-Fi (`iwlwifi` for Intel AX211), Bluetooth, GPU, audio codecs. Without this, most peripherals will not initialize.|
> |`linux-lts`, `linux-lts-headers`|Kernel|Long-Term Support kernel. Provides a stable, conservative baseline. The `-headers` package is mandatory for DKMS to compile out-of-tree modules (NVIDIA) against this kernel.|
> |`sof-firmware`|Firmware|Sound Open Firmware — required for the Intel audio DSP present on modern laptops. Without it, the ASUS G16's speakers and headphone jack will not produce sound.|
> |`intel-ucode`|Firmware|CPU microcode updates from Intel. Loaded at early boot to patch errata in the CPU's instruction decoder. Essential for stability and security on Intel platforms.|
> |`busybox`|Recovery|Provides a minimal rescue shell inside the initramfs. If the boot process fails before reaching the root filesystem, `busybox` gives you a shell with basic utilities (`ls`, `mount`, `cat`) to diagnose the issue.|
> |`btrfs-progs`|Filesystem|Userspace tools for Btrfs — `btrfs subvolume`, `btrfs filesystem`, `btrfs scrub`. Required both at boot (initramfs mounts Btrfs root) and at runtime (maintenance operations).|
> |`xfsprogs`|Filesystem|Userspace tools for XFS — `xfs_repair`, `xfs_info`, `xfs_growfs`. Required to manage the VM storage partition.|
> |`cryptsetup`|Encryption|LUKS2 management tool. **Mandatory** — the initramfs uses `cryptsetup` to unlock the encrypted root and VM containers during boot. Without it, the system cannot access its own drive.|
> |`networkmanager`|Networking|Full-featured network management daemon — handles Wi-Fi, Ethernet, VPN, and DNS resolution for the installed system. Replaces the live ISO's bare `iwd` setup.|
> |`iwd`|Networking|Retained as the Wi-Fi backend for NetworkManager. NetworkManager delegates all wireless operations (scanning, authentication, roaming) to `iwd`, combining NetworkManager's interface management with `iwd`'s superior wireless stack.|
> |`dracut`|Boot|Initramfs generator. Builds the initial RAM filesystem that runs before the real root is mounted. We use Dracut over `mkinitcpio` because it produces Unified Kernel Images natively and integrates cleanly with `systemd-ukify`.|
> |`systemd-ukify`|Boot|Tool for assembling Unified Kernel Images (UKIs) — single `.efi` files containing the kernel, initramfs, command line, and microcode. Used by Dracut during image generation.|
> |`sbctl`|Security|Secure Boot key manager — generates, enrolls, and manages custom Secure Boot signing keys. Allows us to sign our own UKIs so the UEFI firmware trusts them.|
> |`sbsigntools`|Security|Low-level EFI binary signing tools (`sbsign`, `sbverify`). Used by `sbctl` internally and available for manual signing operations.|
> |`neovim`|Editor|Modern terminal text editor. Used throughout this guide for configuration file editing.|
> |`git`|Tooling|Version control system. Required for cloning AUR repositories and general development workflow.|
> |`man-db`, `man-pages`, `texinfo`|Documentation|Offline manual pages and info documentation. Allows `man cryptsetup`, `man btrfs`, etc. to function without internet access.|
> |`nvidia-dkms`|Graphics|NVIDIA proprietary kernel module, built via DKMS. DKMS (Dynamic Kernel Module Support) automatically recompiles the NVIDIA module whenever a kernel is updated — critical since we run two kernels (`linux-lts` and later `linux-g14`).|
> |`nvidia-utils`|Graphics|NVIDIA userspace libraries — OpenGL, Vulkan, CUDA runtime. Required for graphical output on the discrete GPU.|
> |`power-profiles-daemon`|Power|Provides power profile switching (`balanced`, `power-saver`, `performance`) via D-Bus. Integrates with desktop environments and `asusctl` for laptop-specific power management.|

> [!TIP] 
> **Why Dracut Instead of mkinitcpio?**
> 
> Arch Linux ships `mkinitcpio` as its default initramfs generator. We deliberately replace it with `dracut` for the following reasons:
> 
> A - **Native UKI support.** Dracut can produce a Unified Kernel Image directly with `--uefi`, eliminating the need for a separate build step or wrapper script. With `mkinitcpio`, UKI generation requires additional tooling and manual configuration.
> 
> B - **`hostonly` mode.** Dracut can build an initramfs that contains _only_ the modules and drivers needed for the specific hardware it detects at build time. This produces smaller, faster-loading images compared to `mkinitcpio`'s more generic approach.
> 
> C - **Fedora/RHEL heritage.** Dracut is the standard initramfs generator for Fedora, RHEL, and SUSE — distributions with extensive enterprise LUKS and Btrfs deployment experience. Its `crypt` and `btrfs` modules are mature and well-tested.
> 
> _Reference: Arch Wiki — Dracut ([https://wiki.archlinux.org/title/Dracut](https://wiki.archlinux.org/title/Dracut))_

> [!WARNING] 
> **Do Not Install `mkinitcpio`**
> 
> The `base` package does not pull in `mkinitcpio` as a hard dependency (it is an optional dependency). Since we are using `dracut`, avoid installing `mkinitcpio` to prevent conflicting initramfs generation hooks. If both are present, kernel updates may trigger both generators, producing duplicate or incompatible images.

## Part II - Generate the Fstab

The `/etc/fstab` file (File System Table) is read by `systemd` at every boot to determine which filesystems to mount, where to mount them, and with which options. We generate it automatically from the currently active mount hierarchy.

### A - Generate the File

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

> [!NOTE] 
> **Why `-U` (UUIDs)?**
> 
> The `-U` flag tells `genfstab` to identify each filesystem by its UUID rather than its device path (e.g., `/dev/mapper/cryptos`). This is essential for reliability:
> 
> - Device paths like `/dev/nvme1n1p2` can change between boots if NVMe controllers are enumerated in a different order (e.g., after a BIOS update, adding a drive, or changing PCIe slot configurations).
> - Device-mapper names like `/dev/mapper/cryptos` depend on the `cryptsetup open` command being issued with the same name — a human convention, not a hardware guarantee.
> - UUIDs are embedded in the filesystem superblock and are immutable unless the filesystem is reformatted. They are the only truly stable identifier.

### B - Verify the Configuration

This is a mandatory checkpoint. An incorrect `fstab` will render the system unbootable.

```bash
cat /mnt/etc/fstab
```

Verify that every entry matches the expected structure:

|Mount Point|Filesystem Type|Key Mount Options|`subvol=`|
|---|---|---|---|
|`/`|`btrfs`|`noatime,compress=zstd,discard=async`|`subvol=/@`|
|`/home`|`btrfs`|`noatime,compress=zstd,discard=async`|`subvol=/@home`|
|`/var`|`btrfs`|`noatime,compress=zstd,discard=async`|`subvol=/@var`|
|`/.snapshots`|`btrfs`|`noatime,compress=zstd,discard=async`|`subvol=/@snapshots`|
|`/swap`|`btrfs`|`noatime`|`subvol=/@swap`|
|`/boot`|`vfat`|`noatime,nosuid,noexec,nodev,umask=0077`|—|
|`/vms`|`xfs`|`noatime,discard`|—|
|`/swap/swapfile`|`swap`|`defaults`|—|

> [!WARNING] 
> **Critical: Inspect the Swap Entry Path**
> 
> `genfstab` detects the active swapfile and records it. However, because we ran `swapon /mnt/swap/swapfile` from the live environment (where `/mnt` is the root), `genfstab` may record the path as:
> 
> ```
> /mnt/swap/swapfile    none    swap    defaults    0 0
> ```
> 
> This path is **wrong** for the installed system. When the system boots, its root is `/`, not `/mnt`. The correct entry must read:
> 
> ```
> /swap/swapfile    none    swap    defaults    0 0
> ```
> 
> If you do not correct this, swap will silently fail to activate at boot. The system will function — until memory pressure triggers an OOM-killer event rather than gracefully paging to swap.

### C - Correct the Swap Path

Open the fstab:

```bash
nvim /mnt/etc/fstab
```

Locate the swap entry. If the path begins with `/mnt/`, remove the `/mnt` prefix so the line reads:

```
/swap/swapfile    none    swap    defaults    0 0
```

Save and exit.

> [!TIP] 
> **Quick Verification After Editing**
> 
> After saving, run `cat /mnt/etc/fstab` one more time and visually confirm that:
> 
> A - No entry contains the string `/mnt/` anywhere in the mount point or device column.
> 
> B - All Btrfs entries reference `/dev/mapper/cryptos` (via UUID) — not `/dev/nvme1n1p2` (which is the encrypted container, not the filesystem inside it).
> 
> C - The `/boot` entry references `/dev/nvme1n1p1` (via UUID) — the unencrypted ESP partition, which the UEFI firmware must access before any LUKS container is unlocked.
> 
> D - The `/vms` entry references `/dev/mapper/cryptvms` (via UUID).

# Chapter VI - System Configuration (Chroot)

We now transition from the live ISO environment into the installed system. Every command from this point forward writes directly to the NVMe drive — we are configuring the system that will persist after reboot.

## Part I - Enter the Environment

```bash
arch-chroot /mnt
```

> [!Abstract] 
> **What `arch-chroot` Does**
> 
> The `arch-chroot` wrapper (provided by `arch-install-scripts`) performs two operations:
> 
> A — **Bind-mounts** the virtual kernel filesystems (`/proc`, `/sys`, `/dev`, `/run`) from the live environment into the target tree at `/mnt`. These pseudo-filesystems provide access to hardware information, kernel interfaces, and the device tree — without them, commands like `hwclock`, `pacman`, and `systemctl` cannot function inside the chroot.
> 
> B — **Executes `chroot /mnt /bin/bash`**, changing the apparent root directory to `/mnt`. From the shell's perspective, `/mnt` becomes `/`. All path references — `/etc/locale.gen`, `/boot/EFI/Linux/` — now resolve relative to the installed system on the NVMe drive, not the USB stick.
> 
> _Reference: Arch Wiki — Chroot ([https://wiki.archlinux.org/title/Chroot](https://wiki.archlinux.org/title/Chroot))_

**Prompt verification:** Your shell prompt should change to `[root@archiso /]#`. If it still shows the live ISO's hostname without the path change, the chroot did not succeed — verify that `/mnt` is correctly mounted with `findmnt /mnt` before retrying.

## Part II - System Identity and Localization

### A - Set Timezone

Create a symbolic link from the appropriate zoneinfo file to `/etc/localtime`, then synchronize the hardware clock:

```bash
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

> [!NOTE] 
> **Two Clocks, Two Standards**
> 
> Your machine maintains two independent clocks:
> 
| Clock                | Location                                | Persistence                                                                            |
| -------------------- | --------------------------------------- | -------------------------------------------------------------------------------------- |
| System clock         | Kernel memory                           | Volatile — lost on power-off. Set from hardware clock at boot, then maintained by NTP. |
| Hardware clock (RTC) | CMOS battery-backed chip on motherboard | Persistent — survives power-off.                                                       |
> 
> `ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime` tells the OS how to interpret the hardware clock's stored time (UTC) into your local timezone.
> 
> `hwclock --systohc` writes the current system time (which was NTP-synchronized in Chapter II) back to the hardware clock. The `--systohc` flag means "system clock → hardware clock." This ensures the RTC is accurate even if the CMOS battery drifts.
> 
> **Linux convention:** The hardware clock stores UTC. The timezone symlink handles the conversion to local time. This is the correct configuration for a Linux-only machine. If you dual-boot with Windows (which expects the RTC to store local time), you would need to either configure Windows to use UTC or set `timedatectl set-local-rtc 1` — but this is not relevant to our single-boot Arch installation.

### B - Localization

Locales define the system's language, character encoding, date formats, and collation rules. We must generate at least the English UTF-8 locale (required by many system tools) and optionally the French locale.

```bash
nvim /etc/locale.gen
```

Locate and uncomment (remove the leading `#`) the following lines:

```
en_US.UTF-8 UTF-8
fr_FR.UTF-8 UTF-8
```

Save and exit, then generate the compiled locale data:

```bash
locale-gen
```

Set the default system language:

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

> [!NOTE] 
> **`LANG` vs. `LC_*` Variables**
> 
> Setting `LANG=en_US.UTF-8` establishes the default for all locale categories — messages, time display, number formatting, collation, etc. Individual categories can be overridden with specific `LC_*` variables (e.g., `LC_TIME=fr_FR.UTF-8` to display dates in French while keeping English system messages). For now, a single `LANG` entry is sufficient. Per-category overrides can be added to `/etc/locale.conf` later if needed.

### C - Network Identity

Set the machine's hostname:

```bash
echo "zephyrus-g16" > /etc/hostname
```

Create the static hosts file for local name resolution:

```bash
nvim /etc/hosts
```

Enter the following content:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   zephyrus-g16.localdomain zephyrus-g16
```

> [!NOTE] 
> **The Three Entries Explained**
> 
| Entry                                             | Purpose                                                                                                                                                                                                                                            |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `127.0.0.1 localhost`                             | IPv4 loopback resolution. Required by virtually every network-aware application.                                                                                                                                                                   |
| `::1 localhost`                                   | IPv6 loopback resolution. The IPv6 equivalent of the line above.                                                                                                                                                                                   |
| `127.0.1.1 zephyrus-g16.localdomain zephyrus-g16` | Maps the machine's own hostname to a loopback address. This allows programs that call `gethostbyname()` to resolve the local hostname without requiring a DNS server or mDNS daemon. The `.localdomain` suffix is a conventional FQDN placeholder. |
> _Reference: Arch Wiki — Network configuration, section "Local hostname resolution" ([https://wiki.archlinux.org/title/Network_configuration#Local_hostname_resolution](https://wiki.archlinux.org/title/Network_configuration#Local_hostname_resolution))_

## Part III - Security and User Setup

### A - Set Root Password

```bash
passwd
```

Choose a strong, unique passphrase for the root account. This is the unrestricted administrative identity — compromise of this password grants total control over the system.

> [!WARNING] 
> **Root Password vs. LUKS Passphrase**
> 
> These are separate credentials with different threat models. The LUKS passphrase protects data at rest (the drive is unreadable without it). The root password protects the running system (prevents privilege escalation from a user session). Use different passphrases for each. If an attacker obtains your root password via a compromised application, your LUKS encryption remains intact — and vice versa.

### B - Create Your Daily User

```bash
useradd -m -G wheel -s /bin/bash username
passwd username
```

> [!NOTE] 
> **Flag Breakdown**
> 
| Flag           | Effect                                                                                                                       |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `-m`           | Creates the home directory `/home/username` and populates it from `/etc/skel` (default dotfiles).                            |
| `-G wheel`     | Adds the user to the `wheel` group. This group is the conventional Unix mechanism for granting administrative (sudo) access. |
| `-s /bin/bash` | Sets the login shell to Bash.                                                                                                |

### C - Grant Sudo Rights

```bash
EDITOR=nvim visudo
```

Locate the following line (it is commented out by default):

```
# %wheel ALL=(ALL:ALL) ALL
```

Remove the leading `#` and space so it reads:

```
%wheel ALL=(ALL:ALL) ALL
```

Save and exit.

> [!WARNING] 
> **Always Use `visudo`**
> 
> Never edit `/etc/sudoers` directly with a text editor. `visudo` performs a syntax check before saving — if you introduce a syntax error in the sudoers file (a misplaced character, a missing newline), `sudo` will refuse to function entirely, locking you out of administrative access. `visudo` catches these errors before they are written to disk.

## Part IV - NetworkManager and iwd Backend

NetworkManager will be the primary network management daemon on the installed system. By default, NetworkManager uses its internal `wpa_supplicant`-based Wi-Fi backend. We override this to use `iwd`, which offers faster scanning, more reliable roaming, and lower memory footprint.

```bash
nvim /etc/NetworkManager/conf.d/wifi_backend.conf
```

Enter the following content:

```
[device]
wifi.backend=iwd
EOF
```

> [!NOTE] 
> **Why `iwd` Over `wpa_supplicant`?**
> 
| Aspect           | `wpa_supplicant`              | `iwd`                                 |
| ---------------- | ----------------------------- | ------------------------------------- |
| Memory usage     | ~4 MB resident                | ~1.5 MB resident                      |
| Scan speed       | Standard                      | Faster (uses `nl80211` directly)      |
| Configuration    | External config files + D-Bus | Internal state + D-Bus                |
| WPA3/SAE support | Requires external libraries   | Native                                |
| Maintained by    | Jouni Malinen (individual)    | Intel (corporate, active development) |
> Since we already installed `iwd` in the `pacstrap` step and the Intel AX211 chipset in your G16 is developed by the same team that maintains `iwd`, this is the natural fit.
> 
> _Reference: Arch Wiki — NetworkManager, section "Using iwd as the Wi-Fi backend" ([https://wiki.archlinux.org/title/NetworkManager#Using_iwd_as_the_Wi-Fi_backend](https://wiki.archlinux.org/title/NetworkManager#Using_iwd_as_the_Wi-Fi_backend))_

## Part V - G14 Repository and ASUS Kernel

The `asus-linux.org` community maintains a custom Arch repository (named `g14`, for historical reasons — it originated with the Zephyrus G14 but now covers the full ROG lineup including your G16). This repository provides kernel builds patched for ASUS-specific hardware (hotkeys, fan curves, screen panel support) and userspace tools for hardware control.

### A - Import and Trust the Repository GPG Key

```bash
pacman-key --recv-keys 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --finger 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
pacman-key --lsign-key 8F654886F17D497FEFE3DB448B15A6B0E9A3FA35
```

> [!NOTE] 
> **Key Verification Procedure**
> 
> The three commands perform distinct trust operations:
> 
| Command       | Action                                                                                                                                                                                                         |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--recv-keys` | Downloads the public key from a keyserver into your local pacman keyring.                                                                                                                                      |
| `--finger`    | Displays the key's fingerprint for visual verification. Confirm that the full fingerprint matches `8F654886F17D497FEFE3DB448B15A6B0E9A3FA35` — this is the canonical identifier published on `asus-linux.org`. |
| `--lsign-key` | Locally signs the key, marking it as trusted. Without this step, pacman would reject packages signed by this key with a "marginal trust" or "unknown trust" error.                                             |
> 
> The second `--finger` invocation in the original guide is redundant (it re-displays the same fingerprint). It serves as a visual confirmation step — you may run it or skip it.

### B - Add the Repository to pacman.conf

```bash
nvim /etc/pacman.conf
```

Enter the following content:

```
[g14]
Server = https://arch.asus-linux.org
EOF
```

> [!WARNING] 
> **Repository Ordering Matters**
> 
> The `[g14]` block is appended to the end of `pacman.conf`, which means it has the **lowest priority**. If a package exists in both the official Arch repositories and `[g14]`, the official version is used. This is the correct behavior — we only want the G14 repository for packages that do not exist in the official repos (`linux-g14`, `asusctl`, `supergfxctl`). If you placed `[g14]` above `[core]`, it could shadow official packages with custom builds, creating upgrade conflicts.

### C - Synchronize Package Databases

```bash
pacman -Syu
```

> [!NOTE] 
> **`-Syu` vs. `-Suy` vs. `-Sy`**
> 
> A - `pacman -Sy` synchronizes the package databases (downloads the latest package lists from all configured repositories, including the newly added `[g14]`).
> 
> B - `pacman -Syu` synchronizes **and** upgrades all installed packages to their latest versions. This is the correct form. Running `-Sy` alone without `-u` creates a partial-sync state where your local database knows about newer packages but nothing has been upgraded — installing a new package in this state can cause dependency breakage.
> 
> Always use `-Syu` when synchronizing databases.
> 
> _Reference: Arch Wiki — Pacman, section "Partial upgrades are unsupported" ([https://wiki.archlinux.org/title/System_maintenance#Partial_upgrades_are_unsupported](https://wiki.archlinux.org/title/System_maintenance#Partial_upgrades_are_unsupported))_

### D - Install the G14 Kernel and ASUS Tools

```bash
pacman -S linux-g14 linux-g14-headers asusctl supergfxctl
```

> [!NOTE] 
> **Package Roles**
> 
| Package             | Role                                                                                                                                                                                                                                                                       |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `linux-g14`         | Kernel build patched for ASUS ROG hardware. Includes driver fixes for screen panels, hotkey mappings, and fan control interfaces that have not yet been upstreamed to the mainline kernel.                                                                                 |
| `linux-g14-headers` | Kernel headers matching `linux-g14`. **Required** for DKMS to compile the `nvidia-dkms` module against this kernel. Without headers, DKMS silently skips the build and the NVIDIA GPU will not function under the G14 kernel.                                              |
| `asusctl`           | Userspace daemon and CLI tool for ASUS-specific hardware control — keyboard RGB profiles, fan curve management, charge limit thresholds, performance mode switching (Silent / Balanced / Turbo).                                                                           |
| `supergfxctl`       | Graphics mode switching daemon for hybrid NVIDIA/Intel GPU laptops. Manages transitions between `Integrated` (Intel only, maximum battery life), `Hybrid` (NVIDIA on-demand via PRIME), `Dedicated` (NVIDIA only), and `Compute` (NVIDIA for CUDA without display output). |

> [!Abstract] 
> **Dual-Kernel Strategy**
> 
> You now have two kernels installed:
> 
|Kernel|Purpose|Stability|
|---|---|---|
|`linux-lts`|Long-Term Support — conservative, backport-only updates.|High — recommended as default boot target during early setup.|
|`linux-g14`|ASUS ROG-optimized — latest hardware patches, more frequent updates.|Moderate — better hardware support but may occasionally introduce regressions.|
Both kernels will receive their own Unified Kernel Image (UKI) in Chapter VII. The bootloader (systemd-boot) will present both options at boot. **Use `linux-lts` as your fallback** — if a `linux-g14` update causes a regression (e.g., a suspend/resume failure, a GPU driver incompatibility), you can boot the LTS kernel immediately without any repair procedure.

# Chapter VII - Initramfs and Unified Kernel Image (Dracut)

This chapter constructs the boot payload — the Unified Kernel Image (UKI) that the UEFI firmware will execute. This is the most architecturally critical chapter. An error here does not corrupt data, but it renders the system unbootable.

> [!Abstract] 
> **What Is a Unified Kernel Image?**
> 
> A traditional Linux boot sequence involves multiple separate files: the bootloader loads a kernel (`vmlinuz`), an initial ramdisk (`initramfs`), a microcode archive (`intel-ucode.img`), and reads kernel parameters from a configuration file. Each file is stored independently on the ESP, and the bootloader must be told where to find each one.
> 
> A Unified Kernel Image (UKI) collapses all of these into a **single PE (Portable Executable) binary** — the same `.efi` format that UEFI firmware natively understands. One file contains:
> 
> |Component|Role|
> |---|---|
> |Linux kernel|The operating system core.|
> |Initramfs|Early userspace — unlocks LUKS, assembles Btrfs, finds root.|
> |CPU microcode|Intel errata patches, loaded before anything else.|
> |Kernel command line|Boot parameters (`root=`, `rd.luks.uuid=`, etc.), embedded and immutable.|
> |OS release metadata|Identification strings for the bootloader menu.|
> 
> The UEFI firmware can execute this `.efi` file directly. `systemd-boot` discovers UKIs automatically when placed in `/boot/EFI/Linux/`.
> 
> **Security implication:** Because the kernel command line is embedded inside the signed binary, an attacker with physical access to the ESP cannot modify boot parameters (e.g., appending `init=/bin/bash` to bypass authentication). This is a core advantage of UKIs over traditional bootloader configurations.
> 
> _Reference: systemd documentation — Unified Kernel Image ([https://uapi-group.org/specifications/specs/unified_kernel_image/](https://uapi-group.org/specifications/specs/unified_kernel_image/))_

## Part I - Create Optimized Dracut Configurations

Dracut is the initramfs generator. It assembles the early userspace environment that runs before the real root filesystem is available. We configure it through drop-in files in `/etc/dracut.conf.d/`, numbered to control processing order.

### A - Optimization and UEFI Settings

```bash
install -Dm0644 /dev/stdin /etc/dracut.conf.d/00-optimization.conf <<'EOF'
hostonly="yes"
hostonly_mode="strict"
compress="zstd"
uefi="yes"
early_microcode="yes"
EOF
```

> [!NOTE] 
> **Directive Breakdown**
> 
> |Directive|Effect|
> |---|---|
> |`hostonly="yes"`|Builds an initramfs containing **only** the drivers and modules needed by this specific machine. A generic initramfs includes drivers for every possible storage controller, filesystem, and USB chipset — producing a 50+ MB image. Host-only mode produces a 15–25 MB image by interrogating the currently running hardware and including only what is detected.|
> |`hostonly_mode="strict"`|Tightens the host-only filter. In `sloppy` mode (the default when `hostonly="yes"`), dracut includes modules that _might_ be needed. In `strict` mode, it includes only modules that are _provably_ required based on the current system state. This eliminates residual bloat from optional bus drivers and filesystem handlers.|
> |`compress="zstd"`|Compresses the initramfs archive using Zstandard. On NVMe storage, decompression speed is the bottleneck (not I/O bandwidth), and `zstd` offers the fastest decompression among modern algorithms — roughly 1.5 GB/s on a modern Intel CPU. This directly reduces boot time.|
> |`uefi="yes"`|Instructs dracut to produce a Unified Kernel Image (`.efi` PE binary) rather than a standalone `initramfs.img`. This is the flag that triggers UKI generation.|
> |`early_microcode="yes"`|Prepends the Intel microcode archive (`intel-ucode`) to the initramfs so it is loaded by the CPU before any kernel code executes. This ensures errata patches are active from the earliest possible moment.|
> 
> _Reference: dracut.conf(5) man page_

### B - Module Selection

```bash
install -Dm0644 /dev/stdin /etc/dracut.conf.d/10-modules.conf <<'EOF'
add_dracutmodules+=" crypt btrfs systemd busybox "
omit_dracutmodules+=" network cifs nfs nbd mdraid brltty "
EOF
```

> [!NOTE] 
> **Module Rationale**
> 
> |Action|Modules|Justification|
> |---|---|---|
> |**Add**|`crypt`|Provides the LUKS unlock prompt at boot. Without this, dracut cannot open `/dev/nvme1n1p2` and the root filesystem is inaccessible.|
> ||`btrfs`|Loads `btrfs.ko` and userspace tools needed to mount Btrfs subvolumes. Required for `rootflags=subvol=@` to function.|
> ||`systemd`|Uses `systemd` as the init framework inside the initramfs (rather than the shell-based fallback). Provides `systemd-cryptsetup` for password prompting, parallel service startup, and structured logging.|
> ||`busybox`|Embeds a BusyBox recovery shell. If the boot process fails (wrong UUID, missing driver), you drop into this shell to diagnose the problem. Without it, a failed boot produces a kernel panic with no interactive recovery option.|
> |**Omit**|`network`|Network-based root filesystems (iSCSI, NFS root, PXE). Our root is a local NVMe drive. Including this module adds `dhclient`, DNS resolution, and network interface configuration to the initramfs — all unnecessary weight.|
> ||`cifs`, `nfs`, `nbd`|Network filesystem clients. Same reasoning — no network storage is involved in the boot process.|
> ||`mdraid`|Linux software RAID (`md`). Our storage uses LUKS+Btrfs, not RAID arrays.|
> ||`brltty`|Braille TTY display driver. Unless you use a Braille terminal, this module adds a udev rule that has been documented to introduce a multi-second delay during device enumeration. Omitting it is a common boot speed optimization.|

### C - Graphics Drivers (Early KMS)

```bash
install -Dm0644 /dev/stdin /etc/dracut.conf.d/20-graphics.conf <<'EOF'
force_drivers+=" nvidia nvidia_modeset nvidia_uvm nvidia_drm "
EOF
```

> [!NOTE] 
> **Why Force-Load NVIDIA Modules in the Initramfs?**
> 
> By default, NVIDIA kernel modules are loaded late in the boot process — after the display manager starts. This creates a visible "mode switch": the screen flickers, goes black, changes resolution, and then presents the login screen. This occurs because the framebuffer transitions from the basic EFI GOP (Generic Output Protocol) driver to the NVIDIA DRM driver mid-boot.
> 
> `force_drivers` embeds these modules into the initramfs and loads them during early boot, before the root filesystem is even mounted. The NVIDIA DRM driver takes control of the display immediately, eliminating the visible transition. The four modules form a dependency chain:
> 
> |Module|Function|
> |---|---|
> |`nvidia`|Core GPU driver — hardware initialization, memory management, power control.|
> |`nvidia_modeset`|Kernel Mode Setting — controls display resolution, refresh rate, and output routing at the kernel level.|
> |`nvidia_uvm`|Unified Virtual Memory — enables GPU and CPU to share address spaces (required for CUDA).|
> |`nvidia_drm`|DRM (Direct Rendering Manager) integration — exposes the GPU to the Linux graphics stack (`/dev/dri/`).|

## Part II - The Kernel Command Line

With a traditional bootloader, kernel parameters are stored in a text file (e.g., `/boot/loader/entries/*.conf` or `/boot/grub/grub.cfg`) and can be edited at boot time via a menu. With Unified Kernel Images, the kernel command line is **embedded inside the signed `.efi` binary at build time**. It cannot be modified at boot without rebuilding the UKI.

This means the parameters must be correct before generation. An error here means the system will not boot — but it also means an attacker cannot tamper with them.

### A - Retrieve the Required UUIDs

We need two distinct UUIDs. These are not the same value:

|UUID|Source Device|What It Identifies|
|---|---|---|
|**LUKS UUID**|`/dev/nvme1n1p2`|The encrypted LUKS2 container itself — the "outer shell." Dracut uses this to know which device to present a password prompt for.|
|**Root UUID**|`/dev/mapper/cryptos`|The Btrfs filesystem inside the unlocked container — the "inner content." The kernel uses this to locate and mount the root filesystem after LUKS is opened.|

Retrieve both:

```bash
echo "LUKS UUID (encrypted container on p2):"
blkid -s UUID -o value /dev/nvme1n1p2

echo "ROOT UUID (Btrfs filesystem inside cryptos):"
blkid -s UUID -o value /dev/mapper/cryptos
```

> [!WARNING] 
> **Copy These Values Exactly**
> 
> A single transposed character in either UUID will result in an unbootable system. Dracut will either fail to find the LUKS container (no password prompt appears → kernel panic) or fail to find the root filesystem inside it (password prompt succeeds, then kernel panic with `VFS: Unable to mount root fs`).
> 
> If you are working over SSH or a second terminal, copy-paste directly. If you are working on a single TTY, write both values down and double-check each character.

### B - Create the Command Line Configuration

```bash
nvim /etc/dracut.conf.d/30-cmdline.conf
```

Enter the following, replacing the placeholder UUIDs with your actual values:

```
kernel_cmdline="rd.luks.uuid=YOUR-LUKS-UUID-HERE root=UUID=YOUR-ROOT-UUID-HERE rootflags=subvol=@ rw quiet nvidia_drm.modeset=1"
```

> [!NOTE] 
> **Parameter Explanation**
> 
> |Parameter|Function|
> |---|---|
> |`rd.luks.uuid=<LUKS-UUID>`|Instructs the dracut `crypt` module: "At boot, scan for a LUKS container with this UUID and prompt the user to unlock it." The `rd.` prefix denotes a parameter consumed by the initramfs (ramdisk), not the final kernel.|
> |`root=UUID=<ROOT-UUID>`|Tells the kernel: "After the initramfs has done its work, mount the filesystem with this UUID as `/`." This is the Btrfs filesystem UUID inside the unlocked LUKS container.|
> |`rootflags=subvol=@`|Passes mount options to the root filesystem mount. This tells Btrfs to mount the `@` subvolume as root, rather than the top-level subvolume (ID 5). Without this, the system would mount the empty top-level and fail to find `/bin`, `/etc`, or any system files.|
> |`rw`|Mounts root as read-write immediately. The alternative (`ro`) mounts read-only initially, then remounts read-write later via an initscript. `rw` is simpler and avoids edge cases with systemd's remount logic on Btrfs.|
> |`quiet`|Suppresses most kernel log messages during boot, producing a clean visual experience. Remove this parameter temporarily if you need to debug boot failures — it will expose all kernel messages on the console.|
> |`nvidia_drm.modeset=1`|Enables kernel mode setting for the NVIDIA DRM driver. **Mandatory** for Wayland compositors, for proper `GBM` buffer allocation, and for flicker-free boot when combined with early-loaded NVIDIA modules. Without this, the NVIDIA driver operates in a legacy user-space mode setting path that is incompatible with modern display servers.|

## Part III - Generate the Unified Kernel Images

Create the target directory on the ESP where `systemd-boot` expects to find UKI files:

```bash
mkdir -p /boot/EFI/Linux
```

Generate the UKI for each installed kernel. The `--kver` flag specifies which kernel's modules to bundle:

```bash
dracut --force --uefi --kver $(ls /usr/lib/modules/ | grep lts)
dracut --force --uefi --kver $(ls /usr/lib/modules/ | grep g14)
```

> [!NOTE] 
> **Command Breakdown**
> 
> |Flag|Effect|
> |---|---|
> |`--force`|Overwrites any existing UKI for this kernel version. Without `--force`, dracut refuses to regenerate if a file already exists.|
> |`--uefi`|Redundant with `uefi="yes"` in `00-optimization.conf`, but serves as an explicit safety net on the command line.|
> |`--kver <version>`|Specifies the kernel version string. This determines which directory under `/usr/lib/modules/` dracut reads for kernel modules.|
> 
> The subshell `$(ls /usr/lib/modules/ | grep lts)` dynamically resolves the full version string (e.g., `6.12.10-1-lts`). Similarly, `grep g14` resolves the G14 kernel version. This avoids hardcoding version numbers that change with every kernel update.

> [!WARNING] 
> **DKMS Must Have Completed Before This Step**
> 
> When you installed `nvidia-dkms` in Chapter V and `linux-g14`/`linux-g14-headers` in Chapter VI, DKMS should have automatically compiled the NVIDIA modules for both kernels. Dracut's `force_drivers` directive in `20-graphics.conf` will attempt to embed these compiled modules into the UKI. If DKMS failed silently (e.g., due to missing headers), the NVIDIA modules will be absent from the initramfs and the GPU will not initialize at boot.
> 
> Verify DKMS status before generating UKIs:
> 
> ```bash
> dkms status
> ```
> 
> Expected output should show `nvidia` with status `installed` for **both** kernel versions. If either shows `broken` or is missing, resolve the DKMS build first:
> 
> ```bash
> dkms autoinstall --kernelver $(ls /usr/lib/modules/ | grep lts)
> dkms autoinstall --kernelver $(ls /usr/lib/modules/ | grep g14)
> ```

Verify the generated UKIs:

```bash
ls -lh /boot/EFI/Linux/
```

You should see two `.efi` files, each approximately 60–100 MB in size. The filename typically follows the pattern `linux-<version>.efi`.

> [!TIP] 
> **Inspecting UKI Contents**
> 
> If you want to verify what was embedded inside a UKI (kernel version, command line, included modules), you can extract the embedded command line using:
> 
> ```bash
> objdump -s -j .cmdline /boot/EFI/Linux/*.efi
> ```
> 
> This reads the `.cmdline` PE section from the binary. Verify that your LUKS UUID and root UUID appear correctly. If they show the placeholder text rather than actual UUIDs, you must correct `/etc/dracut.conf.d/30-cmdline.conf` and regenerate.

# Chapter VIII - Bootloader and Secure Boot

This chapter installs the mechanism that bridges UEFI firmware to your Unified Kernel Images, then establishes a cryptographic chain of trust ensuring that only signed, unmodified code executes during boot.

## Part I - Install systemd-boot

`systemd-boot` is a minimal UEFI boot manager — not a full bootloader in the GRUB sense. It does not understand filesystems, does not parse configuration files with scripting logic, and does not load kernels from arbitrary partitions. Its sole function is to present a menu of `.efi` executables found on the ESP and hand off execution to the one you select. This simplicity is an asset: less code means a smaller attack surface and fewer failure modes.

> [!Abstract] 
> **systemd-boot vs GRUB**
> 
> |Aspect|systemd-boot|GRUB|
> |---|---|---|
> |Complexity|~120 KB binary. Menu-only.|Multi-megabyte. Includes filesystem drivers, scripting language, network stack.|
> |UKI support|Native — auto-discovers `.efi` files in `ESP/EFI/Linux/`.|Requires manual `chainloader` entries or `grub-mkconfig` hooks.|
> |Secure Boot|Signs one small binary.|Must sign GRUB binary, modules, and configuration. Larger trust surface.|
> |LUKS / Btrfs awareness|None needed — the UKI handles everything internally.|Can read LUKS/Btrfs partitions directly (which means GRUB itself contains crypto and filesystem code that must be kept updated).|
> |Configuration|Single `loader.conf` file plus auto-discovery.|`grub.cfg` generated by `grub-mkconfig` — complex, error-prone templating system.|
> 
> For a UKI-based boot chain, `systemd-boot` is the architecturally correct choice. GRUB's additional capabilities (reading non-FAT filesystems, booting legacy BIOS systems) are irrelevant to our UEFI-only design.
> 
> _Reference: Arch Wiki — systemd-boot ([https://wiki.archlinux.org/title/systemd-boot](https://wiki.archlinux.org/title/systemd-boot))_

### A - Install the Boot Manager

```bash
bootctl install
```

> [!NOTE] 
> **What `bootctl install` Does**
> 
> This command performs three operations:
> 
> A - Copies the `systemd-boot` EFI binary to two locations on the ESP:
> 
> |Destination|Purpose|
> |---|---|
> |`/boot/EFI/systemd/systemd-bootx64.efi`|The primary boot manager binary.|
> |`/boot/EFI/BOOT/BOOTX64.EFI`|The UEFI "fallback" path. If the firmware's NVRAM boot entries are lost (e.g., after a CMOS reset), UEFI firmware will look for this hardcoded path as a last resort. Placing systemd-boot here ensures the system remains bootable even after NVRAM corruption.|
> 
> B - Creates a UEFI NVRAM boot entry (visible via `efibootmgr`) named "Linux Boot Manager" pointing to the primary binary.
> 
> C - Creates the directory structure `/boot/loader/` for configuration and `/boot/EFI/Linux/` for UKI auto-discovery (if not already present).

### B - Configure the Loader

```bash
nvim /boot/loader/loader.conf
```

Content:

```
timeout 3
console-mode max
editor  no
```

> [!NOTE] 
> **Directive Explanation**
> 
> |Directive|Value|Effect|
> |---|---|---|
> |`timeout`|`10`|Displays the boot menu for 10 seconds before auto-selecting the default entry. Provides enough time to select the alternate kernel (LTS vs G14) without adding unnecessary delay to routine boots. Set to `0` later if you want instant boot with menu access via holding the spacebar.|
> |`console-mode`|`max`|Sets the UEFI console to its highest available resolution. On modern displays (2560x1600 on the G16), this produces readable text rather than the enormous default font.|
> |`editor`|`no`|**See warning below.**|

> [!WARNING] 
> **`editor` Must Be `no` — Not `yes`**
> 
> When `editor yes` is set, anyone with physical access to the machine can press `e` at the boot menu and modify kernel parameters on the fly — for example, appending `init=/bin/bash` to bypass all authentication and drop directly into a root shell. This completely defeats the purpose of:
> 
> A - LUKS encryption (the attacker does not need the password if they can bypass `init`).
> 
> B - Secure Boot (the parameters are modified after the signed binary is loaded but before the kernel processes them — `systemd-boot` itself modifies the command line it passes to the stub).
> 
> C - Your root password and user authentication (the system never reaches a login prompt).
> 
> With UKIs, the command line is embedded inside the signed binary. Setting `editor no` enforces this immutability at the boot manager level. There is no legitimate operational reason to allow runtime parameter editing on a Secure Boot system.
> 
> _Reference: systemd-boot(7) — "editor" option security considerations_

## Part II - Secure Boot Setup

Secure Boot is a UEFI specification feature that verifies the cryptographic signature of every executable loaded during the boot process. If any binary (boot manager, UKI, driver) is unsigned or signed with an untrusted key, the firmware refuses to execute it.

We will enroll our own Platform Owner (PO) keys, sign all boot components, and configure the firmware to enforce signature verification.

> [!Abstract] 
> **The Secure Boot Trust Chain**
> 
> UEFI Secure Boot uses a hierarchy of cryptographic keys stored in firmware NVRAM:
> 
> |Key Database|Name|Role|
> |---|---|---|
> |PK|Platform Key|The "master key." Owned by the platform owner (you, not the OEM). Authorizes changes to KEK. Only one PK exists.|
> |KEK|Key Exchange Key|Authorizes changes to the `db` and `dbx` databases. Multiple KEKs can coexist (yours + Microsoft's).|
> |db|Signature Database|Contains the public keys or hashes of binaries that are **allowed** to execute. Your signing key goes here.|
> |dbx|Forbidden Signatures|Contains hashes of revoked binaries. Used to blacklist known-compromised bootloaders (e.g., old GRUB vulnerabilities).|
> 
> When the firmware loads an `.efi` binary, it checks: "Is this binary signed by a key in `db`? Is its hash absent from `dbx`?" If both conditions are met, execution proceeds. Otherwise, the firmware halts with a security violation.
> 
> `sbctl` manages this entire chain from userspace — generating keys, enrolling them into firmware NVRAM, and signing binaries.
> 
> _Reference: UEFI Specification, Section 32 — Secure Boot and Driver Signing; Arch Wiki — Secure Boot ([https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot](https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot))_

### A - Generate Your Secure Boot Keys

First, check the current Secure Boot state:

```bash
sbctl status
```

At this stage, the expected output will show Secure Boot as **disabled** (you should have set it to Setup Mode in UEFI firmware settings before beginning the installation). Setup Mode means the firmware is ready to accept new key enrollment.

Generate a fresh key hierarchy:

```bash
sbctl create-keys
```

> [!NOTE] 
> **What `sbctl create-keys` Generates**
> 
> This creates a complete PKI (Public Key Infrastructure) under `/usr/share/secureboot/keys/`:
> 
> |File|Purpose|
> |---|---|
> |`PK/PK.key` + `PK.pem`|Platform Key — private key and certificate.|
> |`KEK/KEK.key` + `KEK.pem`|Key Exchange Key — private key and certificate.|
> |`db/db.key` + `db.pem`|Signature Database key — this is the key used to sign your `.efi` binaries.|
> 
> The private keys (`*.key`) must remain on the system and should be protected. If an attacker obtains `db.key`, they can sign arbitrary boot code that your firmware will trust.

Enroll the keys into firmware NVRAM:

```bash
sbctl enroll-keys --microsoft
```

> [!WARNING] 
> **The `--microsoft` Flag Is Mandatory for Dual-Boot**
> 
> Without `--microsoft`, `sbctl` enrolls only your custom keys into the `db` database. This means the firmware will **refuse to execute** any binary not signed by your personal key — including Windows Boot Manager (`bootmgfw.efi`), which is signed by Microsoft's UEFI CA key.
> 
> The `--microsoft` flag adds Microsoft's two standard UEFI signing certificates to `db` alongside your own:
> 
> |Microsoft Certificate|What It Covers|
> |---|---|
> |Microsoft Corporation UEFI CA 2011|Signs third-party UEFI drivers and boot managers (including Windows Boot Manager).|
> |Microsoft Windows Production PCA 2011|Signs Windows kernel components.|
> 
> Omitting this flag on a dual-boot system will render Windows unbootable until you enter UEFI Setup and either disable Secure Boot or manually add the Microsoft certificates.

### B - Sign All Boot Components

Every `.efi` binary on the ESP must be signed with your `db` key. The `-s` flag tells `sbctl` to **save** the file path for automatic re-signing when packages are updated.

Sign the `systemd-boot` binaries:

```bash
sbctl sign -s /usr/lib/systemd/boot/efi/systemd-bootx64.efi
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
```

> [!NOTE] 
> **Why Three Paths for systemd-boot?**
> 
> |Path|Reason|
> |---|---|
> |`/usr/lib/systemd/boot/efi/systemd-bootx64.efi`|The **source** binary installed by the `systemd` package. When `systemd` is updated, this file is replaced. Signing it with `-s` ensures `sbctl` re-signs it automatically after updates, so subsequent `bootctl update` operations copy an already-signed binary.|
> |`/boot/EFI/systemd/systemd-bootx64.efi`|The **installed** copy on the ESP — the one firmware actually executes.|
> |`/boot/EFI/BOOT/BOOTX64.EFI`|The **fallback** copy. Must also be signed or the firmware will reject it during fallback boot scenarios.|

Sign both Unified Kernel Images:

```bash
sbctl sign -s /boot/EFI/Linux/*.efi
```

> [!TIP] 
> **The `-s` (Save) Flag and Automatic Re-signing**
> 
> Every path passed with `-s` is recorded in `/usr/share/secureboot/sbctl/files.json`. When you run `sbctl sign-all` (or when a pacman hook triggers it after a kernel update), `sbctl` iterates through this list and re-signs every recorded file. This means:
> 
> A - After a kernel update that regenerates UKIs, running `sbctl sign-all` re-signs them without you needing to remember each path.
> 
> B - After a `systemd` update that replaces the boot manager binary, the same process re-signs it.
> 
> Without `-s`, you would need to manually re-sign every file after every relevant package update - a fragile process that inevitably leads to a Secure Boot violation and an unbootable system.

### C - Verify the Signature Chain

```bash
sbctl verify
```

Expected output: every listed file should show a status of **Signed**. If any file shows **Not Signed**, re-examine the signing commands — the most common cause is a glob pattern (`*.efi`) that did not expand as expected because the working directory or path was incorrect.

> [!WARNING] 
> **Enabling Secure Boot Enforcement**
> 
> At this point, the keys are enrolled and all binaries are signed, but Secure Boot may still be in Setup Mode or disabled in your firmware settings. After rebooting (Chapter IX), enter your UEFI firmware setup (typically by pressing `F2` or `Del` during POST on the ASUS G16) and:
> 
> A - Verify that Secure Boot is set to **Enabled**.
> 
> B - Verify that the mode shows **User Mode** (not Setup Mode). User Mode means the PK is enrolled and enforcement is active.
> 
> If the firmware still shows Setup Mode after key enrollment, the enrollment may have failed silently. Boot back into Arch, run `sbctl status` to inspect the state, and re-enroll if necessary.

# Chapter IX - Services and Finalization

This is the final chapter of the installation. We enable the daemons that provide runtime functionality, then perform a clean dismount sequence that ensures every buffer is flushed, every filesystem is cleanly unmounted, and every encrypted volume is properly closed before the first boot.

## Part I — Enable Essential Services

These services are **enabled** (scheduled to start at boot) but not **started** — we are still inside a chroot with no active init system. They will activate on the first real boot.

```bash
systemctl enable NetworkManager
systemctl enable systemd-timesyncd
systemctl enable power-profiles-daemon
systemctl enable supergfxd
systemctl enable fstrim.timer
```

> [!NOTE] 
> **Service Breakdown**
> 
| Service                 | Unit Type      | Function                                                                                                                                                                                                                                                | Consequence If Missing                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NetworkManager`        | System service | Manages all network interfaces — Wi-Fi (via the `iwd` backend we configured in Chapter VI), Ethernet, VPN connections. Provides `nmcli` and `nmtui` for command-line network management.                                                                | No network connectivity after boot. Wi-Fi will not associate, no DHCP lease will be obtained.                                                                                                                                                                                                                                                                             |
| `systemd-timesyncd`     | System service | Lightweight SNTP (Simple Network Time Protocol) client. Synchronizes the system clock against NTP servers on every boot and periodically thereafter.                                                                                                    | Clock drift accumulates over time. TLS certificate validation may fail (certificates have validity windows). `pacman` signature checks may fail if the clock is sufficiently skewed. Filesystem timestamps become unreliable.                                                                                                                                             |
| `power-profiles-daemon` | System service | Exposes power profile switching via D-Bus — `power-saver`, `balanced`, `performance`. Integrates with desktop environments (GNOME, KDE) and with `asusctl` for ASUS-specific performance modes.                                                         | Laptop runs at default firmware power state. No userspace control over CPU frequency governor or platform power limits. Battery life on a mobile workstation becomes unpredictable.                                                                                                                                                                                       |
| `supergfxd`             | System service | Daemon for `supergfxctl` — manages GPU mode switching between Integrated (Intel UHD only), Hybrid (NVIDIA on-demand via PRIME offloading), Dedicated (NVIDIA only), and Compute (NVIDIA for CUDA workloads without display).                            | The system defaults to whatever GPU mode was last configured in firmware. You lose the ability to dynamically switch between battery-efficient integrated graphics and NVIDIA performance without rebooting.                                                                                                                                                              |
| `fstrim.timer`          | Systemd timer  | Executes `fstrim --fstab --verbose` weekly (every Monday at midnight, with a randomized delay up to 6 hours to avoid thundering herd on multi-system deployments). Sends TRIM commands to all mounted filesystems that support the `discard` operation. | The SSD controller progressively loses track of which blocks are free. Over weeks and months, write amplification increases as the controller must read-erase-write entire NAND blocks instead of writing to pre-erased cells. Sustained write performance degrades measurably — on NVMe drives, this manifests as periodic latency spikes during heavy write operations. |
> _Reference: Arch Wiki — NetworkManager ([https://wiki.archlinux.org/title/NetworkManager](https://wiki.archlinux.org/title/NetworkManager)); Arch Wiki — Solid state drive ([https://wiki.archlinux.org/title/Solid_state_drive#TRIM](https://wiki.archlinux.org/title/Solid_state_drive#TRIM)); asus-linux.org — supergfxctl documentation_

> [!Abstract] 
> **`discard=async` in fstab vs `fstrim.timer` — Why Both?**
> 
> This is a common point of confusion. Our `fstab` entries (generated in Chapter V) already include the `discard=async` mount option. Understanding why the timer is still necessary requires distinguishing between the two TRIM strategies:
> 
| Strategy                        | Mechanism                                                                                                                          | Scope                                                                                                                                                                                                                           |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discard=async` (continuous)    | The filesystem queues TRIM commands as blocks are freed during normal operation, then batches them asynchronously.                 | Handles **ongoing** deletions in real-time. Keeps the SSD informed of newly freed blocks as they occur.                                                                                                                         |
| `fstrim.timer` (periodic batch) | A scheduled job walks the entire filesystem, identifies **all** unallocated regions, and issues TRIM for the complete set at once. | Acts as a **sweep** — catches anything the continuous discard may have missed, and handles edge cases such as blocks freed during early boot before the filesystem was fully mounted, or regions orphaned by unclean shutdowns. |
The continuous mechanism handles the steady state. The weekly timer handles the residual. Together, they ensure comprehensive block hygiene. Neither alone is fully sufficient — `discard=async` misses edge cases, and relying solely on weekly `fstrim` means the drive operates with stale block maps for up to seven days between runs.
There is one additional consideration specific to our encrypted setup. TRIM commands must pass through the LUKS2 layer to reach the physical NVMe controller. By default, `dm-crypt` **blocks** TRIM passthrough for security reasons — TRIM patterns on an encrypted volume can theoretically reveal information about which blocks contain data versus which are empty, weakening the encryption's plausible deniability properties. However, `cryptsetup` since version 2.x with LUKS2 defaults allow this passthrough when `discard` is specified at the filesystem level. Verify that your LUKS containers permit TRIM by checking:
>```bash
>dmsetup table cryptos | grep -o 'allow_discards'
>dmsetup table cryptvms | grep -o 'allow_discards'
>```
If `allow_discards` does not appear, the TRIM commands from both mechanisms are silently discarded at the `dm-crypt` layer and never reach the SSD. In that case, you would need to add `--allow-discards` when opening the LUKS containers (via a dracut configuration option for the root volume, and via `crypttab` for any secondary volumes).
>
_Reference: Arch Wiki — dm-crypt/Specialties — Discard/TRIM support ([https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_\(SSD\)))_

## Part II - Exit and Clean Dismount

The dismount sequence is **order-dependent**. Deviating from this order can cause data loss (unflushed write caches), filesystem corruption (unmounting a Btrfs parent while a subvolume is still mounted), or kernel warnings (closing a LUKS container with mounted filesystems inside).

### A - Exit the Chroot

```bash
exit
```

This terminates the chroot shell session and returns you to the live ISO environment. Your prompt should revert to the live ISO's default (e.g., `root@archiso ~ #`). The path `/mnt` is once again an external mount point, not the system root.

### B - Deactivate Swap

```bash
swapoff /mnt/swap/swapfile
```

> [!NOTE] 
> The swapfile resides on the `@swap` Btrfs subvolume inside the `cryptos` LUKS container. It must be deactivated **before** the filesystem is unmounted and **before** the LUKS container is closed. If the kernel still holds the swapfile open, `umount` will fail with `target is busy`, and `cryptsetup close` will refuse to release the mapped device.

### C - Unmount All Filesystems

```bash
umount -R /mnt
```

> [!NOTE] 
> **The `-R` (Recursive) Flag**
> 
> `/mnt` is the root of a complex mount tree:
> 
> |Mount Point|Source|
> |---|---|
> |`/mnt`|`cryptos`, subvol `@`|
> |`/mnt/home`|`cryptos`, subvol `@home`|
> |`/mnt/var`|`cryptos`, subvol `@var`|
> |`/mnt/.snapshots`|`cryptos`, subvol `@snapshots`|
> |`/mnt/swap`|`cryptos`, subvol `@swap`|
> |`/mnt/boot`|`/dev/nvme1n1p1` (ESP)|
> |`/mnt/vms`|`cryptvms` (XFS)|
> 
> The `-R` flag instructs `umount` to process all mounts under `/mnt` in reverse-topological order — leaf mounts first (`/mnt/home`, `/mnt/var`, `/mnt/boot`, `/mnt/vms`, etc.), then the root (`/mnt`). This guarantees that no parent is unmounted while children still depend on it.
> 
> If `umount -R /mnt` reports any `target is busy` error, identify the blocking process with:
> 
> ```bash
> fuser -vm /mnt
> ```
> 
> The most common cause is a shell session whose working directory is inside `/mnt`, or the swapfile not having been deactivated in the previous step.

### D - Close Encrypted Volumes

```bash
cryptsetup close cryptos
cryptsetup close cryptvms
```

> [!NOTE] 
> **What `cryptsetup close` Does**
> 
> This command performs two actions:
> 
> A - Flushes any remaining data in the `dm-crypt` write buffer to the underlying physical partition (`/dev/nvme1n1p2` and `/dev/nvme1n1p3` respectively).
> 
> B - Removes the device-mapper node (`/dev/mapper/cryptos`, `/dev/mapper/cryptvms`) from the kernel. The encrypted data on the raw partitions is now inaccessible without the passphrase.
> 
> The order matters: `cryptos` must be closed **after** all its Btrfs subvolume mounts are released. `cryptvms` must be closed **after** `/mnt/vms` is unmounted. Since `umount -R /mnt` already handled both, the `close` commands should succeed immediately. If either reports `Device is still in use`, a mount or swap reference was not properly released — retrace the previous steps.

## Part III - Reboot

```bash
reboot
```

> [!WARNING]
> **Remove the installation media** (USB drive) immediately. If the USB remains connected, the UEFI firmware may boot from it again instead of the NVMe drive, depending on your boot priority settings.

> [!NOTE] 
> **Expected First Boot Sequence**
> 
> If everything was configured correctly, the following sequence will occur:
> 
> |Stage|What You See|What Is Happening|
> |---|---|---|
> |UEFI POST|ASUS ROG logo|Firmware initializes hardware, verifies Secure Boot keys.|
> |systemd-boot menu|Kernel selection (10-second timeout)|Boot manager auto-discovers UKIs in `/boot/EFI/Linux/`. Both `linux-lts` and `linux-g14` should appear.|
> |LUKS passphrase prompt|`Please enter passphrase for disk cryptos:`|Dracut's `crypt` module reads `rd.luks.uuid` from the embedded command line, locates `/dev/nvme1n1p2`, and prompts for decryption.|
> |System init|Brief text output or blank screen|`systemd` takes over, mounts filesystems per `fstab`, starts enabled services.|
> |Login prompt|`zephyrus-g16 login:`|System is operational. Log in with the user account created in Chapter VI.|
> 
> **If the LUKS prompt does not appear** — you land in a dracut emergency shell or the firmware reports a Secure Boot violation — boot from the USB installation media, decrypt and mount the volumes, chroot back in, and inspect `/etc/dracut.conf.d/30-cmdline.conf` for UUID errors. Regenerate UKIs and re-sign them.

> [!TIP] 
> **Immediate Post-Boot Verification Checklist**
> 
> After logging in for the first time, run the following to confirm the system is fully operational:
> 
> ```bash
> # Verify all filesystems are mounted
> findmnt --real
> 
> # Verify swap is active
> swapon --show
> 
> # Verify NVIDIA driver is loaded
> lsmod | grep nvidia
> 
> # Verify network connectivity
> nmcli device status
> 
> # Verify Secure Boot enforcement (should show "Secure Boot: enabled")
> sbctl status
> 
> # Verify ASUS daemon is running
> systemctl status supergfxd
> 
> # Check for any failed services
> systemctl --failed
> ```
> 
> Any failures at this stage should be investigated before proceeding to install a desktop environment, window manager, or additional software.

