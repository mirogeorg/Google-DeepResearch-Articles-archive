
# Drive Snapshot Manual

**Program Version 1.50**  
© Tom Ehlert Software e.K.  
[www.drivesnapshot.de](http://www.drivesnapshot.de/de/Handbuch.pdf)

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

A snapshot image contains the exact state of a hard drive at the moment the backup starts.
This means that the snapshot records the current condition of the drive at the beginning of the backup.
The backup is then created from that state, saving all occupied clusters on the drive.
Any changes made during the backup process will not be included in the image later.
With an image of all partitions, you can restore the system to a new hard drive after a total disk failure.
After a successful restoration, the drive will contain exactly the same data as it did at the time of the backup.

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
    
3.  Snapshot can encrypt the image while it is being created.
    The encryption algorithm used is AES256.

    To enable encryption, enter your password in the password and verify password fields.
    When you click the Store Encryption Password button, a key is generated from this password and stored in the Windows Registry.
    This means that all subsequent images created on this computer will also be encrypted with the same password.

    If you additionally click the Store Decryption Password button, the password will be stored in encrypted form in the Registry.
    This allows you to mount an image encrypted with this password on this computer without having to enter the password manually — Snapshot retrieves it automatically from the Registry.
    However, if the image is to be mounted or restored on another computer, the correct password must be entered.

    Storing the key used to encrypt the image is secure and irreversible.
    However, storing the decryption password is not completely secure.
    Even though the password is stored in encrypted form, it is theoretically possible to reconstruct the original password with sufficient effort.
    You should therefore carefully consider whether you really want to use this option.
    

---

## 2.2 Creating a Differential Image

A differential backup contains all the changes that have occurred on the partition since the corresponding full backup.
The differential backup is created by reading the entire contents of the partition to be backed up.
From the data read, hash values are calculated and compared with the hash values stored in the hash file that was generated during the full backup.
If a difference is detected, the corresponding block is written to the differential image.

To create a differential image, only the hash file with the .hsh extension is required.
The full image itself does not need to be accessible when generating the differential image.
---

### 2.2.1 Differential Backup Using the GUI

When creating a differential image, proceed initially in the same way as when creating a full image.
At the point where you specify the filename for the differential backup (see Figure 2.2), choose a name for the differential backup that must differ from the name of the full backup.

It is helpful to choose a name that makes it clear which full image it belongs to and that it is a differential backup.
For example, if you named the full backup MeinImagevonC.sna, a suitable name for the differential backup would be MeinImagevonC-diff.sna.

You can also use all $–variables described in Section 2.6 in this name.
After choosing the name for the differential backup, check the Differential Image checkbox.
This will activate the input box below, where you specify the hash file that was generated when creating the full backup on which the differential backup will be based.

Then, click the Start Copy button to start the differential backup.
    
---

## 2.3 Using VSS (Volume Shadow Copy Service)

The Volume Shadow Copy Service (VSS) in Windows has been available since Windows XP/2003.
It is a service that applications can interact with to create a shadow copy of one or more partitions.

Each application is free to cooperate with this service to perform specific actions during a backup process.
These components that communicate with VSS are called VSS Writers.
Writers can perform actions before and after a backup operation.

You can view a list of all writers present on your system with the following command:

1.  **VSS without writers**
    
    A shadow copy is created that simply contains a copy of the existing data.
    The VSS Writers are not explicitly invoked and therefore do not perform any specific operations.

    In this mode, for example, the transaction logs of an Exchange Server are not truncated after the backup, and a Hyper-V host does not notify running virtual machines about the planned backup.

    Snapshot automatically uses this mode whenever more than one partition is to be backed up at the same time, or when a running Exchange Server is detected on one of the partitions included in the backup.
    
2.  **VSS with writers**
    In the second method of using VSS, all participating VSS Writers are explicitly informed about the upcoming backup.
    This allows the writers to perform special operations before and after the backup.

    For example, in this mode, a Hyper-V server with a virtual machine that does not have integration services running would temporarily pause that machine at the beginning of the backup.
    Immediately after the shadow copy is created, the virtual machine automatically resumes operation.
    An Exchange Server, on the other hand, would delete the transaction logs after a successful backup.

    Which actions the writers perform is entirely up to them — Snapshot has no control over what operations the writers execute.

    Snapshot uses this mode only when explicitly instructed.
    In the Advanced Options dialog (see Figure 2.5), you can force this mode by checking “Explicitly include all VSS writers.”
    This ensures that all VSS writers running on the system are notified of the upcoming backup.

    If you require more fine-grained control over which writers are included, you can use the command line options (see Chapter 5, IncludeWriter and ExcludeWriter).
    This may be necessary, for example, if you already use a dedicated Exchange backup solution and do not want Snapshot to trigger deletion of the transaction logs.

---

## 2.4 Backup Without VSS

When performing a backup without VSS, Snapshot uses an internal driver to keep the image consistent.
All write operations to the partition being backed up are temporarily delayed until the driver has secured the data currently present on the disk.

The image therefore contains only the data that existed on the drive at the moment the backup started.
Any changes that occur while the image is being created are not written to the image.

The internal driver cannot perform simultaneous backups of multiple partitions.
If you have interdependent data distributed across several partitions, you must perform the backup using VSS (see Section 2.3).

However, backups without VSS allow Snapshot to be used on systems that do not have the Volume Shadow Copy Service,
such as all operating systems released before Windows XP/2003.

In the Advanced Options dialog (see Figure 2.5), you can force a backup without VSS by selecting the option “Never use VSS.”

---

## 2.5 Using an (S)FTP Server

Snapshot can also save images to and restore them from an (S)FTP server.
This is done by entering a specially formatted filename.

For an FTP server, the filename has the following format:

ftp://username:password@server:port/path/filename.sna


For an SFTP server, use this syntax:

sftp://username:password@server:port/path/filename.sna


If you have already configured an FTP server in Snapshot, you don’t need to include the password in the filename — it will be inserted automatically.
Passwords are stored encrypted (AES) in the Windows Registry.
Specifying the port is optional.

When using an SFTP server, you can use a PuTTY-generated key file instead of a password.

Please note that when creating a full image on an (S)FTP server, no hash file is generated.
If you want to create a hash file, you must specify the path and filename for a local hash file using the -o option.

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

In the Advanced Options, you can adjust several parameters that are used during image creation.
Figure 2.5 shows the corresponding dialog.

With the VSS options, you can control how the snapshot is created.
By default, the Volume Shadow Copy Service (VSS) is used only if a running Exchange Server or Domain Controller is detected, or if more than one drive is to be backed up.

If --NoVSS is selected, Snapshot uses its internal driver to create the backup.
If --UseVSS is selected, Snapshot will always try to use the Volume Shadow Copy Service first, and only fall back to the internal driver if an error occurs.

With the --ForceVSS option, you can enforce the use of VSS.
If an error occurs while creating the shadow copy, Snapshot will abort the process with an appropriate error message.

This mode is recommended whenever you need to back up multiple drives simultaneously.
With the VSS Writer Options, you define the mode used when creating the shadow copy.
A description of the modes can be found in Chapter 2.3.
This field corresponds to the command-line option --AllWriters.

By checking “Beep when operation has completed,” Snapshot will play a sound when the backup is finished.

By checking “Create directory if not existing,” Snapshot will automatically create the directory where the image is to be saved if it does not already exist.
This corresponds to the command-line option --CreateDir.

By checking “Enable log file” and entering a valid name for the log file, you can enable logging.
Please note that the log file will only be created the next time Snapshot is started.
After entering the log file details, you must click “Save these settings as the default.”
This field corresponds to the command-line option --LogFile:<LogFileName>.

By checking “Save the complete disk, including unused sectors,” Snapshot will also back up the unused areas of a partition.
This mode is usually unnecessary and is primarily used in computer forensics.

In the field “Maximum image single file size,” you can specify the maximum size (in megabytes) of a single image file.
If the value is set to 0, Snapshot creates a single image file.
This field corresponds to the command-line option -L.
Please also refer to the notes on this option in Section 5.5.

In the field “Automatic Backup of small boot partitions,” you can specify the size limit up to which partitions are automatically backed up, even if they are not explicitly selected for backup.
If you set this value to 0, no partitions are backed up automatically.
The default value is 512 MB, which ensures that the “System Reserved” or UEFI partition is automatically included in standard installations.
For a complete backup, you must make sure to back up all partitions that Snapshot detects on the disk.

In the Hash File Options section, checking “Always generate hash file” enables the creation of a hash file during a full backup.
This option is enabled by default and should only be unchecked if you do not plan to create differential images.

With the button “Create hash file from full image,” you can generate a hash file from an existing full image.

The button “Use these settings as the default” saves your current configuration.
Until you make further changes, these settings will be applied to all backups performed using the graphical interface.

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

Restoring with Snapshot always follows the same procedure.
First, you restore the partition structure from one of the images to the target hard drive, which must be at least as large as the original drive.
After that, all partitions (images) are restored.

Starting with version 1.44, Snapshot also allows you to restore entire drives.
In this case, the steps mentioned above are performed automatically in sequence.

Automatic full-disk restoration is available only under Windows —
the DOS version of Snapshot does not support this feature.

---

## 3.1 Restoring Under DOS

If you have Snapshot installed on your computer, you will find an entry for it in the Start menu under Programs.
There, you’ll see an option called “Create disaster recovery diskette.”
This allows you to create a bootable floppy disk that can be used as a boot environment for restoring an image.

If you no longer have a floppy drive, you can instead create a bootable CD.
To do this, open a command prompt and navigate to the directory where Snapshot is installed.
Then run the following command:

snapshot -X networkbootdiskette.sna bootimage.img


This command creates a new file (bootimage.img), which you can specify as the boot image in your CD burning software.

After booting the system from the created disk, you can restore the image using Snapshot.
First, display all drives detected by Snapshot using the command:

snapshot show


From the output, determine the target drive for your image, and restore the partition structure with:

snapshot restore HD1 partitionstructure W:\C-DRIVE.SNA


Replace HD1 with the actual target drive identified by the show command.

After restoring the partition structure, restore the individual partitions with:

snapshot restore HD1 auto W:\C-DRIVE.SNA


This step must be repeated for each partition with its corresponding image file.

For more information on restoring with the DOS version of Snapshot, please visit the official documentation at:
http://www.drivesnapshot.de/de/restdos.htm

---

## 3.2 Restoring Under Windows

For restoring, you can use any Windows PE environment instead of the FREEDOS system provided by Snapshot.
A standard Windows 7 DVD can also be used as a boot medium for restoration.

The advantage of restoring under Windows PE rather than DOS is that you can use the graphical user interface of Snapshot.
Section 6 shows several screenshots illustrating the boot process using the Windows 7 DVD.

Once the system has booted, you can start Snapshot.
Restoration is especially convenient if you have previously copied snapshot.exe into the same directory where your image files are stored—this way, you avoid having to type long paths.

After starting Snapshot, choose “Restore Disk from File.”
If the image is stored on an FTP server, please first refer to Section 3.2.1.

In this dialog (see Figure 3.1), you will see information about the image that was recorded during the backup creation.
By clicking the “Test Image” button, you can check the image for consistency.
During backup, Snapshot calculates checksums that are stored in the image.
When testing the image, these checksums are recalculated from the read data and compared with the ones stored in the image.
If all checksums match, the image is error-free, and all data is exactly as it was at the time of backup.

Once you’ve selected your image and, if desired, tested it, click “Next” to reach the dialog shown in Figure 3.2.
At the bottom of the window, move your mouse pointer over the disk to which you want to restore the data.
Right-click to open the popup menu and select “Restore Partition Structure” to write the entire partition structure (which is included in every image) to the target disk.

The partition structure also includes the Master Boot Record (MBR), which does not need to be restored separately.
After restoring the partition structure, all partitions will be created on the target drive with exactly the same offsets as on the original.

If you restore the partition structure to a larger disk, you can click “Grow all NTFS Partitions…” to make Snapshot expand all NTFS partitions so that the entire disk space is used.
Only partitions larger than 1 GB will be resized.

You can now restore each image into its corresponding partition.
Once all partitions have been restored, the target disk will be in exactly the same state as the original disk at the time of backup.

If there is still unpartitioned space left on the disk, you can use Snapshot to create new partitions or logical drives.
To do this, right-click in the unpartitioned free space area of the disk (see Figure 3.3).

---

### 3.2.1 Restoring from an (S)FTP Server

When restoring an image that is located on an FTP server, the procedure is basically the same as when restoring an image stored on a local drive.
The only difference is the filename, which must include the FTP path.

---

### 3.2.2 Restoring an Entire Disk

Since version 1.44, Snapshot also supports restoring an entire hard drive, provided that all partitions were backed up simultaneously during the backup process.
All partition images must be located in the same directory.

Images stored on an FTP server and full-disk restores during system reboot are not supported in this mode.

To restore an entire hard drive, select “Restore complete Disk from Images” in the main window of Snapshot (see Figure 3.4).
After making this selection, the dialog shown in Figure 3.5 will appear.
In this dialog, you first select one of the images to be restored.
Immediately after making your selection, all related images that belong to the same backup set will appear in the list below.

If you do not want to restore some of these images, you can deselect the corresponding checkboxes.
However, in most cases, all images should be restored to ensure that the hard drive is bootable.

After selecting the images, you need to specify the target disk.
You can do this by clicking on the desired drive in the lower section of the window.
Once a drive is selected, Snapshot will display the current partition layout of that device in graphical form.

Finally, you must decide whether to also restore the partition structure.
Use the checkbox in the lower-left corner of the dialog for this purpose.
By default, this option is enabled, meaning the partition structure will be automatically restored before the image restoration process begins.
If the target disk is unpartitioned, this step is mandatory.
You should only disable this option if you are certain that the disk already has the correct partition structure.

Once all parameters are set, start the restoration process by clicking the Next button.
A final confirmation prompt will appear, showing the target drive once more.
If you are sure that all data on the target drive should be overwritten, confirm by clicking OK.

The restoration process will then begin, and a new dialog will appear showing the progress and an estimated time remaining for the restoration.

---

### 3.2.3 Restoring to Different Hardware

When restoring to different hardware, you must add the driver for the new disk controller to the restored system after the restoration has completed successfully.
This process can be performed directly with Snapshot, but it requires a Windows operating system as a base and cannot be done using the FREEDOS boot disk.

Adding a driver can only be successful if the corresponding hardware is currently present in the system.

If the controller is an IDE controller or a SATA controller operating in IDE mode, start Snapshot with the following command line:

snapshot --mergeide


A dialog will then appear asking you to specify the Windows directory of the restored operating system.
After entering it, click OK.
If the command executes successfully, Snapshot will display a confirmation message.

If the controller is not an IDE controller, you will need the corresponding .inf and .sys files for the correct driver.
In this case, start Snapshot with the following command line:

snapshot --AddDriver


You will then be guided through three dialogs in sequence, where you must specify:

The path to the restored operating system,

The path to the .inf file, and

The path to the .sys file.

If the operation is successful, you will see a confirmation message.

If you receive an error message such as:

Can’t find a device with matching PCI Id.


this means you are using the wrong driver or at least an incorrect .inf file.

---

## 3.3 Restoring During System Reboot

If your operating system is still bootable and you want to restore the system partition that is currently in use, Snapshot offers an additional method besides the conventional restore process — it can perform the restoration during the next system reboot.

To do this, proceed exactly as you would for a normal restore operation (see Section 3.2).
At the point where the restoration would normally begin, the dialog shown in Figure 3.6 will appear.
In this dialog, you can decide whether the system partition should be restored during the next reboot and specify how Snapshot should behave in case of errors.

The image files must be available on a local hard drive, and they must not be located on the partition that is to be restored.

Once you continue by clicking Yes, you will be asked whether you want to restart the computer immediately.
During the next boot process, Snapshot will automatically perform the restoration.

If you accidentally scheduled a restore operation, you can remove it by following the same steps as before but then clicking No (as shown in Figure 3.6).

Restoring during reboot can also be managed via the command line —
details on how to do this can be found in Chapter 5.2.

---

# 4 Mounting an Image as a Virtual Drive

Mounting an image as a virtual drive allows you to copy individual files or directories from the image without having to restore the entire image first.

By clicking “Mount Image as Virtual Disk” on Snapshot’s start screen, you can mount an image as a virtual drive.
The dialog shown in Figure 4.1 will appear.
In this dialog, select the image file you want to mount and choose the drive letter to assign to the virtual drive.

Click “Map Virtual Drive” to mount the image as a virtual drive.
If you instead click “Map and Explore Virtual Drive,” Snapshot will mount the image and then automatically open the Windows File Explorer for the new drive.

In both cases, you’ll get a new drive (for example, Z:), which you can access just like any other disk in Windows.

If Snapshot is installed, you can also mount an image simply by double-clicking a .SNA file.
Both full and differential images can be mounted as virtual drives in exactly the same way.

By clicking the “Set Snapshot as .SNA viewer” button (as shown in Figure 4.1), you can associate the .sna file extension with the Snapshot program.
This association is automatically set up during a full installation using Setup.exe,
but if you’re using Snapshot without installation, this option allows you to manually create the link —
so you can easily mount images as virtual drives by double-clicking them in Windows Explorer.

# 5 Command Line Reference

All functions of Snapshot can also be controlled via the command line.
This allows Snapshot to be fully automated through scripts.
Such scripts can then be executed on a schedule using the Windows Task Scheduler.

The available command-line options are divided into two groups:

Basic options, which begin with a single “-”, and

Advanced options, which begin with two “--”.

---

## 5.1 Command Line Options

A command line for backing up one or more partitions always follows this structure:

snapshot Source Target Options


Example:

snapshot C:+D: X:\image-$disk.sna --LogFile:C:\snapshot.log


The source can consist of one or more drive letters,
or it can be a specific identifier that defines the disk and partition without using a drive letter.
The following syntax applies:

HD1:1  → first partition on the first hard drive  
HD2:1  → first partition on the second hard drive  
HD1:2  → second partition on the first hard drive  
HD2:2  → second partition on the second hard drive  
HD1:*  → all partitions on the first hard drive


You can also specify a drive letter instead of a disk number.
At runtime, Snapshot automatically replaces the drive letter with the number of the disk containing that partition.
For example:

HDC:*


means “back up all partitions on the disk that contains drive C:”.

A special identifier, HDWIN, refers to the disk that contains the currently running operating system.
At runtime, HDWIN is replaced by HD1, HD2, etc.
This allows you to back up the entire system drive with a single command:

snapshot HDWIN:* X:\image-$disk.sna


Multiple partitions can be combined using the “+” symbol.
Whenever more than one partition is backed up in a single command, you must use the $disk variable in the image filename.
At runtime, Snapshot replaces this variable with a unique identifier — usually the drive letter of the partition.

When backing up multiple partitions, Snapshot will first try to use the Microsoft Volume Shadow Copy Service (VSS).
A shadow copy of all partitions included in the backup is created simultaneously,
and Snapshot then backs up each shadow copy sequentially.

If you choose to use the internal driver instead of the VSS service when backing up multiple drives,
the snapshots are created one after another.
That means Snapshot backs up the first partition, then takes a snapshot of the next, and so on.

⚠️ Important:
If your data is distributed across multiple partitions and depends on cross-partition consistency,
this sequential mode may result in an inconsistent image.
In such cases, you should always use the VSS service.

---

## 5.2 Scheduled Restores (Restore-on-Reboot)

The command line for a restore is very similar to that of a backup —
the only difference is that the source and destination are reversed.

For example:

snapshot X:\image.sna D:


restores the image file X:\image.sna to the partition with the drive letter D:.

Restoring an Entire Disk

Restoring a complete hard drive is also possible, provided that:

The backup was created with Snapshot version 1.44 or newer, and

All partitions were backed up simultaneously.

Example command:

snapshot X:\image-C.sna HD4 --entiredisk


In this example:

image-C.sna is the image of any partition from the backed-up disk.

HD4 is the target disk (Snapshot numbers disks starting from 1).

When this command runs, Snapshot first restores the partition structure,
then proceeds to restore all associated images.
After successful completion, the restored disk’s data will be in exactly the same state as the original disk at the time of the backup.

⚠️ The --entiredisk option cannot be combined with the --schedule option.

Scheduled Restore at Next Reboot

If you want to restore the system partition during the next reboot, you can also do this via command line:

snapshot --schedule C: X:\image.sna


This schedules the image X:\image.sna to be restored to drive C: during the next system restart.

⚠️ The image file must be located on a locally accessible drive (not on a network or FTP share).

---

## 5.3 Automatic Full-Disk Backups

Images can also be mounted via the command line in Snapshot.

To do this, start Snapshot with the following command:

snapshot X:\image.sna


This command mounts the image X:\image.sna using the default drive letter you have configured in the graphical interface.

If you want to use a different drive letter, you can specify it in the command line, for example:

snapshot X:\image.sna Z: -v


This mounts the image as drive Z: and automatically opens a Windows File Explorer window for it.

If you only want to mount the drive without opening Explorer, use the -vm option instead of -v:

snapshot X:\image.sna Z: -vm


You can also use the -vq option to make Snapshot wait until the image is explicitly unmounted.
Unmounting is done with the following command:

snapshot --unmount


This removes all mounted virtual drives.

If you have multiple images mounted but want to unmount only a specific drive,
you can specify the drive letter after the --unmount option.
For example, to unmount only drive Z:, use:

snapshot --unmount:Z

---

## 5.4 Network Backups via (S)FTP

A hash file can be created afterwards from any full image.
To do this from the command line, use the following command:

snapshot X:\FullImage.sna -hHashFileName.hsh


This command creates a new hash file named “HashFileName.hsh” from the image “FullImage.sna.”
The generated hash file can then be used later to create a differential image.

---

## 5.5 Basic options

All basic options in Snapshot begin with a single hyphen (“-”).
The following list describes all available basic command-line options:

General Options

-?
Displays a short help overview of all available basic options when Snapshot is started.

-A
Backs up all sectors of the partition, including those not currently used by the file system.

-L650
Splits the image into parts of 650 MB each.
You can replace 650 with another value.
A value of 0 (zero) means that only one single image file will be created.

⚠️ Using a single image file may cause issues.
If the backup fails with an error like
“5aa – Not enough system resources to complete the requested service,”
split the image into smaller parts.

-T
Tests an image file.
During backup, Snapshot writes checksums into the image; during testing, these are recalculated from the read data and compared with the stored ones.
If they don’t match, Snapshot reports an error.

-W
Suppresses the “Hit any key to continue” prompt that appears when Snapshot is run from the command line without a graphical interface.

-R
Empties the Recycle Bin before starting the backup.

Graphical Interface Options

-G
Opens the graphical user interface (GUI) even when Snapshot is started from the command line.

-Go
Opens the GUI and automatically closes Snapshot after the backup completes successfully.

-Gx
Same as -Go, but Snapshot also closes automatically even if an error occurs.

-Gn
Opens the GUI and prevents the user from aborting the running backup.

-Gm
Opens the GUI in minimized form when started from the command line.

Encryption Options

-PW=MyPassword
Encrypts or decrypts the image using the specified password (in this example, MyPassword).

Avoid characters interpreted by the command processor (e.g., the % sign).

-pwgen=KeyFileName
Generates an encryption key file from the given password and saves it as the specified file.
This file can later be used with -pwuse to encrypt images,
so you no longer need to include a plaintext password in the command line.

Must be used together with -pw=YourPassword.

-pwuse=KeyFileName
Encrypts images using the key stored in the specified file (created via -pwgen).
The original password cannot be reconstructed from this file.

Additional Options

-C="My comment"
Stores the given text as a comment inside the image file.
This comment appears in the image information dialog later.

-oPath\FileName.hsh
Creates a hash file alongside the full image, using the specified path and filename.
This option is required when creating a backup over FTP if you plan to later create a differential image.

-o
If used without a filename, no hash file will be created.
Use this option to save disk space if you don’t plan to create differential images.

-hPath\FileName.hsh
Creates a differential image, using the given hash file to reference the original full image.

Variables

$ Variables
Certain strings beginning with a $ symbol are replaced by Snapshot at runtime.
All supported variables and their meanings are listed in Table 2.1 of the manual.

-I
Displays information about an image file (which must be specified in the command line).
The output includes:

Date and time of backup

Computer name

Drive letter

Volume label

File system

Size

Comment

-P
Displays partition information of the disk from which the image originated.
An image filename must be provided.

-Y
Automatically answers “Yes” to all confirmation prompts during execution.

---

## 5.6 Extended options

All advanced options in Snapshot begin with two hyphens (“--”).
The following list describes all available advanced command-line options:

General Options

--?
Displays a short help overview of all available advanced options.

--Version
Outputs the current version number of Snapshot and then exits.

⚠️ This option cannot be combined with any other options.

Partition Management

--Activate X:
Sets the partition associated with drive letter X: as active (bootable).

--Deactivate X:
Removes the Active flag from the partition associated with drive letter X:.

Hardware / Driver Integration

--AddDriver
Adds a driver for a hard disk controller to a restored system.
You need the corresponding .sys and .inf files for your controller.
Paths to the restored Windows installation and driver files can be provided through graphical dialogs.

If you want to use this option in a script, you can specify all three paths directly on the command line in the following order:

Path to the Windows directory of the restored system

Path and name of the .inf file

Path and name of the .sys file

Example:

snapshot --AddDriver C:\Windows X:\iastor.inf X:\iastor.sys


--MergeIDE
Adds the IDE controller driver to a restored system.
For more details on restoring images to changed hardware, see Chapter 3.2.3.

Volume Shadow Copy (VSS) Control

--AllWriters
When used during backup, Snapshot employs the Volume Shadow Copy Service (VSS) in a mode that explicitly notifies all participating applications (VSS Writers) about the backup process.
This allows writers to perform necessary actions before and after the backup.

For example:

A Hyper-V server notifies all running virtual machines.

VMs without integration services are briefly paused during snapshot creation.

An Exchange server deletes its transaction logs after a successful backup.

(See also section 2.3 for more details.)

--ExcludeWriter:"WriterName1","WriterName2","WriterName3",…
Implies --AllWriters, but excludes the listed writers from the backup process.
For writer names and usage details, see Chapter 2.3.

--IncludeWriter:"WriterName1","WriterName2","WriterName3",…
Explicitly notifies only the listed writers during backup.
See Section 2.3 for information about writer names.

--novss
Disables the Microsoft Volume Shadow Copy Service (VSS).
Snapshot instead uses its internal driver to ensure data consistency.

--usevss
Instructs Snapshot to try using Microsoft’s VSS to create the snapshot.
If VSS fails (e.g., due to a service error), Snapshot automatically falls back to the internal driver.

--forcevss
Forces Snapshot to use Microsoft’s VSS service.
If VSS fails (for example, due to an error in the service), Snapshot aborts the process and displays an error message.

Restore Scheduling

--ListSchedule
Displays information about any restore operations scheduled for the next reboot.

--RemoveSchedule
Removes a scheduled restore operation that was set to run at the next reboot.

Reboot Behavior After Scheduled Restore

--autoreboot:off
When performing a restore during reboot, this option controls what happens afterward.
→ The computer does not restart automatically after the restore is completed.

--autoreboot:success
When performing a restore during reboot, this option controls what happens afterward.
→ The computer automatically restarts only if the restore completed successfully.

--autoreboot:any
When performing a restore during reboot, this option controls what happens afterward.
→ The computer automatically restarts regardless of whether the restore succeeded or failed.

Automatic Backup Options

--AutoBackupSize:XX
Defines the maximum partition size (in MB) for which Snapshot automatically performs a backup.
This automatic backup happens only once — if the image file already exists, the partition is not backed up again.

Specify the value directly after the colon.

The default value in the GUI is 512 MB, which ensures that small boot-related partitions (like System Reserved or UEFI partitions) are backed up automatically.

Setting the value to 0 disables this automatic backup feature, so only explicitly specified partitions are backed up.

When Snapshot is run from the command line, the default value is 0, meaning no automatic partition backups are performed.

Directory and Storage Options

--CreateDir
If used during a backup, Snapshot will automatically create the target directory where the image will be stored if it doesn’t already exist.

--DedupTarget
Use this option when the image is stored on a volume with Windows Server 2012 data deduplication enabled.
This option disables Snapshot’s internal compression and ensures data blocks are written in a consistent order across multiple backups — improving deduplication efficiency.

File and Directory Exclusions

--exclude:path1,path2,filename1,filename2
Excludes the specified files or directories from the backup.
Multiple paths or filenames can be separated by commas.

--exclude:@DoNotBackup.txt
Excludes all files and directories listed in the specified text file (in this example, DoNotBackup.txt).
Each file or directory path should be written on a separate line in the text file.

Execution and Media Handling

--exec:"Command"
Executes a command immediately after the snapshot is created.
This option must be the last argument on the command line.
The command to execute is enclosed in quotation marks.

Example:

snapshot C: X:\backup.sna --exec:"notepad C:\log.txt"


--EjectDriveAfterBackup
After the backup finishes, this option ejects a removable drive (e.g., a USB drive).
The drive will remain inaccessible until it is physically removed and reattached.

--EjectMediaAfterBackup
After the backup finishes, this option ejects removable media (e.g., an RDX cartridge or similar medium).

Error Handling and Performance
--ErrorOnBadSectors
Without this option, Snapshot continues the backup even if bad sectors are encountered and returns an exit code of 0 if no other errors occur.
If this option is used, Snapshot still continues the backup, but the exit code is 40 to indicate that bad sectors were found (as long as no other errors occurred).

--FastDiff:
This option can be used when creating a differential backup.
After the colon, provide a comma-separated list of paths that should be checked for changes only by comparing timestamps, file sizes, and cluster positions — the file contents themselves are not read.
Snapshot compares each file’s current timestamp and size against the corresponding values from the full image (which must still be available in its original path).
If timestamps, sizes, and file positions all match, Snapshot assumes the file is unchanged and skips reading its clusters.
If any of these values differ, the file is read and modified blocks are included in the differential image.
This option can significantly reduce the time needed for differential backups, especially for directories containing a few large files (such as virtual machine folders).
However, for folders with many small files, it may actually increase backup time.
Example:
snapshot V: X:\image-v.sna --FastDiff:"\Virtual Machines\*"

In this example, Snapshot checks all files in folders and subfolders containing the string “Virtual Machines” for changes in timestamp, size, and cluster positions — but does not read file contents unless differences are detected.

--FullIfHashIsMissing
When performing a differential backup, if the required hash file is missing, this option makes Snapshot automatically create a full backup instead.

FTP and Network Options
--FTPActive
When using an FTP server as the backup target, this option controls the connection mode.
By default, Snapshot tries to use passive FTP mode.
With --FTPActive, Snapshot instead uses active FTP mode to establish the connection.

--AddFTPAccount:Type,User,Server,Password,PortNumber
Adds a new (S)FTP account that Snapshot can use for backups or restores.
Once configured, you don’t need to include the password in future command lines.
Parameters:


Type – SFTP (for secure FTP) or FTP (for unencrypted FTP)


User – FTP/SFTP username


Server – Server address


Password – Password or path to a key file (for SFTP)


PortNumber – Optional (default FTP/SFTP port is used if omitted)


For SFTP servers, instead of a password, you can specify a PuTTY-format key file (.ppk).

--DeleteFTPAccount:Type,User,Server,Password,PortNumber
Deletes an existing (S)FTP account that was previously added with --AddFTPAccount.
The parameters follow the same format.
You only need to include the PortNumber if the account used a nonstandard port.

--ListFTPAccounts
Displays a list of all configured (S)FTP accounts currently stored by Snapshot.

Throttling and Logging
--LimitIORate:XX
Limits the maximum data write speed during backup operations.
Useful when backing up to slow network drives.
The value XX specifies the maximum write speed in megabytes per second (MB/s) and can range from 1 to 100.

--LogFile:LogFileName.log
Creates a log file with the specified name.
If Snapshot is run multiple times, the log is not overwritten — new messages are appended.

It’s strongly recommended to always use a log file when running Snapshot from the command line.
Avoid placing the log file on a network share, as Snapshot won’t be able to log errors if network issues occur.


Disk Error Analysis
--LocateSector
When using Snapshot’s internal driver, if your disk contains bad sectors, Snapshot creates an additional file named:
BadSectors_X.txt

(where X represents the drive letter) in the same directory as the image.
This file lists the sector numbers of the damaged areas.
Using this option, Snapshot can identify which files contain those bad sectors.
Example:
snapshot --LocateSector @BadSectors_X.txt

You can also perform this analysis using a mounted image instead of the original drive:


Mount the image (e.g., as drive Z:)


Run:
snapshot --LocateSector @BadSectors_X.txt Z:


This allows you to find the file names containing defective sectors even without access to the original hard disk.

Image Management and Optimization

--merge:NewFullImage.sna OldDiffImage.sna
This option lets you create a new full image (including its hash file) from an existing differential image.

This is particularly useful when backing up over slow connections or via FTP.

Example scenario:

Computer A is to be backed up to computer B, but the connection is too slow for full backups.

A full backup is created once and transferred manually to B.

After that, only differential backups are sent over the slow connection.

Over time, differential backups grow larger and exceed the available bandwidth.

Now, you can use the --merge option on computer B to create a new full image (with a new hash file) from the latest differential image.
You can then transfer only the new hash file back to computer A — allowing small differential backups to resume again.

Network Access

--NetUse:
Connects a network share as a network drive for Snapshot to access.
It supports both UNC paths and drive letters.

Syntax (UNC path):

--NetUse:\\Server\Share,Username,Password


Syntax (with drive letter):

--NetUse:\\Server\Share,X:,Username,Password


Here, X: is the assigned drive letter under which the share will be available in Snapshot.

Hash File Management

--NoLocalHashFile
By default, when creating a full backup, Snapshot stores the associated hash file in the same directory as the image.
If the image is stored on a network share, the hash file is first created locally (in the Temp folder) and copied to the share afterward.

Using this option disables that behavior — Snapshot writes the hash file directly into the same directory as the image,
restoring the behavior of earlier versions.

Output and Summary

--PrintTotals
At the end of a backup, Snapshot prints a summary of the backup data processed.
Example output:

11:22:52 Total size of all disks: 14,999 MB
11:22:52 Total used size of all disks: 5,147 MB
11:22:52 Total size of all images: 2,448 MB
11:22:52 Total time used: 0:15 minutes
11:22:52 Average throughput: 343 MB/s

Image Integrity and Repair

--QuickCheck:Image.sna
Performs a quick integrity check of an image file.
Only the image structure and the presence of all parts are verified —
the actual data blocks are not read or validated.
For a full consistency check, use the -T option instead.

--repair:NewImage.sna OldImage.sna
If an image is damaged, this command instructs Snapshot to attempt a repair by copying all valid data to a new image file.
The new image will have a correct internal structure and can be mounted as a virtual drive.

Whether Windows Explorer can access the mounted volume depends on how much data was lost.
Running CHKDSK on the mounted drive may help.
You’ll need roughly the same amount of free disk space as the size of the damaged image.

--repair OldImage.sna
A lighter repair method that requires less time and disk space.
Snapshot reads the entire damaged image and rebuilds new index information for all valid data,
appending this to the end of the image file.

The repaired image can then be mounted —
but only with Snapshot version 1.46 or later.

Licensing and Partition Management

--register:LicenseFile.txt
Registers Snapshot using a license file via the command line.
The file must contain the entire license key provided to you.

--resize X:
Resizes an NTFS partition, including its file system.
If called with only a drive letter (e.g., --resize C:), Snapshot displays the possible values for shrinking or expanding the partition.

If you specify a numeric value (in MB) after the drive letter, Snapshot attempts to resize the partition to that exact size.
Example:

snapshot --resize D: 50000


→ Resizes drive D: to 50,000 MB (≈50 GB).

Boot Validation

--Checkboot HD1
Checks whether the specified drive (e.g., HD1) meets the minimum requirements for being bootable.

Disk Signature and Structure Management

--ClearSignature HD1
Resets the disk signature of the specified drive (here HD1) to 00 00 00 00.
The disk signature is a 4-byte identifier stored in the Master Boot Record (MBR) that uniquely identifies each disk to the operating system.

--SetSignature HD1 01234567
Sets the disk signature of HD1 to the hexadecimal value 01 23 45 67.
This can be useful when restoring or cloning disks that must keep or change their signature to avoid conflicts.

--CleanDisk HD1
Erases the partition table (the first sectors) of the specified drive (HD1).

⚠️ This does not perform a full data wipe — it only removes partition information, effectively rendering the disk empty until repartitioned.

--RestoreMBR HD1 image.sna
Restores the Master Boot Record (MBR) from the specified image (image.sna) to the disk HD1.

⚠️ All existing data on the disk will be lost.

--RestorePartitionStructure HD1 image.sna
Restores the entire partition structure, including logical drives, from the specified image to HD1.

⚠️ All existing data on the disk will be lost.
Unlike --RestoreMBR, this command also restores extended and logical partition details.

--ExtendPartitions:Image.sna HD1
Restores the partition structure from the image to HD1 and then automatically expands all NTFS partitions larger than 2 GB to fill the remaining space of the disk.
Other partition types remain unchanged.

This is especially useful when restoring to a larger disk than the original.
⚠️ All existing data on the disk will be lost.

Image Mounting Behavior

--SearchFull:Path1,Path2,Path3,...
Specifies alternate search paths for Snapshot to locate the full image when mounting a differential image as a virtual drive.
If this option is not used, Snapshot searches first in the original location of the full image and then in the same directory as the differential image.

Encryption Password Management

--SetDefaultEncPwd=Password
Sets the default password for encrypting all future images.
Unlike older versions, Snapshot now stores only a derived encryption key (not the actual password) in the Windows registry.
This key cannot be reversed to retrieve the original password, making this method secure.

--SetDefaultDecPwd=Password
Sets the default decryption password used for decrypting encrypted images.
In this case, Snapshot stores the encrypted password (not the key) in the registry — meaning that with sufficient effort, recovery of the password is theoretically possible.

For maximum security, do not use this option — Snapshot will then ask for the password whenever you open an encrypted image.

--SetDefaultPwd=Password
Sets the given password as the default encryption and decryption password for Snapshot.
It is stored in encrypted form in the registry.
All new images are automatically encrypted with this password, and when mounting images on the same computer, no password prompt appears.
If you restore or mount the image on another computer, Snapshot will ask for the password.

This option combines the functionality of --SetDefaultEncPwd and --SetDefaultDecPwd and shares the same security limitations as the latter.

Disk and Partition Listing

--show
Displays an overview of all available disks and partitions.
Information for each detected disk includes its number, size, and partition layout.
You can limit the output to a specific disk by specifying it as a parameter (e.g., HD1, HD2, etc.).

--showlist
Similar to --show, but the output is formatted for easier automated processing in scripts or batch files.

Virtual Drive Management

--unmount
Unmounts all mounted virtual drives created by Snapshot.

--unmount:Z
Unmounts a specific mounted virtual drive, identified by its drive letter (e.g., Z:).
Example:

snapshot --unmount:Z


→ Removes only the virtual drive Z: from the system.
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

## FAQ

1. Can I back up dynamic disks with Snapshot?

Yes, you can also back up dynamic disks with Snapshot.
However, the partition information is not saved in the image.
This means that in the event of a restore, you cannot use Snapshot to restore the partition structure.
Such a backup therefore cannot be directly used for disaster recovery.

If you manually recreate the partitions exactly as they existed on the original disk, you can then restore the contents with Snapshot.
Images of dynamic disks can also be mounted as virtual drives.

For more detailed information about dynamic disks, see:
https://technet.microsoft.com/en-us/library/bb457110

2. Can I back up GPT partitions with Snapshot?

Yes, Snapshot can also back up disks that use the new GPT (GUID Partition Table) scheme.
When performing the backup, make sure that all partitions recognized by Snapshot on the disk are included.
When restoring, you must first restore the partition structure with Snapshot and then restore all the images.

3. Can I use Snapshot on UEFI-based systems?

Yes, Snapshot also works on computers that boot using UEFI.
However, when migrating to different hardware, ensure that Secure Boot is disabled in the new system’s firmware.
During the backup, make sure you back up all partitions detected by Snapshot on your drive.

4. Do I need to stop certain services before performing a backup?

No, normally all services can continue running during the backup.
However, make sure that all partitions containing the data or log files of a service are backed up at the same time.
In the graphical interface, select all affected partitions together.
From the command line, use the syntax for simultaneously backing up multiple partitions (see Chapter 5).

If you know that a particular service has problems recovering after a power failure, it’s a good idea to stop that service before running Snapshot.

5. When should I use VSS?

You should use VSS (Volume Shadow Copy Service) to create a shadow copy whenever you need to back up more than one partition simultaneously.

VSS creates a simultaneous shadow copy of all partitions to be backed up at the beginning of the process.
Snapshot then sequentially backs up the partitions from these shadow copies.

With Snapshot’s internal driver, it is not possible to back up multiple partitions at the same time.

6. Can I back up network drives with Snapshot?

No, Snapshot can only back up local drives.
Network drives cannot be backed up directly.

An exception applies to network storage that appears to the system as a local disk, such as iSCSI drives.

7. Can I save an image in a single file?

In principle, yes — you can do this by using the basic option -L0, which sets the maximum image file size to zero, meaning all data is stored in a single image file.

However, very large image files may cause system resource problems, as kernel memory usage increases with file size.

If you encounter an error like:

0x5aa - Not enough system resources to complete the requested service


you should reduce the maximum size of each image part by using a smaller value for -L.

8. Can I restore an image to a smaller hard drive?

It depends on how much smaller the new drive is.

Snapshot does not back up individual files, but rather the used sectors of the source drive.
During restoration, these sectors are written back to their original positions.

The target disk must therefore have enough space to accommodate all used sectors at their original offsets.
Additionally, the partition structure can only be partially restored if the new drive is smaller — otherwise, the last partition would extend beyond the physical size of the new disk.

If the new drive is significantly smaller, you can:

Create and format a new partition on it.

Mount the image as a virtual drive.

Copy all files manually from the mounted image to the new partition.

This way, you can transfer data from an image even to a much smaller drive.
The copying can be done, for example, using xcopy with the following command:

xcopy z: q: /e /c /h /r /k /o /x /b


This ensures that all files (including hidden and system files) are copied correctly.

9. Can I restore the image to different hardware?

Yes. For details on migrating to new hardware, please refer to Chapter 3.2.3 of the manual.

10. Can I use Snapshot on drives with data deduplication?

Yes. Drives configured with Microsoft Windows Server 2012 data deduplication can also be backed up using Snapshot.

If you are saving your backups to a deduplicated volume, start Snapshot with the option:

--DedupTarget


This ensures that Snapshot does not compress the data (since deduplication already handles that) and that the order of data blocks stored in the image remains as consistent as possible across multiple backups — improving deduplication efficiency.

11. Can Snapshot send me an email notification after the backup completes?

Snapshot itself cannot send emails directly,
but you can easily achieve this using a script combined with a command-line mail program.

Here’s an example:

snapshot C:+D: X:\image-$disk.sna --forcevss --LogFile:c:\snap.log
if %errorlevel% == 0 (
    echo "Backup completed successfully"
    REM The mail program can be called here
) else (
    echo "An error occurred"
    REM The mail program can be called here
)


After execution, Snapshot stores the exit status of the backup in the %errorlevel% variable.

A value of 0 means the backup finished without errors.

Any nonzero value indicates an error occurred — details can be found in the log file.

12. Do I need to back up partitions without drive letters?

Yes.
For a complete system backup, you must back up all partitions detected by Snapshot on the disk — including those without drive letters (e.g., EFI, recovery, or system-reserved partitions).

13. Can Snapshot write an entry to the Windows Event Log after the backup?

No, Snapshot itself cannot write to the Windows Event Log.

However, since Windows XP, Microsoft provides a command-line tool that can be used to create custom event log entries.
Together with the script from the previous section, you can use it to log success or failure of your backups.

The command-line tool is called eventcreate.
Here are two examples:

eventcreate /T ERROR /L APPLICATION /D "Snapshot Error" /id 100 /SO Snapshot
eventcreate /T SUCCESS /L APPLICATION /D "Snapshot OK" /id 100 /SO Snapshot


These commands create Application log entries indicating either success or failure of the Snapshot backup process.

## 6 Appendix A

If you do not need to add a driver for your hard disk controller, you can open a command prompt immediately after selecting the language and keyboard layout (see Image 1) by pressing the key combination <SHIFT> + <F10>.

At the command prompt, navigate to the directory where the file snapshot.exe is located and start the application.

The screenshots of the boot process with a Windows 7 DVD (see Image 3) show that by clicking the “Load Driver” button, you can insert any drivers that may be required for your hardware.

After opening the command prompt, change to the directory where Snapshot is stored, run the program, and perform the restore operation.

If you booted from a 64-bit DVD, you must also use the 64-bit version of Snapshot (snapshot64.exe), since the 32-bit subsystem is not loaded at that point.

It has proven practical to store snapshot.exe and your image files in the same folder, to avoid having to enter long paths.

If you need network support, you can initialize it by running the following command:

wpeutil initializenetwork


The network addressing will then be handled automatically via DHCP.

---

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