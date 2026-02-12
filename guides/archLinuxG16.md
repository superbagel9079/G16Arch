
## Part I: Drive Identification

Before invoking any destructive commands, we must absolutely confirm the physical path of the Linux drive. Since both drives are 2TB, size is not a differentiator.

>[!WARNING] 
>Critical Safety Check You must identify which controller corresponds to the Windows drive and which to the target. Usually:
>- `/dev/nvme0n1` is the first slot (Windows).
>- `/dev/nvme1n1` is the second slot (Target). **However**, you must verify this by checking the existing partition tables.

Execute the following to confirm the target:

```bash
lsblk -o NAME,MODEL,SIZE,TYPE,FSTYPE
fdisk -l /dev/nvme0n1
fdisk -l /dev/nvme1n1
```

- The drive containing `Microsoft Basic Data` or `NTFS` partitions is your **Windows Drive**. **Touch nothing on this controller.**
- The drive intended for Linux should be the one you target. For this guide, we assume it is **`/dev/nvme1`** (the controller) and **`/dev/nvme1n1`** (the namespace).

## Part II: Capability Analysis

Check if your specific NVMe controller supports the `sanitize` command and which methods are available (Crypto Erase is preferred for speed and security).

Run the following on your target controller (e.g., `nvme1`):

```bash
nvme id-ctrl /dev/nvme1 -H | grep -E 'Sanitize|Crypto'
```

**Interpret the Output:**

- `Crypto Erase Sanitize Operation Supported`: **Yes/No**
- `Block Erase Sanitize Operation Supported`: **Yes/No**

If **Crypto Erase** is supported, we will use that. It takes seconds. If only **Block Erase** is supported, it may take minutes to hours.

## Part III: Execution of Sanitize

>[!NOTE] 
>**Sanitize vs Format**
>
>We use `sanitize` because it acts at the controller level, ensuring all caches are flushed and the process survives a reboot if interrupted (though interrupting is not recommended).

**Option A: Crypto Erase (Fastest & Secure)** If the capabilities check returned "Supported" for Crypto Erase:

```bash
nvme sanitize /dev/nvme1 -a start-crypto-erase
```

**Option B: Block Erase (Slower)** If Crypto Erase is unavailable:

```bash
nvme sanitize /dev/nvme1 -a start-block-erase
```

**Option C: Fallback to Format** If `sanitize` fails or is unsupported by the firmware, fall back to the Format command with Secure Erase Settings (`ses=1`):

```bash
nvme format /dev/nvme1n1 --ses=1
```

## Part IV: Verification

Monitor the progress. Do not power off the machine.

```bash
nvme sanitize-log /dev/nvme1
```

Wait until `Sanitize Status` shows completion (usually `0x101` or similar success code) and the progress bar reaches 65535.

Once the sanitize is complete, the kernel may not immediately register the drive's empty state. It is best practice to trigger a rescan or a quick reboot, but usually, a rescan suffices:

```bash
nvme list
lsblk
```

The drive `/dev/nvme1n1` should now appear completely empty, with no partition table.
