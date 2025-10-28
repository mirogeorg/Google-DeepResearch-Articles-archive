
# **zpaqfranz a Command: The Definitive Technical Reference**

## **Introduction**

zpaqfranz presents itself as a specialized instrument for high-stakes data resilience, described as a "Swiss army knife for the serious backup and disaster recovery manager".1 As a fork of the zpaq 7.15 reference implementation, it significantly extends the original's capabilities, introducing robust support for deduplication, file versioning ("snapshots"), advanced encryption, and a suite of integrity checks designed for what the project terms "paranoid-level tests".1 The a command, for "add" or "archive," is the primary interface for ingesting data into this system, serving as the cornerstone of any backup and recovery strategy built upon the tool.

### **The zpaqfranz Paradigm**

Unlike traditional archivers such as tar or 7z, which are often inefficient for repeated backups, zpaqfranz operates on a versioning model conceptually similar to Apple's Time Machine but with a focus on storage efficiency and data integrity.3 Its core value is derived from the ZPAQ algorithm's ability to maintain a complete history of changes through deduplicated "snapshots." This makes it exceptionally well-suited for scenarios involving daily or weekly backups of large datasets, virtual machine disks (.vmdk, .vhdx), and even text-based database dumps, where data redundancy between versions is high.1 By only storing the unique blocks of data that have changed, it minimizes storage growth over time while allowing for the restoration of any file from any point in the backup history.

The architectural decision to employ an append-only, versioned archive model is the direct mechanism that underpins the claim of being a "Ransomware neutralizer".1 When ransomware encrypts files on a source system, a subsequent backup does not overwrite the clean data; it simply adds the encrypted files as a new version. This process leaves all prior, unencrypted versions intact and fully recoverable from the archive, effectively neutralizing the threat of data loss from such an attack.

### **The Documentation Void**

Users seeking comprehensive, centralized documentation for zpaqfranz will find it sparse. This is not an oversight but a result of the developer's specific support philosophy. As detailed in public issue discussions, past attempts to create detailed guides were met with negative reactions, leading to a preference for addressing user needs and questions on an individual basis through the project's GitHub issue tracker.5 The developer is noted for being highly responsive in this forum.6 Consequently, much of the critical information regarding advanced features, new switches, and operational quirks is distributed across years of release notes, issue threads, and the embedded help system. This situation presents a challenge for administrators who require a complete and unambiguous understanding of the tool before entrusting it with critical data.5

### **Purpose of this Document**

This document is engineered to fill that void. It serves as the single, authoritative technical reference for the zpaqfranz a command, synthesized exclusively from primary source materials including the official GitHub repository, release announcements, issue discussions, and available man pages. It is designed for the professional user who requires a deep, nuanced understanding of the command's anatomy, its full lexicon of switches, its practical applications, and, most importantly, its edge cases and undocumented behaviors.

## **Section 1: Command Anatomy and Core Concepts**

Effective and safe utilization of the zpaqfranz a command requires a foundational understanding of its syntax and the underlying principles of its archive model. These concepts are fundamental to leveraging the tool's capabilities for versioning and data protection.

### **1.1. Fundamental Syntax**

The basic structure for invoking the add command is consistent and follows a standard command-line pattern 3:

zpaqfranz a archive\[.zpaq\]\[files|directory\]... \[-switches\]

* **zpaqfranz a**: The program name followed by the a command, which instructs the tool to add files to an archive.  
* **archive\[.zpaq\]**: The path to the archive file. If the file does not exist, it will be created. If it does exist, new data will be appended as a new version. The .zpaq file extension is assumed by default and can be omitted.  
* **\[files|directory\]...**: A space-separated list of one or more files or directories to be added to the archive. The tool will recursively process directories.  
* **\[-switches\]**: Optional flags that modify the command's behavior, such as enabling encryption, creating disk images, or altering file selection logic.

### **1.2. The Archive Model: Append-Only and Deduplication**

The most critical concept to grasp is that zpaqfranz archives are append-only by design. When an a command is executed against an existing archive, it does not modify or delete any of the data already stored. Instead, it creates a new "snapshot" or transaction at the end of the archive file, containing the new and changed files.1

The ZPAQ algorithm handles deduplication at the block level. When adding a file, zpaqfranz breaks it down into smaller data blocks. It then calculates a hash for each block and checks if a block with that same hash already exists anywhere in the archive's history. If it does, a simple reference is stored instead of the data itself. If the block is new, it is compressed and appended to the archive.

This process is the source of zpaqfranz's efficiency with large, slightly-modified files like virtual disks or database backups.1 A 100 GB virtual machine disk may only have a few hundred megabytes of changed data between daily snapshots. A traditional backup might store another 100 GB, whereas zpaqfranz will only store the few hundred megabytes of new, unique blocks, resulting in enormous storage savings over time. This append-only nature ensures that all historical versions are preserved and available "forever," though for performance reasons on mechanical hard drives, it is common practice to start a new archive after every 1,000 to 2,000 versions.1

### **1.3. Multipart Archives**

zpaqfranz supports the creation and management of multipart archives, which is useful for splitting large backups across multiple volumes or adhering to file size limits. This functionality is invoked by using wildcards in the archive name during the add operation.

The supported wildcards are \* and ? 3:

* ?: Matches a single digit. For example, backup??.zpaq would create backup01.zpaq, backup02.zpaq, etc.  
* \*: Matches the full part number. For example, backup\_\*.zpaq would create backup\_1.zpaq, backup\_2.zpaq, etc.

When using wildcards, the man pages provide a critical and capitalized warning: "DO NOT FORGET THE DOUBLE QUOTEs\!".3 This is to prevent the command-line shell from interpreting the wildcards itself before passing them to the zpaqfranz executable.

An example command to create a multipart archive would be:  
zpaqfranz a "archive\_part\_\*.zpaq" /path/to/data  
This command will create archive\_part\_1.zpaq, archive\_part\_2.zpaq, and so on, as needed to store the data. All subsequent operations on this archive (listing, extracting, testing) must also reference the archive name with the wildcard enclosed in double quotes.

## **Section 2: The Comprehensive Switch Lexicon**

The power and flexibility of the zpaqfranz a command are unlocked through its extensive set of optional switches. This information is often fragmented across release notes and embedded help screens. This section provides a consolidated, authoritative lexicon of these switches, categorized by function.

The following table serves as a quick-reference guide to the switches relevant to the a command. It is designed to be an at-a-glance tool for administrators to identify the appropriate switch and immediately understand its purpose and key operational considerations.

| Switch | Parameter(s) | Category | Description | Key Considerations & Quirks |
| :---- | :---- | :---- | :---- | :---- |
| \-always | None | File Selection | Forces files to be added even if their timestamp has not changed. | Overrides default change-detection logic. Conceptually similar to \-only. 7 |
| \-only | \[wildcard\] | File Selection | Specifies files to include. Can be repeated and used with wildcards. | Useful for selectively backing up specific file types or names. 7 |
| \-key | password | Encryption | Enables standard, zpaq-compatible AES-256 encryption. | The password is provided directly on the command line. 7 |
| \-keyfile | filename | Encryption | Uses the hash of a specified file as an additional encryption key. | File must have sufficient entropy. Use work keyfile to verify. 7 |
| \-franzen | password | Encryption | **Experimental.** Adds a second, zpaq-incompatible layer of encryption. | **NOT FOR PRODUCTION USE.** Breaks compatibility with standard zpaq. 7 |
| \-memzero | None | Security | Wipes FRANZEN-related data structures from memory after execution. | Primarily for high-security paranoia; developer notes it is "mostly unnecessary." 7 |
| \-image | None | Disk Imaging | Creates a raw, sector-by-sector (dd-like) image of a partition or disk. | Cannot be used to restore directly with zpaqfranz. Must be extracted first. 1 |
| \-vhd | None | Disk Imaging | **Experimental.** Used with \-image on Windows to create a mountable VHD. | **NOT FOR PRODUCTION USE.** Windows-only (NTFS). Lacks VSS support. 7 |
| \-sparse | None | I/O Optimization | On Windows, attempts to create a sparse file during extraction. | Not an a command switch, but critical for fast restoration of large files from archives. 7 |
| \-huge | None | I/O Optimization | Uses an alternative algorithm for preparing very large files for archiving. | Recommended for backups of few but enormous files on non-sparse filesystems. 7 |

### **2.1. File Selection and Processing**

These switches control which files are included in the archive operation, allowing for fine-grained control beyond simply specifying a directory.

* **\-always**: By default, zpaqfranz relies on file timestamps to quickly identify which files have changed since the last backup. The \-always switch overrides this behavior, forcing the program to re-evaluate and add all specified files, regardless of their modification time.7 This can be useful in situations where timestamps are unreliable (e.g., on certain network shares) or have been manipulated, ensuring that all content is freshly hashed and compared against the archive.  
* **\-only**: This switch provides an inclusive filter for adding files. When used, only files matching the provided pattern will be considered for the backup. It can be used multiple times on the same command line to include several patterns, and it supports standard wildcards.7 For example, zpaqfranz a backup.zpaq /docs \-only "\*.pdf" \-only "\*.docx" would back up only PDF and DOCX files from the /docs directory.

### **2.2. Encryption and Security**

zpaqfranz offers multiple layers of encryption, evolving from a standard implementation to advanced, experimental features tailored for high-security environments. This evolution reflects a clear trajectory from a general-purpose archiver to a specialized tool for secure data management, driven by the needs of professionals operating under compliance frameworks like GDPR.7

* **Standard Encryption (-key)**: This is the primary method for securing an archive. It uses a user-provided password to encrypt the archive's content with AES-256, maintaining full compatibility with the original zpaq standard. The password is provided as an argument on the command line: \-key "YourSecretPassword".  
* **Keyfile Encryption (-keyfile)**: For enhanced security, the \-keyfile switch allows the use of a separate file as part of the encryption key.7 Instead of just a password, zpaqfranz will hash the contents of the specified file (which must be between 4 KB and 50 MB) and use that hash in the key derivation process. This can be used in conjunction with \-key. The security of this method depends heavily on the entropy (randomness) of the keyfile. The developer provides a built-in tool to assess this: zpaqfranz work keyfile \<filename\>. This command will analyze the file and report whether its entropy is sufficient.7 A low-entropy file, such as a simple text document, is not a suitable keyfile.  
* **Experimental FRANZEN Encryption (-franzen)**: This feature represents a significant, forward-looking but highly experimental addition. It introduces a second, entirely separate layer of encryption that is **not compatible** with the standard zpaq format.7 The developer has issued explicit and strong warnings about its use, stating it is an "early (very immature) version" and "ABSOLUTELY not suitable for anything beyond experimentation".7 Its intended use case is for scenarios requiring dual control over data, such as GDPR-compliant cloud storage. In this model, a cloud provider could be given the "outer" FRANZEN password to manage the archive, while the data owner retains the sole "inner" password from the standard \-key switch, ensuring the provider can never access the actual data.7  
* **Post-Operation Security (-memzero)**: Related to the experimental FRANZEN format, the \-memzero switch instructs zpaqfranz to actively wipe any related keys or data structures from the computer's RAM after the operation completes.7 While included for completeness in high-paranoia scenarios, the developer notes that it is "mostly unnecessary in practice," as the risk of a precisely-timed RAM dump to steal a temporary key is exceptionally low.7

### **2.3. Advanced Archiving Formats (Disk Imaging)**

zpaqfranz extends beyond file-level backups to include powerful, block-level disk imaging capabilities. These features are particularly useful for bare-metal recovery scenarios. The development of these features reveals a clear division in the tool's functionality: a stable core inherited from zpaq and a highly experimental layer where new, powerful capabilities are being tested. Administrators must clearly distinguish between these two tiers to avoid catastrophic data loss.

* **Raw Sector-Level Imaging (-image)**: This switch changes the operating mode from file-based backup to a raw, sector-by-sector imaging tool, similar to the dd command in Unix-like systems.1 When used, zpaqfranz reads the raw blocks of the specified device or partition (e.g., C: on Windows) and stores them in the archive. This captures everything, including the partition table, boot sectors, and even deleted file data residing in unallocated space. It is important to note that zpaqfranz itself cannot directly restore this image; the raw image must first be extracted from the archive to a temporary location, and then other tools must be used to write it back to a disk.1  
* **Virtual Hard Disk Creation (-vhd)**: This is an experimental extension of the \-image command, available only on Windows for NTFS partitions.7 When both \-image and \-vhd are used, zpaqfranz stores the partition data in a special format within the archive. Upon extraction (using zpaqfranz x archive.zpaq \-vhd), this data is reassembled into a mountable Virtual Hard Disk (.vhd) file.7 This is a significant feature, as it allows for file-level recovery by simply mounting the VHD in Windows, without needing to restore the entire partition. However, it is subject to severe caveats: it is explicitly experimental, will display a large on-screen warning, and currently lacks support for Volume Shadow Copy Service (VSS), meaning it cannot safely image a live, running operating system.7

### **2.4. Filesystem and I/O Optimization**

These switches are designed to improve performance, particularly when dealing with very large files or specific types of storage hardware.

* **\-huge**: This switch activates a different internal algorithm for the file preparation phase of an a command.7 It is specifically recommended for backups that consist of a small number of enormous files, such as multi-terabyte virtual machine disks or scientific datasets. It is most beneficial on filesystems that do not support sparse files, where its alternative approach can yield better performance.  
* **\-sparse**: While not a switch for the a command, \-sparse is critically relevant to the lifecycle of backups containing large files. It is used during the extraction (x) command on Windows. It instructs zpaqfranz to attempt to create a sparse file on the NTFS filesystem.7 For large disk images or virtual disks that contain significant amounts of empty space, this can reduce the physical disk space required for the extracted file and can "halve the extraction time".7  
* **\-ssd**: This switch is used with the sum (checksum/hash) command, which is a vital part of a robust backup strategy for verifying data integrity post-backup.7 The release notes advise using the \-ssd flag only when the files being checked reside on solid-state drives, as it likely enables an I/O pattern optimized for flash storage.

## **Section 3: Applied Scenarios and Best Practices**

Synthesizing the commands and switches into practical workflows is essential for deploying zpaqfranz in a professional environment. The following scenarios illustrate common use cases, incorporating best practices derived from the tool's documentation and design principles.

### **3.1. Use Case: Incremental File Server Backup**

This is the canonical use case for zpaqfranz, leveraging its deduplication and versioning for efficient daily backups of a file server.

* Command:  
  zpaqfranz a //fileserver/backups/data.zpaq //fileserver/prod/data \-key "p@$$w0rd\_for\_backup"  
* **Workflow**:  
  1. The initial run of this command will create data.zpaq and perform a full backup of the //fileserver/prod/data share, encrypting it with the provided password.  
  2. Each subsequent daily run of the exact same command will append a new "snapshot." zpaqfranz will quickly scan the source files, and thanks to its block-level deduplication, only the changed portions of files will be compressed and added to the archive. This results in very fast and storage-efficient incremental backups.  
  3. **Archive Management**: An archive that is continuously appended to will grow indefinitely. While zpaqfranz is designed for this, performance, particularly during listing or extraction, can degrade on mechanical hard drives after thousands of versions. The developer notes that users "typically start from scratch every 1,000 or 2,000 or so versions for speed reasons".1 A robust strategy would involve rotating archives on a quarterly or yearly basis (e.g., creating data\_2024\_Q1.zpaq, data\_2024\_Q2.zpaq, etc.) to keep individual archive performance high.

### **3.2. Use Case: Bare-Metal System Imaging (Windows)**

This scenario details the process for creating a restorable image of a Windows system partition using the experimental \-vhd feature. This should be performed from a recovery environment, as VSS is not yet supported for live imaging.

* Step 1: Create the Backup Image:  
  Boot the target machine from a Windows Preinstallation Environment (WinPE) or similar recovery media. Assuming the system partition is C: and the backup destination is D:, run the following command:  
  zpaqfranz a D:\\system\_backups\\C\_drive.zpaq C: \-image \-vhd \-key "bare\_metal\_recovery\_key"  
  This command creates a raw, encrypted, VHD-compatible image of the C: drive. Note the large on-screen warning acknowledging the experimental nature of this feature.7  
* Step 2: Verify the Backup:  
  Data integrity is paramount. After the backup, perform a verification test on the archive.  
  zpaqfranz t D:\\system\_backups\\C\_drive.zpaq \-key "bare\_metal\_recovery\_key"  
  A successful test provides confidence that the archive is not corrupt.  
* Step 3: Restore and Mount the VHD:  
  To recover files or inspect the backup, extract the VHD file.  
  zpaqfranz x D:\\system\_backups\\C\_drive.zpaq \-vhd \-key "bare\_metal\_recovery\_key"  
  This will generate a image\_C.vhd file in the current directory.7 This file can then be mounted directly in Windows Disk Management for file-level access. For a full system restore, third-party imaging tools would be required to write the contents of the VHD back to a physical partition.

### **3.3. Use Case: Secure Cloud Deployment with Double Encryption**

This scenario illustrates the theoretical, GDPR-compliant workflow for which the experimental \-franzen feature was designed. **This is for illustrative purposes only and is not recommended for production use** until the feature is stabilized.

* Command:  
  zpaqfranz a /path/to/cloud/backup.zpaq /home/user/critical\_data \-key "User\_Only\_Secret\_Key" \-franzen "Password\_For\_Cloud\_Provider"  
* **Workflow**:  
  1. This command first encrypts the data using the standard \-key method. Only the end-user knows this password.  
  2. It then applies a second layer of FRANZEN encryption over the already-encrypted data stream.7 The password for this layer can be shared with a cloud service provider.  
  3. This allows the provider to perform integrity checks (work crc32) or manage the encrypted object without ever having the ability to decrypt the underlying user data.7 This separation of duties is a key principle in many compliance frameworks.  
  4. Again, the developer's warnings must be heeded: this feature is immature and should not be trusted with important data at this time.7

### **3.4. Use Case: Efficiently Backing Up Virtual Machine Disks**

zpaqfranz is described as "Ideal for virtual machine disk storage" due to its deduplication capabilities.1 VM disks are typically large, monolithic files that undergo small, localized changes between snapshots.

* Command:  
  zpaqfranz a /backup/vm\_archive.zpaq /vms/server.vmdk \-huge  
* **Workflow**:  
  1. The command adds the server.vmdk file to the archive. The \-huge switch is recommended to optimize the handling of this multi-gigabyte or terabyte file, especially if the /backup destination is a filesystem without sparse file support.7  
  2. When the VM is snapshotted and the command is run again, zpaqfranz will only need to read the changed blocks of the .vmdk file and store those. The result is an archive that contains the full history of the VM's disk states with minimal storage overhead.  
  3. For restoration, extracting a specific version of the .vmdk will yield a complete, bootable disk image from that point in time. When extracting to an NTFS volume, using the \-sparse switch is highly recommended to accelerate the process.7

## **Section 4: Quirks, Caveats, and Undocumented Behaviors**

To use zpaqfranz safely and effectively, an administrator must understand not just its commands, but also its development philosophy, environmental dependencies, and known limitations. The tool's behavior is influenced by more than just its command-line arguments; it is a dynamic utility whose functionality can be affected by its filename, its compilation environment, and the developer's direct engagement model. This section details the "tribal knowledge" required for professional use.

### **4.1. The Documentation and Support Philosophy**

The scarcity of formal documentation is a deliberate choice. The developer has stated a preference for an issue-driven support model over creating static guides, citing low user engagement and negative feedback on past efforts.5

* **Implication for the Administrator**: The primary and most effective method for resolving undocumented issues, clarifying behavior, or reporting bugs is to file a well-described issue on the official GitHub repository. The developer and community are noted as being highly responsive and helpful.6 The embedded help, accessed via zpaqfranz h (short help) and zpaqfranz h h (full help), should be considered the most up-to-date command reference.3

### **4.2. Navigating Experimental Features**

A recurring theme is the clear distinction between stable and experimental features. Failure to respect this distinction can lead to data loss.

* **Consolidated Warning**: The \-vhd and \-franzen features are explicitly and repeatedly labeled as experimental, immature, and not suitable for production use.7 They are powerful and demonstrate the future direction of the tool, but they should not be used for any data that cannot be lost.  
* **Recommended Strategy**: Administrators interested in these capabilities should adopt a strategy of testing them on non-critical data. Furthermore, they should actively monitor the GitHub repository's releases and issue tracker for official announcements regarding the stabilization of these features before considering them for production workloads.

### **4.3. Environmental and Compilation Dependencies**

A highly unusual characteristic of zpaqfranz is that its behavior can be altered by factors outside of its command-line arguments.

* **Behavior by Filename**: The executable's behavior can change based on the name it is given.3  
  * If named dir, it acts similarly to the Windows dir command.  
  * If named robocopy, it runs in a mode that mimics some functionality of Microsoft's RoboCopy utility.  
  * **Critical Hazard**: When run as robocopy, the \-kill switch is automatically enabled. This signifies a "wet run" that will perform destructive actions like deleting files. This is a significant and potentially data-losing side effect that administrators must be aware of before renaming the executable.  
* **Compile-Time Defines**: Not all zpaqfranz binaries are identical. The features and performance characteristics depend on flags used during compilation.3  
  * \-DNOJIT: Disables the Just-In-Time compiler, necessary for non-Intel CPUs like Apple M1/M2 or other ARM processors.  
  * \-DOPEN: Creates a special zpaqfranz-open build that is stripped of all binary-only or hard-to-interpret code, designed for high-security environments where a fully auditable open-source toolchain is required.7  
  * \-DHWSHA2: Enables hardware-accelerated SHA2 hashing, which can improve performance on modern CPUs that support it, such as AMD Ryzen processors.3  
  * An administrator must be aware of which binary they are using, as a generic build may lack performance optimizations or compatibility features present in a platform-specific compilation.

### **4.4. Known Limitations and Workarounds**

The tool has several known limitations that require specific workarounds.

* **Metadata Storage**: zpaqfranz currently does not store Unix-style owner/group permissions or other extended filesystem metadata.3 This is a planned future feature.  
  * **Official Workaround**: For users who must preserve this metadata, the recommended procedure is to first create a tar archive of the files. The .tar file, which preserves the metadata internally, can then be added to a .zpaq archive. This encapsulates the metadata within the versioned, deduplicated backup.  
* **Memory Constraints**: The man pages warn that the program can be unstable and may crash on systems with very little available memory, such as inexpensive Network Attached Storage (NAS) devices or certain ESXi server configurations.3 Administrators planning to run zpaqfranz on such hardware should conduct thorough stability testing before relying on it for production backups.

## **Conclusion**

The zpaqfranz a command is a powerful and highly specialized interface for creating versioned, deduplicated, and encrypted data archives. Its design prioritizes storage efficiency and data resilience, making it a compelling choice for professional backup and disaster recovery management. However, its power is matched by its complexity and a unique development ecosystem that demands a high level of user sophistication.

### **Summary of Capabilities**

The analysis of the a command and its associated switches demonstrates its suitability for a wide range of tasks, from incremental backups of file servers and efficient versioning of large virtual disks to experimental, bare-metal system imaging. Its flexible encryption options, including standard password protection and advanced keyfile methods, provide robust security for data at rest.

### **The Duality of zpaqfranz**

A central theme that emerges is the tool's dual nature. It is composed of a rock-solid, production-ready core of features inherited and enhanced from zpaq, which can be trusted for critical data. Layered on top of this is a bleeding-edge, experimental tier of functionality, such as VHD creation and FRANZEN encryption. These features showcase the developer's innovative vision but carry explicit warnings against production use. A successful administrator must recognize and respect this division, leveraging the stable core for reliability while cautiously exploring the experimental layer for future possibilities.

### **Final Recommendations for the Paranoid Administrator**

To operate zpaqfranz with the level of safety and confidence required for managing critical data, the following practices are recommended:

1. **Trust but Verify**: The zpaqfranz ecosystem provides the tools for its own validation. Always follow backup operations with verification commands like t (test) or sum to ensure archive integrity and the consistency of restored data. Never assume a backup is good without testing it.  
2. **Segregate Workloads**: Adhere strictly to the developer's warnings. Use the stable, well-documented features (a, x, t, \-key) for all critical production data. Reserve experimental switches (-vhd, \-franzen) for testing, evaluation, and non-essential workloads until they are officially declared stable in future release notes.  
3. **Embrace the Ecosystem**: Given the sparse formal documentation, engagement with the project's ecosystem is mandatory. Use the embedded help (zpaqfranz h h) as the primary command reference. Actively monitor the GitHub releases page for announcements of new features, bug fixes, and changes in feature stability. For support, utilize the GitHub issue tracker, which serves as the de facto channel for developer communication.  
4. **Know Your Binary**: Understand that the specific zpaqfranz executable you are using may have different capabilities based on how it was compiled. Verify if your binary includes platform-specific optimizations like hardware SHA acceleration. Avoid renaming the executable to dir or robocopy unless you fully understand the significant and potentially destructive changes in behavior this will cause.

#### **Works cited**

1. fcorbelli/zpaqfranz: Deduplicating archiver with encryption ... \- GitHub, accessed October 27, 2025, [https://github.com/fcorbelli/zpaqfranz](https://github.com/fcorbelli/zpaqfranz)  
2. Debian \-- Details of package zpaqfranz in trixie, accessed October 27, 2025, [https://packages.debian.org/stable/utils/zpaqfranz](https://packages.debian.org/stable/utils/zpaqfranz)  
3. zpaqfranz \- Swiss army knife for backup and disaster recovery \- Ubuntu Manpage, accessed October 27, 2025, [https://manpages.ubuntu.com/manpages/plucky/man1/zpaqfranz.1.html](https://manpages.ubuntu.com/manpages/plucky/man1/zpaqfranz.1.html)  
4. Swiss army knife for backup and disaster recovery | Man Page | Commands | zpaqfranz | ManKier, accessed October 27, 2025, [https://www.mankier.com/1/zpaqfranz](https://www.mankier.com/1/zpaqfranz)  
5. Documentation clarity · Issue \#128 · fcorbelli/zpaqfranz \- GitHub, accessed October 27, 2025, [https://github.com/fcorbelli/zpaqfranz/issues/128](https://github.com/fcorbelli/zpaqfranz/issues/128)  
6. zpaqfranz Reviews \- 2025 \- SourceForge, accessed October 27, 2025, [https://sourceforge.net/projects/zpaqfranz/reviews/](https://sourceforge.net/projects/zpaqfranz/reviews/)  
7. Releases · fcorbelli/zpaqfranz \- GitHub, accessed October 27, 2025, [https://github.com/fcorbelli/zpaqfranz/releases](https://github.com/fcorbelli/zpaqfranz/releases)