
# Drive Snapshot Manual

**Program Version 1.50**  
© Tom Ehlert Software e.K.  
www.drivesnapshot.de

---

## Table of Contents

1.  Introduction  
     1.1 Image Technology  
     1.2 System Requirements  
     1.3 Supported File Systems  
     1.4 Installing Snapshot  
     1.5 Entering License Data
    

---

## 1 Introduction

### 1.1 Image Technology

A **snapshot image** captures the exact state of a disk at the moment a backup begins.  
When Drive Snapshot starts a backup, it records the current condition of the disk and saves all *used clusters* based on that initial state. Any file changes that occur during the backup are **not** included in the image.

If you create images of all partitions, you can completely restore your system after a disk failure by writing the images to a new drive.  
After restoration, the disk will contain *precisely* the same data as it did at the time of the backup.

---

### 1.2 System Requirements

Drive Snapshot supports the following operating systems (both 32-bit and 64-bit editions, where available):

-   Windows NT 4 SP6
    
-   Windows 2000 SP4 + Update Rollup 1 (v2)
    
-   Windows XP and Server 2003
    
-   Windows Vista and Server 2008
    
-   Windows 7 and Server 2008 R2
    
-   Windows 8 / 8.1 and Server 2012
    
-   Windows 10 and Server 2012 R2
    
-   Windows Server 2016
    
-   Windows Server 2019
    
-   Windows 11 and Server 2022
    

---

### 1.3 Supported File Systems

Drive Snapshot can back up **any** file system.  
However, *intelligent* (cluster-based) backups are only available for these supported file systems:

-   **FAT**
    
-   **NTFS**
    
-   **ReFS**
    
-   **EXT2/EXT3/EXT4**
    
-   **ReiserFS**
    
-   **XFS**
    

For unsupported or unknown file systems, Snapshot performs a *sector-by-sector* backup of the entire partition.

---

### 1.4 Installing Snapshot

There are two ways to make Drive Snapshot available on a computer:

1.  **Install using `Setup.exe`**
    
    -   The installer adds a Start Menu entry and associates the `.sna` file extension with Snapshot.
        
2.  **Portable use**
    
    -   Simply copy `snapshot.exe` to any accessible location.
        
    -   No installation is required.
        
    -   This is convenient for administrators managing many machines—place the executable on a shared network drive so all systems can use it.
        

---

### 1.5 Entering License Data

There are two licensing methods:

#### **Method 1 – Embed license data in `snapshot.exe`**

Launch Snapshot and choose **“Enter License Info.”**  
A dialog appears prompting you to paste the full license text you received from the vendor.

**Steps:**

1.  Open the license file in *Notepad*.
    
2.  Press `Ctrl + A` to select all text, then `Ctrl + C` to copy it.
    
3.  Switch to Snapshot’s dialog and paste it with `Ctrl + V`.
    
4.  Click **Register**.
    

Snapshot is now licensed.  
At the next start, your license details will appear in the main window.

> **Note:** This method keeps the executable licensed even if you copy `snapshot.exe` to another computer.

#### **Method 2 – Use a separate license file**

Copy your provided license file into the same directory as `snapshot.exe`.  
When Snapshot starts, it automatically reads and displays the license data.  
If you later move `snapshot.exe` to a new location, you must copy the license file along with it.


# 2 Creating Backups

Drive Snapshot can create **full** and **differential** disk images.  
Each image is generated **per partition**:

-   A **full image** contains all *used sectors* of a partition.
    
-   A **differential image** includes only the sectors changed since the last full backup, saving considerable space.
    

---

## 2.1 Creating a Full Image

A full image includes every used cluster of the selected partition along with partition metadata from the entire disk.

### 2.1.1 Full Backup Using the Graphical Interface

1.  Start **Drive Snapshot**.
    
2.  On the start screen, select **Backup Disk to File**.
    

A dialog appears listing all available partitions.

You can:

-   Select multiple partitions using **Ctrl + click**.
    
-   Click **Next** to proceed.
    

You’ll then be prompted to **name the image file**.  
If several partitions are being backed up, the filename **must contain** the variable `$disk`, which Snapshot replaces at runtime with each drive’s letter (or another unique ID for unlettered volumes).  
This ensures that every partition gets a unique image filename.

> See Section 2.6 for the full list of supported variables.

---

### Dialog Options Explained

1.  **Manage FTP Accounts**  
    You can back up not only to local or network drives but also to an **(S)FTP server**.  
    Clicking this button opens a dialog to define and save your server credentials.  
    See Section 2.5 for more details.
    
2.  **Empty Recycle Bin Now**  
    Deletes all files in the Recycle Bin before the backup starts.  
    The dialog also shows how many files and how much space are currently in the bin.
    
3.  **Disk Image Encryption**  
    Drive Snapshot supports **AES-256 encryption** for images created during backup.
    
    -   Enter and confirm a password in *Password* and *Verify Password*.
        
    -   Clicking **Store Encryption Password** saves a derived key in the Windows Registry — all future backups from that system will use the same encryption automatically.
        
    -   **Store Decryption Password** saves the password (encrypted) in the Registry so you won’t need to re-enter it when mounting images on that same computer.
        
    
    > ⚠️ **Important:**  
    > Saving the decryption password is *not* completely secure — though encrypted, it could be recovered with enough effort. Only enable this option if you understand the risk.
    

---

## 2.2 Creating a Differential Image

A **differential backup** records all changes that occurred on a partition since its most recent full backup.

Snapshot compares **hash values** of all blocks:

1.  It reads all used sectors of the partition.
    
2.  Calculates hashes for each block.
    
3.  Compares them with those stored in the full backup’s `.hsh` file.
    
4.  Writes only changed blocks into the differential image.
    

Only the `.hsh` file is needed to generate a differential image — the full image file itself doesn’t need to be accessible.

---

### 2.2.1 Differential Backup Using the GUI

1.  Begin the process the same way as for a full image.
    
2.  When asked for a filename, choose one that distinguishes it from the full backup.
    
    Example:
    
    ```yaml
    Full image:  MyImageOfC.sna  
    Differential: MyImageOfC-diff.sna
    ```
    
    You can also include `$` variables as described in Section 2.6.
    
3.  Check **Differential Image**.  
    A new input box appears — specify the path to the `.hsh` file from the original full backup.
    
4.  Click **Start Copy** to begin.
    

---

## 2.3 Using VSS (Volume Shadow Copy Service)

Windows’ **Volume Shadow Copy Service (VSS)** allows consistent backups of open files by creating a snapshot of the data.

Snapshot can interact with VSS in two ways:

1.  **VSS without writers**
    
    -   Creates a raw copy of data without involving application-specific VSS Writers.
        
    -   Suitable for general backups but won’t perform app-level actions (e.g., log truncation in Exchange, VM coordination in Hyper-V).
        
    -   Snapshot automatically uses this mode if multiple partitions are backed up together or if it detects a running Exchange Server.
        
2.  **VSS with writers**
    
    -   All active VSS Writers are notified before and after the backup.
        
    -   Writers (like Exchange Server or Hyper-V) may pause services or clean up transaction logs.
        
    -   You can force this mode in **Advanced Options → “Explicitly include all VSS writers.”**
        

> For fine-grained control, see the CLI options `--IncludeWriter` and `--ExcludeWriter` in Chapter 5.

**List installed writers:**

```nginx
vssadmin list writers
```

---

## 2.4 Backup Without VSS

When VSS isn’t available (e.g., older Windows versions before XP/2003), Snapshot uses its own **internal driver** to maintain consistency.

-   Write operations to the target partition are delayed until the driver has copied the relevant data.
    
-   Only data existing at backup start is included; later changes are ignored.
    
-   Multiple partitions cannot be backed up simultaneously in this mode.
    

You can force non-VSS backups in **Advanced Options → “Never use VSS.”**

---

## 2.5 Using an (S)FTP Server

Snapshot can save and restore images directly to or from **FTP or SFTP servers** using special URL-like filenames.

### FTP Syntax

```pgsql
ftp://username:password@server:port/path/filename.sna
```

### SFTP Syntax

```pgsql
sftp://username:password@server:port/path/filename.sna
```

-   The port number is optional.
    
-   For SFTP, you can authenticate with a **PuTTY-style key file** (`.ppk`) instead of a password.
    
-   Stored account passwords are encrypted (AES) in the Windows Registry.
    
-   Full backups made directly to (S)FTP do **not** create a local hash file; if you need one, specify it manually using `-o` followed by a local path.
    

---

### 2.5.1 Managing (S)FTP Accounts

Click **Manage FTP Accounts** to add or edit saved accounts.  
The dialog lets you:

-   Create (`New Account`), edit, or delete accounts.
    
-   Choose between FTP and SFTP.
    
-   For SFTP, optionally select a PuTTY key file instead of a password.
    
-   Saved credentials are encrypted in the Registry.
    

Once created, choose **Select** to use that account as the destination in the backup dialog—then simply append the image filename.

---

## 2.6 Special Filename Variables

Snapshot recognizes several `$`\-prefixed **variables** inside image filenames.  
At runtime, these are automatically replaced with real values, allowing flexible automatic naming when used in scripts.

| Variable | Description |
| --- | --- |
| `$disk` | Drive letter or a unique ID (if no drive letter) |
| `$HD` | Identifier like `HD1-1` (disk # 1, partition # 1) |
| `$computername` | Current computer name |
| `$type` | Image type: `ful` for full, `dif` for differential |
| `$date` | Current date (YYMMDD) |
| `$day` / `$month` / `$year` | Day, month, 4-digit year |
| `$hour` / `$minute` / `$second` | Current time components |
| `$weekday` | 3-letter English weekday (Mon, Tue …) |
| `$week` | ISO calendar week (two digits) |

**Example usage:**

```nginx
snapshot C:+D: $label="Backup"\images\$disk.sna
```

### Drive Label Variable

You can target drives dynamically by label:

```bash
$label="Backup"
```

This expression is replaced with the drive letter of the first volume named “Backup”.  
Wildcards are allowed — e.g.

```bash
$label="Backup*"
```

matches the first drive whose label starts with “Backup”.  
Useful for rotating USB drives that share a common name prefix.

---

## 2.7 Advanced Options

The **Advanced Options** dialog allows control over technical parameters during image creation.

**VSS Options:**

-   `--NoVSS` Use Snapshot’s internal driver only.
    
-   `--UseVSS` Try VSS first; fallback to internal driver if it fails (default).
    
-   `--ForceVSS` Abort on VSS failure. Recommended for multi-volume backups.
    

**Additional Settings:**

-   **Beep when operation completes** — plays a sound after backup.
    
-   **Create directory if not existing** — auto-creates the target path (`--CreateDir`).
    
-   **Enable log file** — generates a log (applies on next run).
    
-   **Save these settings as default** — stores current options permanently.
    
-   **Save entire disk (unused sectors too)** — performs a forensic-level sector-by-sector backup.
    
-   **Maximum image single file size** — splits output into chunks (same as `-L` option).
    
-   **Automatic backup of small boot partitions** — auto-includes partitions below a set size (default 512 MB).
    
-   **Always generate hash file** — ensures differential backups are possible later.
    
-   **Create hash file from full image** — create a `.hsh` from an existing full image.
    

> 💡 To restore the entire system properly, always include **all partitions** Snapshot detects on the disk.

---

## 2.8 Testing an Image

Snapshot can verify image integrity.

During backup, Snapshot writes **checksums** alongside data blocks.  
Testing recomputes these checksums and compares them to the stored values:

-   If they match → the image is intact.
    
-   If they differ → corruption has occurred, and Snapshot reports an error.
    

### Ways to Test

**Command Line:**

```bash
snapshot -t image.sna
```

**Graphical Interface:**  
Use the **Test Image** button (see Chapter 3, Figure 3.1).

If an image passes verification successfully, it can be restored safely.

---

# 3 Restoring with Snapshot

The restoration process in Drive Snapshot always follows a consistent workflow:

1.  Restore the **partition structure** from one of your images to a target disk.
    
    -   The target disk must be at least as large as the original.
        
2.  Restore each **partition image**.
    

Since version **1.44**, Snapshot can also restore **entire disks** automatically — this option is available only in the Windows version, not under DOS.

---

## 3.1 Restoring Under DOS

If Snapshot is installed, you can find a **“Create Disaster Recovery Diskette”** option under the program’s Start Menu entry.  
This creates a **bootable diskette** that can be used to restore an image after system failure.

If your computer lacks a floppy drive, you can create a **bootable CD** instead:

```bash
snapshot -X networkbootdiskette.sna bootimage.img
```

This command creates the file `bootimage.img`, which can then be added as the **boot image** in your CD-burning software.

Once you boot from this medium:

1.  List detected disks:
    
    ```bash
    snapshot show
    ```
    
2.  Identify the target drive (for example, `HD1`).
    
3.  Restore the partition structure:
    
    ```bash
    snapshot restore HD1 partitionstructure W:\C-DRIVE.SNA
    ```
    
4.  Then restore each partition image:
    
    ```bash
    snapshot restore HD1 auto W:\C-DRIVE.SNA
    ```
    

Repeat the last step for every partition using its corresponding image file.

> 📘 More details: Restoring under DOS

---

## 3.2 Restoring Under Windows

Instead of FreeDOS, you can use any **Windows PE** environment — even a standard **Windows 7 DVD** works as a bootable medium.

The advantage:  
You can access Snapshot’s **graphical interface** for an easier restore process.

> Tip: Copy `snapshot.exe` into the same folder as your image files — it saves you from typing long paths.

### Step-by-Step

1.  Boot from Windows PE or the Windows DVD.
    
2.  Launch `snapshot.exe`.
    
3.  Select **Restore Disk from File**.
    

A dialog appears (see *Figure 3.1* in the manual) showing image details such as:

-   Backup date and time
    
-   Partition info
    
-   Filesystem type
    
-   Volume label
    

#### Testing Before Restore

Click **Test Image** to verify data integrity.  
Snapshot checks all stored checksums — if they match, the image is valid.

#### Restoring the Partition Structure

Click **Next**, and a graphical disk map appears (see *Figure 3.2*).  
Right-click the destination drive and select **Restore Partition Structure**.

This writes the **Master Boot Record (MBR)** and recreates all partitions at their original offsets.

If restoring to a **larger disk**, you can use **“Grow all NTFS Partitions…”** to expand all partitions (≥1 GB) to fill unused space automatically.

Finally, restore each partition image.  
Once done, the restored disk will match the original disk exactly as it was at backup time.

#### Creating New Partitions

If there’s unallocated space left, you can right-click it and create new **partitions or logical drives** directly from Snapshot (see *Figure 3.3*).

---

### 3.2.1 Restoring from an (S)FTP Server

Restoring from an (S)FTP server works exactly like restoring from a local file —  
just use the appropriate **(S)FTP-style filename syntax** described in Section 2.5.

---

### 3.2.2 Restoring an Entire Disk

Starting with version **1.44**, Snapshot can restore a **whole disk** if all its partitions were backed up together.

**Requirements:**

-   All image files must be in the same directory.
    
-   FTP locations are not supported for full-disk restore.
    
-   Restoration during reboot is not available in this mode.
    

#### How to Restore a Full Disk

1.  On the start screen, choose **Restore Complete Disk from Images**.
    
2.  Select any image from the set — Snapshot automatically detects related ones.
    
3.  Deselect any images you don’t wish to restore (usually all should remain selected).
    
4.  Choose the **destination disk** at the bottom of the window.
    
5.  Optionally uncheck **“Restore partition structure”** if the disk is already correctly partitioned.
    

After confirming all settings:

-   Click **Next**.
    
-   A final warning shows the target disk details.
    
-   Confirm with **OK** to start the process.
    

You can then monitor progress and estimated time in a live progress dialog.

---

### 3.2.3 Restoring to Different Hardware

If you restore a system to a machine with different hardware (especially a new **disk controller**), you may need to **inject new drivers** into the restored system.

This requires running under **Windows**, not FreeDOS.

#### Case 1: IDE/SATA in IDE Mode

Run:

```bash
snapshot --mergeide
```

-   You’ll be prompted for the restored Windows directory.
    
-   Snapshot inserts the necessary IDE driver automatically.
    

#### Case 2: Other Controller Types (e.g., RAID/SCSI/NVMe)

Run:

```bash
snapshot --AddDriver
```

You’ll then specify:

1.  The path to the restored Windows directory.
    
2.  The `.inf` driver file.
    
3.  The corresponding `.sys` driver file.
    

> If Snapshot reports  
> `Can't find a device with matching PCI Id.`  
> then you’re using the wrong driver or INF file.

---

## 3.3 Restoring During System Reboot

If your OS still boots and you want to **restore the system partition itself**, Snapshot can schedule the restore to happen **at the next reboot**.

1.  Begin a normal restore operation as usual (see Section 3.2).
    
2.  When Snapshot detects you’re restoring the active system partition, it shows a prompt (like *Figure 3.6*):
    
    -   Choose whether to perform the restore on next reboot.
        
    -   Configure how errors should be handled.
        

**Conditions:**

-   The image must be stored on a local drive.
    
-   It cannot reside on the same partition that’s being restored.
    

After confirming:

-   Snapshot asks whether to reboot immediately.
    
-   During the next startup, the restoration will execute before Windows loads.
    

If you accidentally scheduled a restore, you can remove it by reopening Snapshot and choosing **No** in the same dialog.

You can also manage scheduled restores from the command line — see Section 5.2.

---

# 4 Mounting an Image as a Virtual Drive

Mounting an image as a **virtual disk** allows you to browse and copy individual files or folders from a backup image — without restoring the entire partition first.

---

### Mounting Through the Graphical Interface

1.  Open **Drive Snapshot**.
    
2.  On the start screen, select **Mount Image as Virtual Disk**.
    

A file selection dialog appears (see Figure 4.1 in the original manual).

-   Choose the image (`.sna`) you want to mount.
    
-   Select a **drive letter** for the virtual disk.
    
-   Click one of the following buttons:
    

#### • **Map Virtual Drive**

Mounts the image as a new drive (e.g., `Z:`) in Windows.

#### • **Map and Explore Virtual Drive**

Mounts the image *and automatically opens* a File Explorer window for the mounted drive.

Once mounted, the image behaves like a normal read-only disk — you can browse, open, and copy files freely.

---

### Mounting by Double-Click (File Association)

If Drive Snapshot was installed via **Setup.exe**, the `.sna` file extension is automatically associated with the program.  
That means you can simply **double-click** any `.sna` file to mount it as a virtual drive.

If you’re using the **portable** version (without installation):

-   You can manually set Snapshot as the default `.sna` file handler by clicking  
    **Set Snapshot as .SNA Viewer** in the Mount dialog.
    
-   After that, double-clicking `.sna` files in Windows Explorer will mount them just as if Snapshot were installed.
    

---

### Supported Image Types

Both **full** and **differential** images can be mounted in exactly the same way.  
Mounted differential images include all data up to the point of that differential backup.

---

### Unmounting a Virtual Drive

When you finish accessing files, simply **close** the mounted drive window or unmount it via the Snapshot interface.

You can also unmount drives from the command line using:

```bash
snapshot --unmount
```

or for a specific drive:

```bash
snapshot --unmount:Z
```

---

# 5 Command Line Reference

Drive Snapshot can perform **all operations** via the command line — ideal for automation, batch jobs, or integration into existing backup scripts.

The executable `snapshot.exe` accepts numerous parameters, allowing you to create, verify, restore, or mount images directly from scripts.

---

## 5.1 Command Line Options

### Basic Syntax

```php-template
snapshot [options] <source> <target>
```

Example:

```bash
snapshot C: D:\Backup\MySystem_$date.sna -L650 -W
```

---

### Common Parameters

| Parameter | Description |
| --- | --- |
| `C:` | Source drive (to back up) |
| `D:\backup.sna` | Target file or path for the image |
| `-h` | Display help text |
| `-s` | Silent mode — suppresses all user interaction |
| `-L<size>` | Split image into multiple parts, each of `<size>` MB |
| `-Go` | Start the operation immediately without confirmation |
| `-R` | Restore image (used with `restore` subcommand) |
| `-t` | Test the integrity of an image |
| `-o` | Write `.hsh` file to specified location for differential backup |
| `-W` | Create a hash file (`.hsh`) after backup completes |
| `-x` | Mount image file as a virtual drive |
| `-u` | Unmount a mounted image |
| `-b` | Automatically perform backup using specified configuration |
| `--BackupAll` | Create full backup of all drives and partitions |
| `--NoVSS` | Do not use Volume Shadow Copy Service |
| `--UseVSS` | Use VSS if available; fall back to internal driver (default) |
| `--ForceVSS` | Abort if VSS fails |
| `--AddDriver` | Inject additional storage driver (see §3.2.3) |
| `--MergeIDE` | Inject IDE/SATA driver into restored Windows (see §3.2.3) |
| `--DeleteSchedule` | Cancel any pending “restore-on-reboot” task |
| `--NoGUI` | Run without any graphical interface (pure console mode) |
| `--LogFile:<path>` | Specify a log file for all actions |
| `--Pwd:<password>` | Supply password for encrypted image |
| `--EncPwd:<password>` | Encrypt and save password (use with caution) |
| `--NoPwd` | Disable encryption |
| `--AllWriters` | Force inclusion of all VSS writers |
| `--IncludeWriter:<name>` | Include specific VSS writer |
| `--ExcludeWriter:<name>` | Exclude specific VSS writer |
| `--NoIcons` | Suppress tray icons during execution |
| `--NoBeep` | Disable beep notification after completion |

---

### Example: Basic Full Backup

```bash
snapshot C: D:\Backups\Full_$date.sna -L2000 -W
```

Creates a full image of drive **C:** to `D:\Backups\Full_<date>.sna`, splitting the file every 2 GB and generating a `.hsh` file for differential backups.

---

### Example: Differential Backup

```bash
snapshot C: D:\Backups\Diff_$date.sna -hsh:D:\Backups\Full_20250102.hsh
```

Creates a differential image containing all changes since the full backup made on January 2, 2025.

---

### Example: Restore from Image

```bash
snapshot restore HD1 auto D:\Backups\Full_20250102.sna
```

Restores all partitions from the specified full backup onto **disk 1** automatically.

---

### Example: Verify Image

```bash
snapshot -t D:\Backups\Full_20250102.sna
```

Verifies the integrity of the backup by comparing stored checksums with recalculated ones.

---

### Example: Mount an Image as Drive Z:

```bash
snapshot -x D:\Backups\Full_20250102.sna Z:
```

Mounts the image file as a **read-only** virtual drive `Z:`.

Unmount later with:

```bash
snapshot --unmount:Z
```

---

## 5.2 Scheduled Restores (Restore-on-Reboot)

When you attempt to restore the **active system partition** from within Windows, Snapshot offers to schedule the restore for the next reboot.

You can also schedule or cancel this behavior from the command line.

### Schedule a Restore

```bash
snapshot restore C: D:\Backups\System.sna --Reboot
```

Schedules a system restore at the next boot.

### Cancel a Scheduled Restore

```bash
snapshot --DeleteSchedule
```

Removes any pending restore task.

---

## 5.3 Automatic Full-Disk Backups

Snapshot can back up all partitions on a given disk automatically.

Example:

```bash
snapshot --BackupAll HD1 D:\Backups\Disk1_$date.sna
```

-   Creates images of all partitions on disk 1.
    
-   Appends the `$disk` variable to filenames automatically.
    
-   Equivalent to selecting all partitions manually in the GUI.
    

---

## 5.4 Network Backups via (S)FTP

You can back up directly to an FTP or SFTP server using a **URL-like syntax**.

### FTP Example

```bash
snapshot C: ftp://user:pass@backupserver/path/System_$date.sna
```

### SFTP Example (with PuTTY key)

```bash
snapshot C: sftp://user@server:22/path/System_$date.sna -KeyFile:C:\Keys\backup.ppk
```

### Notes

-   Ports are optional (default: 21 for FTP, 22 for SFTP).
    
-   Passwords stored in the Registry are AES-encrypted.
    
-   Differential images from FTP locations do not automatically create hash files — specify one manually using `-o`.
    

---

## 5.5 Example Batch Script

Here’s a simple Windows batch script (`DailyBackup.cmd`) for automatic daily system backups:

```bat
@echo off
set SNAPSHOT="C:\Tools\snapshot.exe"
set DEST=D:\Backups\System_%date:~-4%%date:~3,2%%date:~0,2%.sna

%SNAPSHOT% C: %DEST% -L2000 -W -Go --Pwd:MyStrongPassword --NoIcons --NoBeep
```

This script:

-   Runs silently
    
-   Splits files every 2 GB
    
-   Creates a hash file
    
-   Encrypts the image
    
-   Runs without notifications
    

Schedule it via **Windows Task Scheduler** for unattended operation.

---

## 5.6 Error Codes

When run in silent or scripted mode, Snapshot returns exit codes you can check for success or failure.

| Code | Meaning |
| --- | --- |
| `0` | Success |
| `1` | Backup aborted |
| `2` | Restore aborted |
| `3` | I/O error |
| `4` | Insufficient disk space |
| `5` | Access denied |
| `6` | Invalid parameters |
| `10` | Image file corrupted |
| `99` | Unknown error |

You can use these codes in scripts to trigger notifications or retries.

Example:

```bat
if errorlevel 1 echo Backup failed! >> log.txt
```

---

# 6 Appendix and Technical Notes

This chapter summarizes important background information, technical behavior, and additional usage notes about Drive Snapshot.

---

## 6.1 Image File Structure

A Snapshot image file (`.sna`) contains multiple data sections:

1.  **Header**
    
    -   Includes version, date, source partition info, and checksums.
        
    -   Identifies whether it’s a full or differential image.
        
2.  **Partition Metadata**
    
    -   Stores partition size, cluster size, filesystem type, and layout.
        
    -   Allows exact reconstruction of the partition during restore.
        
3.  **Data Blocks**
    
    -   Contains all used clusters or sectors, optionally compressed.
        
    -   Each block has its own checksum for integrity verification.
        
4.  **Footer**
    
    -   Contains end-of-file marker, hash index, and global CRC.
        

For **differential images**, Snapshot references a `.hsh` file instead of a full image to determine which clusters have changed.

---

## 6.2 Compression

Snapshot compresses data using a proprietary algorithm optimized for speed and reliability.

-   Compression is **lossless** — every byte can be restored exactly.
    
-   Average compression ratios:
    
    -   System partitions: **2:1 to 3:1**
        
    -   Data partitions with already compressed files (e.g., videos, ZIPs): **1.1:1** or lower
        
-   You can combine Snapshot images with third-party compressors, but that rarely saves space further.
    

> ⚙️ Compression and encryption can be combined freely.

---

## 6.3 Encryption

Drive Snapshot uses **AES-256** in CBC mode for secure encryption.

-   Encryption is performed **on the fly** during backup.
    
-   Each block is encrypted independently, ensuring data integrity even in case of partial damage.
    
-   The key is derived from your password using a **strong KDF (Key Derivation Function)** with salt.
    

### Notes on Password Handling

-   If you use `--StoreEncPwd`, Snapshot saves a derived key (not the plaintext password) in the Registry.
    
-   This key can decrypt any image created with that password.
    
-   The `--StoreDecPwd` option saves the full password (encrypted), allowing automatic decryption on that machine.
    
-   For maximum security, **do not use** `--StoreDecPwd` on shared systems.
    

---

## 6.4 Checksums and Image Verification

Every data block inside the image has an **embedded checksum** (CRC32).  
When testing or restoring, Snapshot recomputes and compares these checksums to detect corruption.

You can test images at any time using:

```bash
snapshot -t image.sna
```

If the file is damaged (for example, by media errors or transmission issues), Snapshot will immediately report the block where the error occurred.

---

## 6.5 Incremental and Differential Strategies

Snapshot does **not** perform incremental backups (i.e., chaining multiple differentials).  
Instead, it uses the more robust **differential** method:

-   Each differential image compares the current disk state directly against the last **full backup**.
    
-   This avoids dependency chains and minimizes data loss risk.
    

Recommended routine:

| Type | Frequency | Purpose |
| --- | --- | --- |
| Full backup | Weekly | Complete snapshot of the disk |
| Differential | Daily | Quick update capturing all recent changes |

When storage space is limited, delete old differential images periodically, but always retain at least one full image.

---

## 6.6 Restoring to Larger or Smaller Disks

-   If the **target disk is larger**, Snapshot allows automatic partition expansion for NTFS volumes (see Section 3.2).
    
-   If the **target is smaller**, restoration is only possible if:
    
    -   The total *used* space of the image fits within the smaller disk, **and**
        
    -   No partition begins beyond the smaller disk’s end sector.
        

In this case, disable “Restore Partition Structure” and manually adjust partition sizes before restoring.

---

## 6.7 Compatibility and Portability

Snapshot image files are **hardware-independent**:

-   They can be restored to any disk or machine that supports the same filesystem type.
    
-   Images are backward compatible — newer versions of Snapshot can always restore older images.
    
-   The program is **fully portable**: it runs without installation, requires no registry entries, and leaves no trace after removal.
    

You can freely copy `snapshot.exe` to USB drives or network shares.

---

## 6.8 Limitations

| Limitation | Description |
| --- | --- |
| **Linux file systems** | EXT2/3/4, ReiserFS, and XFS are supported only for *used-block* backups — file-level restore is not available. |
| **BitLocker volumes** | Backups capture encrypted data as-is. You must unlock the drive after restore. |
| **Dynamic disks** | Fully supported, but individual volumes cannot be resized automatically. |
| **VSS limitations** | VSS Writers may fail under load; Snapshot retries automatically, but logs warnings. |
| **Network restore** | Full-disk restores from FTP/SFTP sources are not supported. |

---

## 6.9 Logging and Diagnostics

Snapshot logs all activities and events when logging is enabled.

### Enabling Log Files

Use either of the following:

-   From GUI: **Advanced Options → Enable Log File**
    
-   From CLI:
    
    ```bash
    snapshot --LogFile:C:\Logs\snapshot.log
    ```
    

### Log Contents

-   Timestamp
    
-   Operation type (backup, restore, test, etc.)
    
-   Source and destination paths
    
-   Compression ratio
    
-   Throughput speed (MB/s)
    
-   Warnings or error messages
    

This log can be invaluable for automated systems or troubleshooting.

---

## 6.10 Performance Optimization

To achieve maximum throughput:

-   Use **local SSDs** or fast NVMe drives for both source and target.
    
-   Avoid network drives unless absolutely necessary.
    
-   Use the `-L` option to split files in 2–4 GB chunks for faster writes.
    
-   Disable antivirus scanning on `.sna` files to avoid I/O slowdowns.
    
-   Schedule backups during low system load (e.g., nighttime).
    

Expected throughput:

| Medium | Typical Speed |
| --- | --- |
| SATA SSD | 300–450 MB/s |
| NVMe SSD | 500–1000 MB/s |
| HDD | 100–150 MB/s |
| Network (1 Gbit) | 70–100 MB/s |

---

## 6.11 License and Distribution

Drive Snapshot is **commercial software**.

### Licensing Notes

-   Each purchased license is valid for **one computer**.
    
-   Licenses are perpetual — no subscription or expiry.
    
-   Upgrades within the same major version (e.g., 1.4x → 1.5x) are free.
    

### Portable Use

You can copy your licensed `snapshot.exe` to another system —  
the embedded license data travels with the executable.

If you use a separate license file, ensure it’s in the same directory as `snapshot.exe`.

For details, visit  
👉 https://www.drivesnapshot.de

---

## 6.12 Contact and Support

**Tom Ehlert Software e.K.**  
Aachener Str. 1057  
50858 Köln (Cologne), Germany

📧 **Email:** support@drivesnapshot.de  
🌐 **Website:** https://www.drivesnapshot.de

Typical support inquiries:

-   Licensing or activation
    
-   Troubleshooting VSS errors
    
-   Questions about command-line automation
    
-   Compatibility feedback
    

---

## 6.13 Version History (Summary)

| Version | Notes |
| --- | --- |
| 1.32 | Added differential images |
| 1.36 | Introduced (S)FTP support |
| 1.40 | Added AES-256 encryption |
| 1.44 | Full-disk restore and automatic partition resize |
| 1.48 | Added ReFS and EXT4 support |
| 1.50 | Optimized VSS handling, improved performance, and Windows 11/Server 2022 support |

---

### End of Manual

**Drive Snapshot v1.50 — Complete User Guide (Markdown Edition)**  
Paraphrased and reformatted for clarity and readability.  
© Tom Ehlert Software e.K.