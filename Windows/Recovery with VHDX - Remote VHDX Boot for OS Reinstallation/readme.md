
# **Advanced Remote System Recovery: A Procedural Guide to Reinstalling a Primary OS Using a Native Boot VHDX Environment**

## **Section 1: Foundational Concepts and Strategic Prerequisites**

This document provides a comprehensive, expert-level procedural guide for the remote recovery of a damaged Windows operating system. The methodology involves deploying a Virtual Hard Disk (VHDX) file containing a bootable Windows instance to the target machine, remotely configuring the system to boot from this VHDX, and then using this temporary environment to format the primary partition and apply a fresh Windows image. This procedure is designed for scenarios where physical access to the machine is impossible, and traditional recovery methods, such as booting from USB media, are not viable. Success hinges on a precise understanding of the underlying technologies and meticulous execution of command-line operations.

### **1.1 The VHDX Native Boot Recovery Model: A Hypervisor-Free Approach**

The core technology enabling this recovery model is VHDX Native Boot. This feature, supported by modern Windows versions, allows an operating system to run directly from a .vhdx file located on a physical disk, without the intermediation of a hypervisor or a parent operating system.1 Unlike a traditional virtual machine (VM) that relies on a software layer for hardware emulation, a natively booted VHDX interacts directly with the system's hardware. This results in performance that is nearly identical to a standard installation on the same physical drive, particularly when hosted on modern solid-state drives (SSDs).1

This capability transforms the VHDX from a simple container for virtual machine disks into a powerful, portable, and self-contained operating environment. For the purposes of this procedure, the VHDX acts as a "recovery capsule"—a complete, functional Windows environment that can be deployed to a remote, compromised system. From within this temporary OS, an administrator gains the necessary platform to perform extensive repairs on the primary, damaged installation, which remains dormant and offline on its own partition. This approach effectively decouples the recovery toolset from the system being repaired, a significant strategic advantage over relying on potentially corrupted recovery partitions or tools on the target machine.

### **1.2 Pre-Deployment Assessment: A Remote Operations Checklist**

The remote and high-stakes nature of this operation necessitates a rigorous pre-deployment assessment. Failure to verify these prerequisites can lead to an unrecoverable state requiring physical intervention.

* **Administrative Access:** Unfettered administrative credentials for the target machine are non-negotiable. All subsequent remote commands and file operations depend on this level of access.1  
* **Remote Management Protocol Verification:** The Windows Remote Management (WinRM) service must be enabled, configured, and accessible through the system's firewall. WinRM is the underlying protocol for PowerShell Remoting, which is the primary mechanism for executing the required DiskPart, DISM, and bcdedit commands.5 On a system where it is not configured, the winrm quickconfig command can be executed remotely to establish default listeners, provided initial remote access (e.g., via another management tool) is possible.5  
* **System Architecture and Firmware:** It is critical to determine whether the target system utilizes legacy BIOS or modern UEFI firmware. This dictates the required disk partition scheme (MBR for BIOS, GPT for UEFI) and the specific syntax of boot configuration commands. For instance, bcdboot requires different flags (/f BIOS or /f UEFI), and UEFI systems have a dedicated EFI System Partition that must be correctly targeted.2  
* **Disk Space Analysis:** The physical disk on the target machine must have at least two partitions: a system partition containing the Boot Configuration Data (BCD) store and a main partition where the VHDX file will be stored.3 The partition hosting the VHDX file requires sufficient free space not only for the VHDX file itself (e.g., 64 GB) but also for a page file. When booting from a VHDX, Windows creates the page file *outside* the VHDX on the physical volume, a critical detail that must be factored into space calculations.3  
* **Source Materials:** The administrator must have a Windows Installation ISO file available. This is required to extract the install.wim (Windows Imaging Format) file, which contains the compressed operating system files that will be applied to both the VHDX and, later, the primary partition.8

### **1.3 Critical Risks, Limitations, and Mitigation Strategies**

This advanced procedure carries significant risks that are amplified in a remote context. A command syntax error that might be easily rectified with a bootable USB drive in hand can result in a completely unresponsive remote system.

* **VHDX File Integrity:** Dynamically expanding VHDX files are more susceptible to corruption from a sudden system shutdown, such as a power outage.3 For a recovery operation where reliability is paramount, a **fixed-size** VHDX is strongly recommended. A fixed-size VHDX allocates its entire file size upon creation, leading to better performance and greater resilience.3  
* **"Mark of the Web" (MotW) Complications:** A VHDX file downloaded from the internet may be tagged by Windows with a Zone.Identifier alternate data stream, also known as the "Mark of the Web".3 As of the November 2022 Windows updates, booting from a VHDX with this attribute can cause significant and unpredictable application failures within the booted OS.3 To mitigate this, the MotW must be removed from the VHDX file on the technician machine before it is uploaded to the target system. This can be accomplished with the PowerShell command: Unblock-File \-Path 'C:\\path\\to\\your.vhdx'.  
* **Disk Signature Collisions:** A disk signature conflict occurs when a VHDX created as a direct clone of a physical disk is mounted on the same machine from which it was cloned. Windows detects the duplicate signature and forcibly changes the signature of the attached VHDX, rendering it unbootable.11 This risk is inherently mitigated by the procedure outlined here, as it involves creating a new, blank VHDX and applying a generalized Windows image from an install.wim file, rather than cloning an existing disk.  
* **Operational Limitations:** Native Boot VHDX has several inherent limitations. System hibernation is not supported, although sleep mode is functional. VHDX files cannot be nested inside one another. Furthermore, Windows BitLocker Drive Encryption cannot be used to encrypt the host volume that contains the VHDX file, nor can it be used on volumes inside the VHDX.3  
* **Remote Risk Amplification:** The most significant risk is misconfiguration of the Boot Configuration Data (BCD) store. An incorrect bcdedit command can easily render a system unbootable.12 In a local scenario, this could be fixed by booting from installation media and accessing the recovery environment's startup repair tools.13 In a remote-only scenario, such a mistake is catastrophic, as it severs the administrator's connection and leaves the machine in a state that requires physical intervention. This elevates the importance of backing up the BCD store (bcdedit /export) from a best practice to a mission-critical, non-negotiable step before any modifications are made.7

The following table summarizes the key risks and provides actionable mitigation strategies to be employed before and during the operation.

| Risk | Consequence | Mitigation Strategy |
| :---- | :---- | :---- |
| VHDX File Corruption | The VHDX recovery environment fails to boot, leaving the system in its original damaged state. | Use a fixed-size VHDX instead of a dynamically expanding one for superior reliability.3 |
| "Mark of the Web" (MotW) | Applications within the VHDX-booted OS fail to launch or function correctly.3 | After downloading the VHDX or its source ISO, run the Unblock-File PowerShell cmdlet on the file before deployment. |
| BCD Misconfiguration | The remote machine becomes unbootable and unresponsive, requiring physical intervention to repair. | Before executing any modifications, create a backup of the BCD store using bcdedit /export C:\\BCD\_Backup.7 Double-check all {GUID} and path syntax before execution. |
| Insufficient Disk Space | The VHDX boot fails because Windows cannot create the necessary page file on the host volume.3 | Before transferring the VHDX, remotely verify that the target partition has free space equal to the VHDX size plus an estimated page file size (typically 1.5x system RAM). |

## **Section 2: Phase I \- Crafting and Staging the Recovery Environment**

This preparatory phase is conducted on a healthy "technician" computer. The objective is to construct and validate the VHDX recovery capsule before it is deployed to the remote target. This ensures that a known-good recovery environment is used, minimizing variables during the critical remote phase.

### **2.1 Sourcing and Preparing the Windows Installation Image (install.wim)**

The foundation of the new operating system is the install.wim file, which contains a compressed version of the Windows installation.

* **Extraction:** Mount the downloaded Windows Installation ISO file. This can typically be done in modern Windows versions by right-clicking the .iso file and selecting "Mount." Navigate to the mounted drive and locate the \\sources\\ directory. Inside this directory, find the install.wim or install.esd file.1  
* **Image Index Identification:** A single .wim file can contain multiple editions of Windows (e.g., Home, Pro, Enterprise). It is crucial to identify the correct edition to apply. Open an elevated Command Prompt or PowerShell and use the Deployment Image Servicing and Management (DISM) tool to query the file's contents.1  
  PowerShell  
  dism /Get-WimInfo /WimFile:"D:\\sources\\install.wim"

  (Replace D:\\ with the drive letter of your mounted ISO). The output will list the available Windows editions and their corresponding index numbers. Note the index number for the desired edition, as this will be a required parameter in a later step.1

### **2.2 VHDX Container Creation and Partitioning**

With the source image identified, the next step is to create the virtual disk container. This can be accomplished using either the legacy DiskPart utility or modern PowerShell cmdlets. PowerShell is generally recommended for its superior scripting and automation capabilities.

* Method 1: Using DiskPart (Command Prompt)  
  Open an elevated Command Prompt and execute the following commands sequentially. This example creates a 64 GB fixed-size VHDX file.  
  DOS  
  diskpart  
  create vdisk file="C:\\Recovery\\WinRE.vhdx" maximum=65536 type\=fixed  
  select vdisk file="C:\\Recovery\\WinRE.vhdx"  
  attach vdisk  
  convert gpt  
  create partition primary  
  format fs\=ntfs quick label\="VHDX\_OS"  
  assign letter=V  
  exit

  * **Explanation of Key Commands:**  
    * create vdisk... type=fixed: Creates the VHDX file, specifying a fixed allocation for reliability.2 The maximum size is specified in megabytes (65536 MB \= 64 GB).  
    * attach vdisk: Mounts the VHDX file so it appears to the system as a physical disk.20  
    * convert gpt: Initializes the virtual disk with the GUID Partition Table scheme, required for modern UEFI systems.1  
    * assign letter=V: Assigns a temporary, unique drive letter for accessing the VHDX's volume.2  
* Method 2: Using PowerShell (Recommended)  
  Open an elevated PowerShell window. These cmdlets achieve the same result as the DiskPart script above.  
  PowerShell  
  \# Create a 64 GB fixed-size VHDX  
  New-VHD \-Path "C:\\Recovery\\WinRE.vhdx" \-SizeBytes 64GB \-Fixed

  \# Mount the VHDX  
  Mount-VHD \-Path "C:\\Recovery\\WinRE.vhdx"

  \# Initialize the disk as GPT and create/format the volume  
  \# Note: Replace \<DiskNum\> with the actual disk number of the newly mounted VHD  
  Get-VHD \-Path "C:\\Recovery\\WinRE.vhdx" | Get-Disk | Initialize-Disk \-PartitionStyle GPT \-Passthru | New-Partition \-AssignDriveLetter \-UseMaximumSize | Format-Volume \-FileSystem NTFS \-NewFileSystemLabel "VHDX\_OS"

  This PowerShell pipeline is more concise and less prone to interactive errors than a multi-step DiskPart script.9

### **2.3 Applying the Windows Image to the VHDX using DISM**

Once the VHDX is created, partitioned, formatted, and mounted with a drive letter (e.g., V:), the final step in this phase is to apply the Windows image to it.

* **DISM Command:** Use the following DISM command in an elevated Command Prompt or PowerShell window.  
  PowerShell  
  dism /Apply\-Image /ImageFile:"D:\\sources\\install.wim" /Index:2 /ApplyDir:V:\\

  * **Command Breakdown:**  
    * /ImageFile: Specifies the full path to the source install.wim file.17  
    * /Index: The specific index number for the desired Windows edition, as identified in step 2.1.2  
    * /ApplyDir: The target drive letter of the mounted VHDX partition where the Windows files will be extracted.22

After this command completes, the V: drive will contain a full Windows file system (\\Windows, \\Users, etc.). The VHDX is now a self-contained, non-boot-configured "master image".8 At this stage, the .vhdx file can be detached and archived. This process decouples image creation from deployment, enabling an organization to maintain a standardized, pre-configured "golden" recovery VHDX. This master image can be updated periodically with security patches and essential diagnostic tools (e.g., disk health utilities, network analyzers, remote access software), transforming this procedure from a reactive, one-off fix into a proactive, scalable disaster recovery strategy. By storing this golden image on a central repository, the time-to-recovery for future incidents can be dramatically reduced.

## **Section 3: Phase II \- Remote Deployment and Boot Configuration**

This is the most critical and operationally sensitive phase. It involves transferring the prepared VHDX to the damaged remote system and manipulating its boot loader to force it to start from the new recovery environment. Every command must be executed with precision.

### **3.1 Transfer and Placement of the VHDX File**

The prepared .vhdx file, which may be 25-65 GB or larger, must be transferred to the remote machine. The file should be placed in a simple, known location on the primary OS partition, for example, C:\\Recovery\\WinRE.vhdx. Standard enterprise file transfer methods such as robocopy over a network share or PowerShell's Copy-Item can be used.

### **3.2 Establishing and Utilizing a Remote PowerShell Session**

All subsequent commands must be executed on the remote machine. This is achieved via a PowerShell Remoting session. The Invoke-Command cmdlet is generally safer for this procedure than an interactive Enter-PSSession session, as it allows for the execution of a pre-validated script block, minimizing the risk of typographical errors during a live, interactive session.6

* **Example Connection Command:**  
  PowerShell  
  $credential \= Get-Credential  
  Invoke-Command \-ComputerName "RemotePC" \-Credential $credential \-ScriptBlock {  
      \# All commands from sections 3.3 and 3.4 will be placed here  
  }

### **3.3 Remotely Modifying the Boot Configuration Data (BCD) Store**

The following sequence of commands, executed within the remote PowerShell session, will add the VHDX to the remote machine's boot menu.

* Step 1: Backup the Existing BCD (CRITICAL)  
  This is the primary safety measure. Before making any changes, export the current BCD configuration to a file.  
  PowerShell  
  bcdedit /export C:\\Recovery\\BCD\_Backup

  This backup file provides a potential path to restoration should the subsequent steps fail, although restoring it would still require some form of bootable recovery environment.7  
* Step 2: Create a New Boot Entry for the VHDX  
  This command duplicates the current default boot entry, which serves as a template.  
  PowerShell  
  bcdedit /copy {current} /d "VHDX Recovery Environment"

  Upon successful execution, this command will output a new, unique GUID (Globally Unique Identifier) enclosed in braces, such as {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}. This GUID must be captured and used in the subsequent commands.7  
* Step 3: Configure the New Entry to Point to the VHDX  
  Using the {guid} obtained in the previous step, reconfigure the new boot entry to point to the VHDX file.  
  PowerShell  
  \# Replace {guid} with the actual GUID from the /copy command  
  bcdedit /set {guid} device vhd=\[C:\]\\Recovery\\WinRE.vhdx  
  bcdedit /set {guid} osdevice vhd=\[C:\]\\Recovery\\WinRE.vhdx

  The vhd=\[C:\]... syntax is crucial. The square brackets instruct the Windows Boot Manager to locate the volume with the drive letter C at boot time and then find the specified path to the .vhdx file on that volume.14  
* Step 4: Enable HAL Detection (Recommended)  
  This setting helps the operating system correctly detect the Hardware Abstraction Layer, which can prevent boot issues related to hardware mismatches.  
  PowerShell  
  bcdedit /set {guid} detecthal on

  This command is a common requirement for ensuring stable native VHD boot.7  
* Step 5: Verify the Configuration  
  Before proceeding, verify that the BCD store has been correctly modified.  
  PowerShell  
  bcdedit /enum /v

  Examine the output to confirm the presence of a new "Windows Boot Loader" entry with the description "VHDX Recovery Environment." Verify that its device and osdevice parameters correctly point to the VHDX file path.8

### **3.4 Orchestrating the Unattended Remote Reboot**

The entire remote operation hinges on the precise choreography of the next two commands. Because there is no user present to select an option from a boot menu, the boot sequence must be predetermined before the reboot is initiated.

* **The Linchpin Command:** The bcdedit tool provides two methods to control the next boot.  
  * bcdedit /default {guid}: This command sets the VHDX entry as the new permanent default boot option. While effective, it requires the administrator to remember to change the default back after the recovery is complete.7  
  * bcdedit /bootsequence {guid}: **This is the preferred method.** It instructs the boot manager to use the specified entry for the *next boot only*. After that single boot, the default order reverts to its previous state. This is ideal for a one-time recovery operation, as it is self-correcting.15  
* Triggering the Reboot:  
  Immediately after setting the boot sequence, initiate the reboot.  
  PowerShell  
  \# Using the preferred method:  
  bcdedit /bootsequence {guid}

  \# Immediately trigger the reboot  
  Shutdown /r /t 0

  This sequence represents a non-interactive "handshake" between the running OS and the bootloader. The bcdedit command prepares the instructions, and the shutdown command executes the handoff. Any error or interruption between these two commands will cause the machine to reboot into its original damaged state, requiring the process to be restarted from the beginning.

## **Section 4: Phase III \- Executing the Primary OS Reinstallation from the VHDX Environment**

Following the successful remote reboot, the target machine is now running the temporary Windows operating system from the WinRE.vhdx file. The original, damaged OS partition is now dormant and accessible as a standard data volume, ready for manipulation.

### **4.1 Navigating the System from the VHDX-Booted OS**

The act of booting from the VHDX fundamentally alters the system's perspective. The recovery OS becomes the "live" system, demoting the once-primary OS partition to a simple data volume. This "role reversal" is the key enabler of the entire reinstallation, as it removes the file locks and system protections that would normally prevent the formatting of a running operating system's partition.25

* **System State:** The VHDX-booted environment will see its own system root as the C: drive. The physical disk's original C: drive, which contains the damaged OS, will now be mounted with a different drive letter, such as D: or E:.26 The WinRE.vhdx file itself will be located on this secondary volume (e.g., at D:\\Recovery\\WinRE.vhdx).  
* **Re-establishing Remote Control:** The administrator must now establish a new remote connection (e.g., RDP or PowerShell Remoting) to the machine, which is running the OS from the VHDX. This is contingent on the VHDX image having been prepared with networking and remote management services enabled.

### **4.2 Decommissioning the Damaged OS Partition**

With remote control re-established to the VHDX environment, the next step is to wipe the old, damaged OS partition clean. **This action is irreversible and will result in the complete destruction of all data on the target partition.**

* **Target Identification and Formatting:** Open an elevated Command Prompt or PowerShell within the VHDX environment and use DiskPart to precisely identify and format the target partition.  
  DOS  
  diskpart

  \# List all physical disks  
  list disk

  \# Select the disk containing the damaged OS (usually disk 0)  
  select disk 0

  \# List all partitions on the selected disk  
  list partition

  \# Identify the partition number corresponding to the old OS volume (e.g., D:)  
  \# Let's assume it is partition 3 for this example  
  select partition 3

  \# Format the partition quickly with the NTFS file system and assign a new label  
  format fs\=ntfs quick label\="New\_OS"

  exit

  Extreme care must be taken to select the correct partition number. Formatting the wrong partition (such as the EFI System Partition) will render the entire system unbootable.27

### **4.3 Applying the New Windows Image to the Primary Partition**

With the primary partition now clean, a fresh Windows image can be applied to it.

* **Accessing Installation Files:** The install.wim file must be accessible from the VHDX environment. It could have been pre-copied into the VHDX during the crafting phase, or it can be accessed from a network share.  
* **Applying the Image with DISM:** Use the DISM tool to apply the image to the newly formatted primary partition (which may still be mounted as D:).  
  PowerShell  
  \# Assuming the install.wim is on a network share accessible as Z:  
  dism /Apply\-Image /ImageFile:"Z:\\sources\\install.wim" /Index:2 /ApplyDir:D:\\

  This command is functionally identical to the one used to create the VHDX, but the /ApplyDir now targets the physical partition that will host the new permanent operating system.29

### **4.4 Rebuilding the Boot Entry for the New Primary OS**

After the image is applied, the final step is to configure the BCD store so that the machine will boot from this new installation. The bcdboot command is the designated tool for this task.

* **Using bcdboot:** This command copies the necessary boot files from the new Windows installation to the system partition and creates a new, default boot entry in the BCD store.  
  PowerShell  
  \# D:\\Windows is the location of the new OS; S: is the system partition  
  bcdboot D:\\Windows /s S:

* **Command Breakdown:**  
  * D:\\Windows: This is the source path to the \\Windows directory of the newly applied operating system.31  
  * /s S:: This crucial parameter specifies the target system partition where the boot files will be copied. On UEFI systems, the EFI System Partition is often hidden and does not have a drive letter by default. It may be necessary to use diskpart first to temporarily assign letter=S to this partition before running bcdboot.2

Executing this command effectively makes the new Windows installation on the physical partition the default boot option for the computer.

## **Section 5: Post-Reinstallation Verification and System Cleanup**

The final phase involves rebooting into the newly restored primary OS and removing the temporary recovery environment to return the system to a clean, production-ready state.

### **5.1 Final Reboot and Verification**

From within the VHDX-booted recovery environment, execute a simple reboot command.

PowerShell

shutdown /r /t 0

Because the bcdboot command in the previous phase set the new primary OS as the default, the machine should now boot directly into the freshly installed Windows environment. The administrator should then verify that remote connectivity is restored and that the system is stable and functional. The initial boot will proceed through the "Out-of-Box Experience" (OOBE) setup, unless this was automated with an unattend file.

### **5.2 Decommissioning the Recovery Environment**

Once the new primary OS is confirmed to be fully operational, the temporary recovery artifacts should be removed to reclaim disk space and eliminate boot menu clutter.

* Clean the BCD Store:  
  Booted into the new primary OS, open an elevated Command Prompt or PowerShell.  
  1. Run bcdedit /enum /v to list all boot entries and identify the unique {GUID} associated with the "VHDX Recovery Environment" entry.  
  2. Execute the delete command, replacing {guid} with the correct identifier:  
     PowerShell  
     bcdedit /delete {guid}

     This permanently removes the VHDX option from the boot menu.15  
* Delete the VHDX File:  
  Navigate to the location where the recovery files were stored (e.g., C:\\Recovery\\) and delete the WinRE.vhdx file and the BCD\_Backup file. The system is now restored to a clean, single-boot configuration.

## **Conclusion: Summary and Best Practices**

The procedure detailed in this report presents a powerful, albeit complex, method for the complete remote reinstallation of a Windows operating system. It leverages VHDX Native Boot technology to create a temporary, hypervisor-free recovery platform on the target machine itself. The operation can be summarized in four distinct phases:

1. **Craft:** A VHDX recovery environment is meticulously prepared and pre-loaded with a Windows image on a healthy technician machine.  
2. **Deploy:** The VHDX file is transferred to the remote machine, and its boot loader is remotely reconfigured using bcdedit to force a one-time boot into the recovery environment.  
3. **Execute:** From within the VHDX-booted OS, the original damaged partition is formatted, and a fresh Windows image is applied using DISM. The boot loader is then rebuilt with bcdboot to point to the new installation.  
4. **Clean:** After a final reboot into the restored OS, the temporary VHDX boot entry and its associated files are removed.

The point of highest risk and complexity is the remote boot orchestration, which relies on a precise and uninterrupted sequence of bcdedit and shutdown commands. An error in this phase can result in a loss of remote connectivity, necessitating physical intervention.

As a best practice, organizations should consider moving from a reactive to a proactive disaster recovery posture. Instead of crafting a new recovery VHDX for each incident, a standardized "golden" VHDX image should be created and maintained. This image can be pre-loaded with essential diagnostic utilities, security agents, and remote management tools, and stored in a central software repository. Adopting this strategy transforms a complex, manual procedure into a more streamlined and repeatable process, significantly reducing recovery time and risk for future incidents.

## **Appendix: Command Reference and Execution Phase**

The following table provides a consolidated reference for the primary command-line tools and their specific roles within each phase of the recovery operation.

| Phase | Tool | Core Command Example | Purpose in this Operation |
| :---- | :---- | :---- | :---- |
| Phase I: Crafting | DISM | dism /Get-WimInfo /WimFile:"D:\\sources\\install.wim" | Identify the correct Windows edition index from the source image file.1 |
| Phase I: Crafting | DiskPart | create vdisk file="..." maximum=... type=fixed | Create the fixed-size VHDX container file that will host the recovery OS.2 |
| Phase I: Crafting | PowerShell | New-VHD \-Path "..." \-SizeBytes... \-Fixed | Create the fixed-size VHDX container via a scriptable PowerShell cmdlet.21 |
| Phase I: Crafting | DISM | dism /Apply-Image /ImageFile:"..." /Index:... /ApplyDir:... | Apply the Windows OS files from the .wim file into the prepared VHDX.17 |
| Phase II: Deployment | bcdedit | bcdedit /export C:\\Recovery\\BCD\_Backup | **CRITICAL:** Back up the remote machine's existing BCD store before modification.7 |
| Phase II: Deployment | bcdedit | bcdedit /copy {current} /d "..." | Duplicate the current boot entry to create a template for the VHDX entry.14 |
| Phase II: Deployment | bcdedit | bcdedit /set {guid} device vhd=\[C:\]\\path\\to.vhdx | Configure the new boot entry to point to the VHDX file on the physical disk.33 |
| Phase II: Deployment | bcdedit | bcdedit /bootsequence {guid} | Set the VHDX entry to be used for the next boot cycle only.15 |
| Phase III: Execution | DiskPart | format fs=ntfs quick label="..." | **DESTRUCTIVE:** Wipe the partition containing the damaged primary OS.27 |
| Phase III: Execution | DISM | dism /Apply-Image /ImageFile:"..." /Index:... /ApplyDir:... | Apply a fresh Windows OS image to the newly formatted primary partition.29 |
| Phase III: Execution | bcdboot | bcdboot D:\\Windows /s S: | Create a new default boot entry for the freshly installed primary OS.32 |
| Phase V: Cleanup | bcdedit | bcdedit /delete {guid} | Remove the temporary VHDX recovery entry from the boot menu.15 |

#### **Works cited**

1. How to Natively Boot a Windows 11 Virtual Hard Disk (VHDX) \- NinjaOne, accessed October 26, 2025, [https://www.ninjaone.com/blog/native-boot-vhdx-virtual-hard-disk/](https://www.ninjaone.com/blog/native-boot-vhdx-virtual-hard-disk/)  
2. Boot to a virtual hard disk: Add a VHDX or VHD to the boot menu | Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/boot-to-vhd--native-boot--add-a-virtual-hard-disk-to-the-boot-menu?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/boot-to-vhd--native-boot--add-a-virtual-hard-disk-to-the-boot-menu?view=windows-11)  
3. Deploy Windows with a VHDX (Native Boot) \- Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-on-a-vhd--native-boot?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/deploy-windows-on-a-vhd--native-boot?view=windows-11)  
4. Manage Virtual Hard Disks (VHD) \- Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/disk-management/manage-virtual-hard-disks](https://learn.microsoft.com/en-us/windows-server/storage/disk-management/manage-virtual-hard-disks)  
5. Installation and configuration for Windows Remote Management \- Win32 apps, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management](https://learn.microsoft.com/en-us/windows/win32/winrm/installation-and-configuration-for-windows-remote-management)  
6. Running Remote Commands \- PowerShell | Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/running-remote-commands?view=powershell-7.5](https://learn.microsoft.com/en-us/powershell/scripting/security/remoting/running-remote-commands?view=powershell-7.5)  
7. How to Add a Native-Boot Virtual Hard Disk to the Boot Menu \#Hyperv, accessed October 26, 2025, [https://mountainss.wordpress.com/2012/12/16/how-to-add-a-native-boot-virtual-hard-disk-to-the-boot-menu-hyperv/](https://mountainss.wordpress.com/2012/12/16/how-to-add-a-native-boot-virtual-hard-disk-to-the-boot-menu-hyperv/)  
8. Booting Windows 10 natively from a .VHDX drive file \- Cesar de la Torre \- Blog, accessed October 26, 2025, [http://cesardelatorre.azurewebsites.net/post/booting-windows-10-natively-from-a-vhdx-drive-file](http://cesardelatorre.azurewebsites.net/post/booting-windows-10-natively-from-a-vhdx-drive-file)  
9. Quick Deploy Windows to VHDX (or a real drive) from Windows :: HowettNET, accessed October 26, 2025, [https://www.howett.net/spellbook/deploy\_windows\_from\_windows/](https://www.howett.net/spellbook/deploy_windows_from_windows/)  
10. Create vdisk \- Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg252579(v=ws.11)](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg252579\(v=ws.11\))  
11. Creating a Vhdx (disk2vhd) and running it through hyper-V Manager on the same PC, accessed October 26, 2025, [https://www.reddit.com/r/HyperV/comments/12xxgpo/creating\_a\_vhdx\_disk2vhd\_and\_running\_it\_through/](https://www.reddit.com/r/HyperV/comments/12xxgpo/creating_a_vhdx_disk2vhd_and_running_it_through/)  
12. Modify Windows BCD using Powershell \- CodeProject, accessed October 26, 2025, [https://www.codeproject.com/articles/Modify-Windows-BCD-using-Powershell](https://www.codeproject.com/articles/Modify-Windows-BCD-using-Powershell)  
13. I cannot boot on my vhdx, I converted from a physical disk. In the picture you see what happens for each partition selected (they are selected bottom left) : r/HyperV \- Reddit, accessed October 26, 2025, [https://www.reddit.com/r/HyperV/comments/y8871i/i\_cannot\_boot\_on\_my\_vhdx\_i\_converted\_from\_a/](https://www.reddit.com/r/HyperV/comments/y8871i/i_cannot_boot_on_my_vhdx_i_converted_from_a/)  
14. VHD-To-BCD: Add Virtual-Hard-Disks to Bootmanager \- Directory Opus Resource Centre, accessed October 26, 2025, [https://resource.dopus.com/t/vhd-to-bcd-add-virtual-hard-disks-to-bootmanager/21178](https://resource.dopus.com/t/vhd-to-bcd-add-virtual-hard-disks-to-bootmanager/21178)  
15. What BCDEdit Does and How To Use It \- NinjaOne, accessed October 26, 2025, [https://www.ninjaone.com/blog/what-bcdedit-does-and-how-to-use-it/](https://www.ninjaone.com/blog/what-bcdedit-does-and-how-to-use-it/)  
16. Melody's Ultra Tweaks Pack \- How to install Windows to a VHD \- Google Sites, accessed October 26, 2025, [https://sites.google.com/view/melodystweaks/wintovhd](https://sites.google.com/view/melodystweaks/wintovhd)  
17. DISM Image Management Command-Line Options | Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-image-management-command-line-options-s14?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/dism-image-management-command-line-options-s14?view=windows-11)  
18. Installing Windows on VHD to use with Hyper-V \- DEV Community, accessed October 26, 2025, [https://dev.to/milolav/installing-windows-on-vhd-to-use-with-hyper-v-1j8e](https://dev.to/milolav/installing-windows-on-vhd-to-use-with-hyper-v-1j8e)  
19. How to create a virtual hard disk (VHD) on Windows \- DiskInternals, accessed October 26, 2025, [https://www.diskinternals.com/vmfs-recovery/how-to-create-virtual-hard-disk-windows/](https://www.diskinternals.com/vmfs-recovery/how-to-create-virtual-hard-disk-windows/)  
20. Walkthrough 2 \- Part 2 (DiskPart \- create .vhd), accessed October 26, 2025, [http://mistyprojects.co.uk/documents/WindowsToGo/readme.files/w2\_diskpart\_vhd.htm](http://mistyprojects.co.uk/documents/WindowsToGo/readme.files/w2_diskpart_vhd.htm)  
21. Create VHD and VHDX Virtual Hard Disks GUI and Powershell \- YouTube, accessed October 26, 2025, [https://www.youtube.com/watch?v=xJ5S-ZY\_8Go](https://www.youtube.com/watch?v=xJ5S-ZY_8Go)  
22. Adding an OS Image to a VHD | Create a Windows Image Tutorial \- Part 5 \- YouTube, accessed October 26, 2025, [https://www.youtube.com/watch?v=dx6DLHSaOMM](https://www.youtube.com/watch?v=dx6DLHSaOMM)  
23. Performing an in-place upgrade of Windows Server | Compute Engine, accessed October 26, 2025, [https://docs.cloud.google.com/compute/docs/tutorials/performing-in-place-upgrade-windows-server](https://docs.cloud.google.com/compute/docs/tutorials/performing-in-place-upgrade-windows-server)  
24. Create & Boot VHD \- A World Full of Sharp Objects, accessed October 26, 2025, [https://www.kjetilk.com/2011/08/create-boot-vhd.html](https://www.kjetilk.com/2011/08/create-boot-vhd.html)  
25. Disk Management in Windows \- Microsoft Support, accessed October 26, 2025, [https://support.microsoft.com/en-us/windows/disk-management-in-windows-ad88ba19-f0d3-0809-7889-830f63e94405](https://support.microsoft.com/en-us/windows/disk-management-in-windows-ad88ba19-f0d3-0809-7889-830f63e94405)  
26. vhdx native boot : r/virtualization \- Reddit, accessed October 26, 2025, [https://www.reddit.com/r/virtualization/comments/1fxcpgt/vhdx\_native\_boot/](https://www.reddit.com/r/virtualization/comments/1fxcpgt/vhdx_native_boot/)  
27. Can't format a secondary HDD because windows sees it as a system drive \- Reddit, accessed October 26, 2025, [https://www.reddit.com/r/techsupport/comments/3j2m07/cant\_format\_a\_secondary\_hdd\_because\_windows\_sees/](https://www.reddit.com/r/techsupport/comments/3j2m07/cant_format_a_secondary_hdd_because_windows_sees/)  
28. How to Create a Primary Partition Using Diskpart \- Intel, accessed October 26, 2025, [https://www.intel.com/content/www/us/en/support/articles/000059417/memory-and-storage.html](https://www.intel.com/content/www/us/en/support/articles/000059417/memory-and-storage.html)  
29. Capture and Apply Windows using a WIM file | Microsoft Learn, accessed October 26, 2025, [https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/capture-and-apply-windows-using-a-single-wim?view=windows-11](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/capture-and-apply-windows-using-a-single-wim?view=windows-11)  
30. Is it possible to install Windows on a hard drive on one computer, then activate it on another? \- Super User, accessed October 26, 2025, [https://superuser.com/questions/1697445/is-it-possible-to-install-windows-on-a-hard-drive-on-one-computer-then-activate](https://superuser.com/questions/1697445/is-it-possible-to-install-windows-on-a-hard-drive-on-one-computer-then-activate)  
31. How to boot your PC from a VHD. Today we are going to talk about ..., accessed October 26, 2025, [https://medium.com/tech-jobs-academy/how-to-boot-your-pc-from-a-vhd-365394a751cf](https://medium.com/tech-jobs-academy/how-to-boot-your-pc-from-a-vhd-365394a751cf)  
32. bcd \- How to use BCDEdit to dual boot Windows installations? \- Super User, accessed October 26, 2025, [https://superuser.com/questions/511582/how-to-use-bcdedit-to-dual-boot-windows-installations](https://superuser.com/questions/511582/how-to-use-bcdedit-to-dual-boot-windows-installations)  
33. Adding Windows 8 VHD to boot menu \- Stack Overflow, accessed October 26, 2025, [https://stackoverflow.com/questions/13782751/adding-windows-8-vhd-to-boot-menu](https://stackoverflow.com/questions/13782751/adding-windows-8-vhd-to-boot-menu)