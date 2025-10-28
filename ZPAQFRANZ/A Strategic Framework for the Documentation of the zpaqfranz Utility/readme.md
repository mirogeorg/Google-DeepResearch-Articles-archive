
# **zpaqfranz: The Definitive Guide to a High-Integrity Journaling Archiver**

## **Part 1: Foundational Concepts**

This section establishes the conceptual groundwork for understanding zpaqfranz, detailing its origins, core technological principles, and the specific problems it is engineered to solve. It provides the necessary context for the comprehensive command and switch references that follow, moving beyond a simple functional description to explain the fundamental design philosophy that drives the utility's development.

### **1.1 Introduction: From zpaq to zpaqfranz**

zpaqfranz is a powerful, cross-platform, command-line journaling archiver that represents a significant evolution of the original zpaq utility developed by Matt Mahoney.1 The original zpaq (version 7.15 being the final release from Mahoney) established a robust foundation as an open-source archiver renowned for its innovative append-only journaling format, efficient content-aware deduplication, and the ability to roll back an archive to any previous state.3 It was designed for reliable, incremental backups, adding only files that had changed since the last update.

The zpaqfranz project, initiated and maintained by Franco Corbelli, is a direct fork of zpaq 7.15. It is not merely a continuation but a substantial expansion of the original concept, aptly described as zpaq "on steroids".2 It inherits all the core strengths of its predecessor while introducing a vast array of new commands, modern cryptographic features, performance optimizations, and system-level utilities. This transforms it from a pure archiver into a comprehensive "Swiss army knife for the serious backup and disaster recovery manager".1

A core design principle that distinguishes zpaqfranz is its deep-seated focus on verifiable data integrity, a philosophy that caters to users for whom data loss is not an option. This approach directly addresses the long-term risks inherent in data storage, such as silent data corruption ("bit rot") and the threat of malicious data alteration by ransomware. While the original zpaq provided a solid framework for data preservation, its reliance on SHA-1 for fragment hashing—a standard sufficient for deduplication but no longer considered cryptographically secure against sophisticated collision attacks—presented a potential future vulnerability for long-term archives.3

Recognizing this, zpaqfranz implements a multi-layered verification strategy as a central feature. By default, it performs a triple-check on data using chunked SHA-1, XXHASH64, and CRC-32, providing a high degree of confidence in data integrity out of the box.1 For users demanding even higher assurance, it incorporates a suite of modern, cryptographically secure hashing algorithms, including SHA-2 (SHA-256), SHA-3 (SHA-3-256), BLAKE3, and WHIRLPOOL.1 This deliberate and expansive focus on "paranoid-level tests" is a defining characteristic of the fork, positioning zpaqfranz as a tool engineered for maximum data resilience in mission-critical environments.1

### **1.2 Core Technologies Explained**

The power and efficiency of zpaqfranz are built upon several key technologies inherited from zpaq and enhanced in the fork. Understanding these mechanisms is essential for leveraging the tool's full potential.

#### **Journaling Archives**

zpaqfranz utilizes a journaling, or append-only, archive format.3 When an archive is updated, new data and metadata are always appended to the end of the file; existing data is never modified or overwritten. This design has several profound advantages:

* **Inherent Safety:** Since the original data in the archive is never altered, the risk of corrupting an archive during an interrupted update is minimized. If an update process is terminated, the incomplete transaction at the end of the file is simply ignored on the next run or can be explicitly removed with the trim command.4  
* **Efficiency for Synchronization:** The append-only nature is ideal for network synchronization and remote backups. Tools like rsync can be used with the \--append flag to transfer only the newly added data, dramatically reducing bandwidth consumption compared to re-uploading an entire modified archive file.1  
* **Suitability for WORM Media:** This format is naturally suited for Write-Once, Read-Many (WORM) storage media, as it does not require modification of existing blocks.

#### **Content-Defined Chunking & Deduplication**

At the heart of zpaqfranz's storage efficiency is its advanced deduplication mechanism, which operates at the sub-file level.4 Unlike traditional archivers that store whole files or use fixed-size blocks, zpaqfranz employs content-defined chunking to split files into variable-sized fragments.

The process works as follows:

1. **Rolling Hash:** As the archiver processes a file, it calculates a rolling hash over a small window of bytes (typically the last 32). This hash value changes as the window slides along the file's content.3  
2. **Boundary Detection:** A fragment boundary is marked whenever the rolling hash value meets a specific mathematical condition (e.g., its leading 16 bits are all zero). Because this condition depends on the file's content rather than a fixed position, the boundaries are consistent. If a section of a file is inserted or deleted, only the fragments around the change will be affected; the rest of the file's fragments will remain identical. This results in an average fragment size of 64 KiB.3  
3. **Fragment Hashing:** For each identified fragment, a cryptographically strong SHA-1 hash is computed.3  
4. **Deduplication:** This SHA-1 hash is compared against a database of all fragment hashes already stored in the archive. If the hash already exists, the fragment is considered a duplicate, and only a pointer to the previously stored fragment is saved. If the hash is new, the fragment is compressed and appended to the archive, and its hash is added to the database.4

This method is exceptionally effective for backups of large, frequently modified files like virtual machine disk images (VMDK, VHDx), database dumps, and TrueCrypt containers.1 Even a small change within a multi-gigabyte file will only result in a few new fragments being stored, rather than the entire file.

#### **Versioning and "Snapshots"**

The combination of the journaling format and content-defined deduplication enables zpaqfranz to maintain a complete history of all backed-up data, creating what is conceptually described as a "snapshot" with each update.1 It is important to distinguish this from a true filesystem snapshot (like those in ZFS or LVM). A zpaqfranz snapshot is a logical, versioned view of the files and directories as they existed at the time of a specific backup operation.

Each time an archive is updated, a new transaction is appended, recording which files were added, modified, or deleted. Because old data is never removed, users can effectively "travel back in time" to view or restore the state of their data from any previous backup. This functionality, conceptually similar to Apple's Time Machine or the versioning in Git, allows for the retention of thousands of historical versions within a single, space-efficient archive file.1 This history can be navigated and accessed using the \-until switch, which allows specifying a version number or a date and time to roll the archive back to a particular point in time.4

#### **The ZPAQL Virtual Machine**

A foundational and highly advanced feature of the ZPAQ format is its use of ZPAQL, a sandboxed, virtual machine language designed specifically for describing compression algorithms.3 The ZPAQL code that defines the exact compression method used (e.g., LZ77, BWT, context mixing) is embedded directly within the archive header.

This self-describing format provides exceptional forward compatibility. A future version of zpaqfranz could introduce a novel, highly advanced compression algorithm, and archives created with it could still be decompressed by any older version of the program that supports the ZPAQ Level 1 format (dating back to 2009). The older program simply reads the ZPAQL bytecode from the archive and executes it in its virtual machine to decompress the data.4

On x86-32 and x86-64 processors, this ZPAQL code is Just-In-Time (JIT) compiled into native machine code for maximum performance, making it as fast as algorithms written in compiled languages like C++. On other architectures (such as ARM or SPARC64), the ZPAQL code is interpreted, which is typically slower but ensures universal compatibility.4

## **Part 2: Installation and Configuration**

This section provides practical guidance for installing and configuring zpaqfranz across various operating systems. It clarifies the different pre-compiled binaries available, outlines installation via package managers, and details the process of compiling from source, including the critical compile-time flags required for specific hardware and software environments.

### **2.1 Choosing the Right Binary Version**

For Windows users, the zpaqfranz project provides several pre-compiled executables, each tailored for a specific use case. Understanding the differences is key to selecting the optimal version for a given environment. These binaries are typically available from the project's GitHub releases page or its SourceForge mirror.1

The following table summarizes the available Windows binaries and their intended purposes:

| Binary Name | Target Platform/Architecture | Key Features & Characteristics | Recommended Use Case |
| :---- | :---- | :---- | :---- |
| zpaqfranz.exe | Windows 64-bit | Auto-detects CPU support for hardware-accelerated SHA1/SHA2. Can self-update and download required DLLs from the internet. | The standard, recommended version for most modern 64-bit Windows systems with an internet connection.1 |
| zpaqfranz32.exe | Windows 32-bit | 32-bit version with limitations on memory usage and available threads. Noticeably slower than 64-bit versions. | Legacy 32-bit Windows systems where a 64-bit executable cannot be run.1 |
| zpaqfranzhw.exe | Windows 64-bit | Uses a different codebase for hardware-accelerated SHA1 but requires the \-hw switch to be explicitly enabled. | Systems where the standard binary's auto-detection may fail or for users who prefer explicit control over hardware acceleration.1 |
| zpaqfranz-full.exe | Windows 64-bit | A statically linked version containing all necessary libraries. It does not require an internet connection to download dependencies. | Air-gapped or offline environments, or for creating portable deployments where internet access is unavailable or restricted.1 |
| zpaqfranz-open.exe | Windows 64-bit | Stripped of all binary blobs and non-open-source components. Cannot create temporary files. | High-security environments where a fully auditable, 100% open-source toolchain is mandated.1 |
| zpaqfranzxp.exe | Windows 32-bit | A specialized 32-bit build with specific workarounds and limitations to maintain compatibility with Windows XP. | The glorious but ancient Windows XP operating system.1 |

This structured choice allows administrators to deploy zpaqfranz appropriately, whether on a standard workstation, a legacy system, or within a highly secure, isolated network.

### **2.2 Installation Across Platforms**

For most modern operating systems, zpaqfranz can be installed easily using the native package manager, which is the recommended method as it handles dependencies and system integration automatically.

* Debian / Ubuntu (and derivatives):  
  sudo apt-get install zpaqfranz  
* FreeBSD:  
  sudo pkg install zpaqfranz  
* OpenBSD:  
  sudo pkg\_add zpaqfranz  
* macOS (via Homebrew):  
  brew install zpaqfranz  
* OpenSUSE:  
  sudo zypper install zpaqfranz

For systems where zpaqfranz is not available in the official repositories, or for users who wish to install it manually, the process involves downloading the appropriate binary and placing it in a directory included in the system's PATH environment variable (e.g., /usr/local/bin on Linux or FreeBSD).

### **2.3 Compiling from Source**

One of the design goals of zpaqfranz is ease of compilation. The entire program is largely contained within a single C++ source file (zpaqfranz.cpp), allowing it to be compiled with a simple command and without complex build systems like Makefiles, although a minimal Makefile is provided for convenience.11 This is particularly advantageous for compiling on non-standard or resource-constrained systems like NAS devices or hypervisor shells.12

The standard compilation command on a Unix-like system is:  
g++ \-O3 \-march=native \-Dunix zpaqfranz.cpp \-pthread \-o zpaqfranz  
This command uses the GNU C++ compiler (g++) with high optimization (-O3), targets the native CPU architecture for best performance (-march=native), defines the target as Unix-like (-Dunix), links the required POSIX threads library (-pthread), and outputs the final executable named zpaqfranz.14

For certain environments, specific compile-time flags (defined with \-D) are necessary to enable or disable features and ensure compatibility. These flags are critical for successful compilation on non-x86 or specialized hardware.2

* \-DNOJIT: Disables the Just-In-Time (JIT) compilation of ZPAQL to native x86 machine code. This is **mandatory** for non-x86 processors, such as ARM, Apple Silicon (M1/M2), POWER, and SPARC. On these platforms, ZPAQL will be interpreted instead.2  
* \-DSOLARIS: Enables specific code paths and workarounds required for compilation and proper operation on Solaris-based operating systems.2  
* \-DANCIENT: For very old systems, such as cheap NAS devices or legacy servers. This flag may disable certain modern features or use alternative, more compatible code to function on older compilers or kernels.2  
* \-DBIG: For big-endian architectures, where the most significant byte is stored at the smallest memory address. This is required for platforms like SPARC64 and some MIPS or PowerPC systems.2  
* \-DESX: Enables specific settings for compiling directly within a VMware ESXi hypervisor shell.2  
* \-DALIGNMALLOC: Enforces strict memory alignment. This is necessary for certain RISC CPUs, such as SPARC64 and HP-PA, that will fault if data is not aligned on specific memory boundaries.2  
* \-DHWSHA2: Explicitly enables support for hardware-accelerated SHA2 instructions, which can provide a significant performance boost on compatible CPUs (e.g., modern AMD Ryzen processors).2

After compiling, the autotest command can be run to perform a series of internal checks, verifying that the compiled binary is functioning correctly on the target system.2

## **Part 3: Comprehensive Command Reference**

This section provides a detailed reference for all commands available in zpaqfranz. The commands are grouped by their primary function to facilitate quick lookups and a better understanding of the tool's capabilities. Each entry includes the command's purpose, syntax, and operational details.

### **3.1 Syntax and Help System**

The general syntax for zpaqfranz is consistent across all commands:

zpaqfranz command archive\[".zpaq"\]\[files|directory\]... \[-switches\] 2

* command: The specific action to perform (e.g., a, x, l).  
* archive: The path to the archive file. The .zpaq extension is assumed if not provided.  
* files|directory: Optional list of files or directories to process.  
* \-switches: Optional flags that modify the command's behavior.

**Multipart Archives:** zpaqfranz can treat a sequence of numbered files as a single, unified archive. This is invoked by using wildcards in the archive name. An asterisk (\*) matches the part number, while a question mark (?) matches a single digit. For example, backup\_??.zpaq would match backup\_01.zpaq, backup\_02.zpaq, etc., concatenating them in numerical order starting from 1\. When using wildcards, it is critical to enclose the archive name in double quotes to prevent the shell from expanding them.2

**Embedded Help System:** The primary source of documentation is the comprehensive help system built directly into the executable. The developer strongly encourages its use.5

* zpaqfranz h: Displays a list of all available commands and switches.  
* zpaqfranz h h: Provides the full, unabridged help text for all features ("ALL IN").  
* zpaqfranz h: Shows detailed help for a specific command.  
* zpaqfranz \-he: Displays practical usage examples for a specific command.

### **3.2 Core Archive Management Commands**

These commands form the foundation of zpaqfranz, handling the creation, extraction, and inspection of archives.

* **a (add)**: Adds files and directories to an archive. If the archive does not exist, it will be created. If it exists, this command appends a new version (snapshot) containing the added or modified files. This is the primary command for creating and updating backups.2  
* **x (extract)**: Extracts the most recent version of files from an archive, recreating the original directory structure. By default, it will not overwrite existing files unless the \-force switch is used.2  
* **l (list)**: Lists the files and directories contained in the most recent version of the archive. Various switches, such as \-all, can be used to display all historical versions of every file.2  
* **e**: Extracts files to the current directory, ignoring their original path information. This is useful for extracting specific files into a flat structure.2  
* **i**: Displays detailed information about an archive, including the number of versions, total files, fragment and block counts, and total size.2  
* **trim**: Trims an archive by removing the last, potentially incomplete transaction. This is a recovery command used if an add operation was interrupted, ensuring the archive is left in a consistent state.2  
* **m (merge)**: Consolidates a multipart archive (e.g., arc01.zpaq, arc02.zpaq) into a new, single-file archive. This is useful for long-term storage or distribution after a series of incremental backups.2  
* **password**: Allows changing or removing the encryption password of an existing single-file (non-multipart) archive.2

### **3.3 Backup and System-Level Commands**

This group of commands extends zpaqfranz into a more specialized backup and system administration tool, offering functionality beyond basic archiving.

* **backup**: A specialized command for creating "hardened" multipart backups. Unlike a standard multipart add, this command also generates index files (\_backup.txt, \_backup.index) that store a list of all archive parts along with their sizes and cryptographic hashes (MD5 by default, XXH3 with \-backupxxh3). This index allows for rapid and robust verification of the entire backup set's integrity, ensuring no parts are missing, corrupted, or out of order—a common vulnerability in simple multipart archives.15  
* **w**: Provides a "chunked" extraction or test mechanism for very large files. This command is a performance optimization for systems with ample RAM and slow magnetic disks. It works by loading large sections of the archive into memory before writing to disk, thereby minimizing slow disk seek operations and improving overall extraction speed.1  
* **k (kill)**: A high-risk but powerful command that synchronizes a destination directory to match the contents of an archive. It will delete any files in the destination directory that are *not* present in the specified archive version, making it functionally similar to using rsync with the \--delete flag. This should be used with extreme caution.2  
* **ntfs**: A command for working with disk images of NTFS partitions. Its primary function is to regenerate a full, sector-by-sector disk image from a compact image created with a \-image \-ntfs. The command reconstructs the original partition by filling the unused sectors with zeros. This is an experimental feature intended for bare-metal recovery scenarios.10  
* **mysqldump**: While not a standalone command in recent versions, zpaqfranz is explicitly designed to be highly effective for backing up mysqldump or mariadump outputs. The versioning and deduplication are ideal for storing daily text-based database dumps, as only the changes between dumps are stored, resulting in massive space savings.1

### **3.4 Data Integrity and Verification Commands**

This suite of commands embodies the "paranoia as a feature" philosophy of zpaqfranz, providing multiple layers of data verification to ensure archive integrity and restoration fidelity.

* **t (test)**: Performs a standard integrity test on an archive. It verifies headers, decompresses data to check for errors, and validates internal checksums. This command is a more intuitive replacement for the original zpaq x \-test syntax and can operate on multiple archives at once (e.g., zpaqfranz t \*.zpaq).18  
* **p (paranoid test)**: A significantly more rigorous and resource-intensive test. It performs all the checks of the t command but may also involve additional, deeper verification steps. This command requires more time and memory and is intended for situations where the highest possible assurance of integrity is required.2  
* **v (verify)**: Compares the contents of an archive against the files on the live filesystem. It re-calculates hashes of the local files and compares them to the hashes stored in the archive to detect any discrepancies, confirming that the archived versions match the source files.2  
* **testbackup**: Works in conjunction with the backup command. It reads the index file (\_backup.txt) and verifies that all multipart archive chunks are present, have the correct size, and match their stored cryptographic hashes. This provides a fast and reliable way to audit the integrity of a large, distributed backup set without reading the content of every file.2  
* **sum**: A versatile utility command for filesystem integrity checks. It can calculate a wide variety of cryptographic hashes and checksums for files on disk. A primary use case is to identify duplicate files within a directory tree by comparing their hashes.2  
* **versum**: A powerful command for end-to-end verification, similar in function to hashdeep. It takes a manifest file (a list of filenames and their corresponding hashes) and verifies it against a target directory or a .zpaq archive. This allows a user to generate a hash list on a source machine, restore the data to a completely different machine, and then use versum to prove that the restored data is bit-for-bit identical to the original, providing ultimate confidence in the backup and restore process.2

The following table details the various hash algorithms supported by zpaqfranz for its verification and integrity-checking commands.

| Algorithm Name | Activating Switch | Security Level | Relative Speed | Primary Use Case |
| :---- | :---- | :---- | :---- | :---- |
| XXHASH64 | (default in sum) | Non-cryptographic | Extremely Fast | Default for file hashing and integrity checks where speed is paramount. |
| CRC-32 | (internal default) | Non-cryptographic | Extremely Fast | Internal error detection; excellent at detecting accidental corruption. |
| MD5 | \-md5 | Cryptographically Broken | Very Fast | Legacy compatibility, quick integrity checks. Not for security. |
| SHA-1 | \-sha1 | Cryptographically Weak | Fast | Full-file SHA-1 check for higher integrity than default fragment hashes. |
| XXH3-128 | \-xxh3 | Non-cryptographic | Extremely Fast | A modern, high-performance non-cryptographic hash for checksumming. |
| BLAKE3 | \-blake3 | Cryptographically Secure | Very Fast | A modern, highly parallelizable secure hash offering excellent performance. |
| SHA-2-256 | \-sha256 | Cryptographically Secure | Moderate | A widely trusted NIST standard for strong cryptographic hashing. |
| SHA-3-256 | \-sha3 | Cryptographically Secure | Slow | The latest NIST secure hashing standard, offering a different internal design from SHA-2. |
| WHIRLPOOL | \-whirlpool | Cryptographically Secure | Slow | An ISO/IEC standard secure hash function. |

### **3.5 Filesystem and Utility Commands**

zpaqfranz includes a broad set of commands that are not directly related to archiving but extend its functionality into a comprehensive toolkit for system administrators. This reflects a deliberate expansion of scope, turning the program into a multipurpose file management utility.

* **1on1**: Compares two folders and deletes files in the second folder that have the same name and hash as files in the first folder.2  
* **c (compare)**: Compares a master directory against one or more slave directories to find differences.2  
* **cp**: A file copy utility that provides an estimated time of arrival (ETA) and is resumable.2  
* **d (deduplicate)**: Finds and hard-links duplicate files within a single folder, saving disk space at the filesystem level.2  
* **dir**: A command that provides directory listings, similar in function to the dir command on Windows.2  
* **dirsize**: Scans and displays the cumulative size of one or more folders.2  
* **f (fill)**: A disk utility command that can either fill the free space on a drive with data (for reliability testing) or wipe it with zeros (for privacy).2  
* **find**: Searches for files using wildcards.2  
* **isopen**: Checks if a specified file is currently open and locked by another process (Windows-specific).2  
* **last**: A helper command for multipart archives that returns the filename of the last part in a sequence.2  
* **n (decimate)**: A file management command that prunes older files in a directory, keeping only the newest X number of files.2  
* **r (robocopy)**: A powerful file synchronization command that replicates a master directory to one or more slave directories, similar to the functionality of Microsoft's Robocopy.2  
* **rd**: A utility to remove directories that are difficult to delete on Windows, such as those with paths exceeding the MAX\_PATH limit.2  
* **rsync**: A cleanup utility designed to find and delete dangling temporary files left behind by rsync operations.2  
* **s**: A utility to get the size of directories and report the remaining free disk space.2  
* **utf**: A filename utility to convert filenames to Latin character sets or fix issues with overly long filenames.2  
* **z**: Scans and removes empty directories from a specified path.2

### **3.6 ZFS Integration Commands**

zpaqfranz features a unique and powerful set of commands designed for deep integration with the ZFS filesystem. These commands leverage ZFS's native snapshot and replication capabilities to create extremely efficient and robust backup workflows.

* **zfsadd**: "Freezes" ZFS snapshots by storing their metadata within the zpaq archive, creating a record of the snapshot's existence at a point in time.2  
* **zfsbackup**: The core command for ZFS integration. It automates the process of creating a new ZFS snapshot and then using zfs send to stream the incremental difference between the new snapshot and the previous one directly into the .zpaq archive. This is exceptionally fast and bandwidth-efficient as ZFS itself calculates the data that has changed, eliminating the need for a full filesystem scan.20  
* **zfslist**: Lists ZFS snapshots on the system, supporting wildcards for filtering the output.2  
* **zfspurge**: Purges (deletes) old ZFS snapshots, often used in scripts to manage snapshot retention after they have been successfully backed up.2  
* **zfsreceive**: The primary command for restoring ZFS data. It extracts the ZFS stream data from a .zpaq archive and pipes it to a zfs recv command to recreate the dataset and its snapshots on a target pool. It can also generate a shell script of the required commands for manual review and execution.2  
* **zfsrestore**: A helper command used in the ZFS restoration process, typically for handling extracted .zfs stream files.2  
* **zfsproxbackup / zfsproxrestore**: Experimental commands designed for backing up and restoring Proxmox Virtual Environment (PVE) virtual machines that are stored on ZFS datasets.2

## **Part 4: Comprehensive Switch Reference**

This section categorizes and explains the command-line switches that modify the behavior of zpaqfranz commands. The switches are a source of the utility's power and flexibility, but their sheer number can be daunting. They are grouped here into logical categories: main switches inherited from the original zpaq, switches specific to zpaqfranz's new features, and advanced switches for specialized use cases.

### **4.1 Main Switches (Inherited from zpaq)**

These switches provide the core functionality for compression, encryption, and version control, and are largely compatible with the original zpaq 7.15.

* **\-key \[password\]**: Encrypts the archive data using AES-256. If a password is not provided on the command line, zpaqfranz will interactively prompt for one, and will ask for confirmation to prevent typos. Using \-key without a password is the recommended secure practice.3  
* **\-method \[0-5\]**: Sets the compression level, trading speed for compression ratio.  
  * 0: No compression. Data is stored raw, but deduplication is still performed. Useful for archiving already-compressed files where the goal is versioning and deduplication.  
  * 1: The default level. Uses fast LZ77 compression, suitable for regular backups where compression speed is important.  
  * 2-3: Intermediate levels offering better compression at the cost of speed.  
  * 4-5: The strongest compression levels, using advanced context modeling. These can be very slow and memory-intensive but achieve superior compression ratios.3  
* **\-force**: Overrides default safety checks. When used with add, it forces zpaqfranz to re-evaluate files even if their modification date hasn't changed, adding them if their content hash is different. When used with extract, it allows overwriting existing files in the destination directory.4  
* **\-until**: This is the primary switch for accessing historical data. It instructs zpaqfranz to treat the archive as if it ended at a specific point in time. It can take a version number (e.g., \-until 5 for the state after the 5th update, or \-until \-1 for the state before the very last update) or a precise date and time. This affects list, extract, and even add operations (where it will truncate the archive at that point before appending new data).4  
* **\-threads \[N\]**: Specifies the number of CPU threads to use for compression and hashing. By default, zpaqfranz attempts to use all available processor cores.8

### **4.2 zpaqfranz-Specific Switches**

These switches enable the new features and enhancements introduced in the zpaqfranz fork. They represent a departure from the original's feature set, introducing capabilities for improved metadata handling, system imaging, and compatibility control.

A key consideration with these new features is the potential impact on backward compatibility. While zpaqfranz is designed to produce archives that can be read by the original zpaq 7.15, using switches that store new types of metadata (like \-tar) may result in that specific metadata being ignored by the older program. To address this, zpaqfranz provides a specific switch to enforce strict compatibility. The \-715 switch acts as a "compatibility mode," disabling any zpaqfranz extensions that would violate the original ZPAQ format specification. This gives users a clear choice: leverage advanced features for modern backup workflows or create universally compatible archives for maximum long-term interoperability.18

* **\-715**: Enables strict compatibility mode with zpaq v7.15. When this switch is used, any zpaqfranz-specific features that would alter the archive format are disabled, ensuring the resulting archive is 100% readable and correctly interpreted by the original zpaq.  
* **\-tar**: When used with the add or a command on Unix-like systems, this switch saves file metadata, including access rights (permissions), user ID, and group ID, within the archive. When used with extract or x, it restores this metadata. The list or l command will display the stored metadata when this switch is present.10  
* **\-image**: Used with the add command to perform a raw, sector-by-sector backup of a block device (e.g., a hard drive partition), similar to the dd utility. This creates a full image of the partition for disaster recovery.10  
* **\-ntfs**: A dual-purpose switch for working with the NTFS filesystem.  
  1. **With \-image**: When creating a disk image, this switch makes the process "NTFS-aware," causing zpaqfranz to store only the *used* sectors of the partition, resulting in a much smaller image file than a full sector-by-sector copy.  
  2. **Without \-image**: When adding files from an NTFS volume, this switch dramatically speeds up the initial file enumeration process. Instead of traversing the directory tree, it reads the Master File Table (MFT) directly, a technique similar to that used by the "Everything" search utility on Windows.10  
* **\-ramdisk**: An experimental performance enhancement switch that instructs zpaqfranz to use a RAM disk for its temporary files during decompression. This can significantly speed up extraction, especially for large archives, by avoiding I/O bottlenecks on physical disks.23  
* **\-franzhash**: A specialized, parallel-friendly hash format developed for zpaqfranz. It can provide significantly faster hashing performance on multi-core CPUs for very large files compared to standard sequential hashing.10  
* **\-frugal**: When used with VSS (Volume Shadow Copy Service) backups on Windows, this switch instructs zpaqfranz to back up only common user data folders (Desktop, Downloads, etc.), excluding program files and the operating system itself. This allows for smaller, more focused user-state backups.1  
* **\-longpath**: Enables support for file paths exceeding the traditional 260-character limit on Windows, ensuring that files in deep directory structures are handled correctly.18  
* **\-tmp**: When creating multipart archives, this switch causes new parts to be created with a .tmp extension. Only after a part is successfully and completely written is it renamed to .zpaq. This is a critical feature for backups to network locations that are monitored by file synchronization software (like Syncthing or Dropbox), as it prevents the software from trying to upload an incomplete archive part.15  
* **\-nojit**: A runtime switch that provides the same functionality as the \-DNOJIT compile-time flag. It disables the JIT compilation of ZPAQL, forcing the interpreter to be used. This can be a useful diagnostic tool or a workaround on systems where the JIT compiler is unstable.15

### **4.3 Advanced and "Voodoo" Switches**

This category, as named in the embedded help, includes switches for niche use cases, fine-grained control over hashing, and performance tuning.

* **Hashing Algorithm Switches (-md5, \-sha1, \-sha256, \-sha3, \-blake3, \-xxh3, etc.)**: These switches are used with commands like sum and backup to specify which cryptographic hash or checksum algorithm to use. For example, zpaqfranz backup archive.zpaq files \-backupxxh3 will use the XXH3 algorithm for the integrity file instead of the default MD5.15  
* **\-ssd**: A performance optimization switch used with various commands (like sum or versum) to indicate that the operation is being performed on solid-state drives. This may enable more aggressive multithreading or different I/O patterns better suited for high-IOPS devices.17  
* **\-nomore**: Disables the internal "more" pager that pauses output on long listings. This is useful when redirecting the output of zpaqfranz to another command for processing, such as grep or less (e.g., zpaqfranz h h \-nomore | less).15  
* **\-shutdown**: A switch available for the add command on Windows that will shut down the computer after the backup operation completes successfully.10  
* **\-monitor**: A Windows-only switch for the add command that monitors for system inactivity before starting the backup process.10  
* **\-quick**: Used with the sum command, this switch calculates a "similarity hash" instead of a full-file hash. For files larger than 64 KB, it hashes only the head, middle, and tail of the file. This is not a true hash but is extremely fast and useful for quickly finding likely duplicates or performing a rapid, non-cryptographic check of a backup set.17

## **Part 5: Advanced Use Cases and Workflows**

This section provides practical, step-by-step guides for accomplishing common but complex tasks with zpaqfranz. These workflows synthesize information from scattered examples, release notes, and forum discussions into coherent, actionable procedures for system administrators and advanced users.

### **5.1 Workflow: Incremental Backups to a Network Share**

This workflow details how to set up a robust, versioned, incremental backup to a network share or a cloud storage folder that is synchronized from a local directory. It leverages the backup command for integrity and the \-tmp switch for safe synchronization.

**Goal:** Create a daily backup of a project folder (C:\\Projects) to a local staging directory (D:\\BackupStaging) which is then synchronized to a remote location by a tool like Syncthing or Dropbox.

1. **Initial Backup:** The first step is to create the initial hardened, multipart backup. The backup command is used, and the ? wildcard in the archive name tells zpaqfranz to create multipart files.  
   Bash  
   zpaqfranz backup "D:\\BackupStaging\\projects\_????.zpaq" C:\\Projects \-key MySecretPassword \-tmp

   * backup: Uses the hardened backup mode, which creates integrity-checking index files.  
   * "projects\_????.zpaq": Defines the multipart archive name. The ???? will be replaced by part numbers like 0001, 0002, etc.  
   * \-key: Encrypts the archive.  
   * \-tmp: This is the critical switch for this workflow. New archive parts will be created as projects\_0001.tmp, and only renamed to projects\_0001.zpaq upon successful completion. This prevents the sync client from uploading a partial, corrupted file.15  
2. **Subsequent Backups:** For daily updates, the exact same command is run.  
   Bash  
   zpaqfranz backup "D:\\BackupStaging\\projects\_????.zpaq" C:\\Projects \-key MySecretPassword \-tmp

   zpaqfranz will read the existing archive parts, identify changed or new files in C:\\Projects, and append only the new data in a new part file (e.g., projects\_0005.zpaq). The \_backup.txt index file will be updated to include the new part and its hash.  
3. **Verification:** Periodically, the integrity of the entire backup set can be verified quickly without reading all the data.  
   Bash  
   zpaqfranz testbackup "D:\\BackupStaging\\projects\_????.zpaq"

   This command reads the index file and checks that all listed .zpaq parts exist, have the correct size, and match their stored hash, confirming the integrity of the backup set.5  
4. **Restoration:** To restore the latest version of a specific file:  
   Bash  
   zpaqfranz x "D:\\BackupStaging\\projects\_????.zpaq" C:\\Projects\\path\\to\\file.txt \-to C:\\Restore \-key MySecretPassword

   To restore the entire project folder as it existed before the last backup, use the \-until \-1 switch:  
   Bash  
   zpaqfranz x "D:\\BackupStaging\\projects\_????.zpaq" \-until \-1 \-to C:\\Restore \-key MySecretPassword

### **5.2 Workflow: Full System Backup and Restore on Windows**

zpaqfranz offers two distinct methods for backing up a Windows system drive: a file-level backup of user data and a full image-level backup for bare-metal recovery.

#### **Method 1: File-Level VSS Backup of User Data**

**Goal:** Create a daily backup of essential user data from a live Windows system without backing up the OS or applications.

1. **Backup Command:** This method uses the Volume Shadow Copy Service (VSS) to access files that are in use and the \-frugal switch to target user directories.  
   Bash  
   zpaqfranz a C:\\Backups\\UserData.zpaq C:\\ \-vss \-frugal \-key MyPassword

   * C:\\: The source is the root of the C: drive.  
   * \-vss: Engages the Volume Shadow Copy Service to create a temporary, consistent snapshot of the drive, allowing locked files (like Outlook PSTs or databases) to be backed up safely.  
   * \-frugal: A specialized switch that restricts the backup to common user-profile folders (%USERPROFILE%, Desktop, Documents, Downloads, etc.), ignoring Windows system files and Program Files.1

This creates a compact, versioned backup of just the user's critical data, which can be easily restored to another machine.

#### **Method 2: Image-Level Backup for Bare-Metal Recovery**

**Goal:** Create a full, sector-level image of the Windows system partition (C:) for complete disaster recovery.

1. **Backup Command:** This method uses the \-image and \-ntfs switches to create an efficient disk image. It should be run from a recovery environment or another OS, as backing up a live system partition at the block level is not recommended.  
   Bash  
   zpaqfranz a D:\\Backups\\C\_Drive.zpaq \\\\.\\C: \-image \-ntfs \-key MyPassword \-m2

   * \\\\.\\C:: The source is the block device for the C: partition.  
   * \-image: Specifies a raw, sector-level backup mode.10  
   * \-ntfs: Makes the imaging process NTFS-aware, causing it to back up only the sectors that are currently in use by the filesystem. This dramatically reduces the size of the backup image compared to a full dd-style copy.10  
   * \-m2: Using a moderate compression level is effective for disk images.  
2. Restoration Process: zpaqfranz does not have a built-in function to directly restore an image to a disk. The process requires two steps:  
   a. Extract the Image: First, extract the raw image file from the archive.  
   Bash  
   zpaqfranz x D:\\Backups\\C\_Drive.zpaq \-to E:\\Restore \-key MyPassword

   This will produce a file like E:\\Restore\\C\_Drive.img.  
   b. **Use External Tools:** Use a third-party disk imaging tool (like dd on Linux, or OSFMount/Rufus on Windows) to write the extracted .img file back to a physical partition. Alternatively, the image can be mounted as a virtual drive using OSFMount or the Windows Disk Management tool to browse and extract individual files.1

### **5.3 Workflow: High-Efficiency VM Backups with ZFS Integration**

This workflow, based on procedures discussed by advanced users, demonstrates how to leverage zpaqfranz's deep ZFS integration to create extremely fast and space-efficient incremental backups of datasets, which is ideal for virtual machine storage.

**Goal:** Back up a ZFS dataset (tank/vms) containing virtual machine disk images to a separate backup pool (backuppool/archives).

1. **Initial Backup:** The first backup uses zfsbackup to create an initial snapshot and stream a full copy of the dataset into a new zpaqfranz archive.  
   Bash  
   zpaqfranz zfsbackup /backuppool/archives/vms.zpaq tank/vms

   This command will automatically:  
   * Create a new ZFS snapshot on tank/vms (e.g., tank/vms@zpaqfranz\_00000001).  
   * Perform a zfs send of this full snapshot.  
   * Compress and encrypt (if \-key is used) the stream, storing it in vms.zpaq.  
2. **Incremental Backups:** Subsequent runs of the same command perform highly efficient incremental backups.  
   Bash  
   zpaqfranz zfsbackup /backuppool/archives/vms.zpaq tank/vms \-kill

   On the second run, zpaqfranz will:  
   * Create another snapshot (e.g., tank/vms@zpaqfranz\_00000002).  
   * Perform an incremental zfs send \-i @zpaqfranz\_00000001 @zpaqfranz\_00000002.  
   * Append only this small incremental stream to vms.zpaq.  
   * The \-kill switch will then delete the older snapshot (@zpaqfranz\_00000001), keeping the snapshot count on the source system minimal.20 This process is extremely fast because ZFS tracks the changed blocks, eliminating any need for file-system scanning.20  
3. **Restoration:** Restoration is a guided process using zfsreceive.  
   Bash  
   zpaqfranz zfsreceive /backuppool/archives/vms.zpaq tank/restored\_vms \-out restore\_script.sh

   This command does not perform the restore directly. Instead, it extracts the ZFS streams from the archive and generates a shell script (restore\_script.sh) containing the exact zfs recv commands needed to rebuild the dataset and all its snapshots in sequence on the new location (tank/restored\_vms). The user must then review and execute this script manually. This is a critical safety feature to prevent accidental data loss, as the tool currently lacks exhaustive input checks for this powerful operation.20

### **5.4 Workflow: Creating Self-Extracting Archives (SFX) for Windows**

For easy distribution of archived files to Windows users who may not have zpaqfranz installed, a self-extracting archive (SFX) can be created. This requires the separate zsfx.exe utility.

**Goal:** Package the contents of my\_project.zpaq into a single executable file my\_project\_installer.exe.

1. **Create the Archive:** First, create a standard .zpaq archive.  
   Bash  
   zpaqfranz a my\_project.zpaq C:\\path\\to\\project\\\*

2. **Build the SFX Executable:** Use the zsfx command to combine the SFX module with the archive.  
   Bash  
   zsfx x my\_project.zpaq \-output my\_project\_installer.exe

   * zsfx x: The command to create the SFX.  
   * my\_project.zpaq: The source archive.  
   * \-output...: Specifies the name of the final executable.24

The resulting my\_project\_installer.exe is a standalone program. When run, it will extract the contents of my\_project.zpaq. Extraction options like \-to can be passed to the SFX creation command to control the default extraction path.24 Note that there is a Windows limitation that the final .exe file cannot exceed 2 GB in size.24

### **5.5 Workflow: Cross-Platform Paranoid Data Verification**

This workflow provides the highest level of assurance that data restored from a backup is bit-for-bit identical to the original source data, even if the restoration occurred on a different operating system.

**Goal:** Verify the integrity of files restored from a backup of a Linux server (/srv/data) onto a Windows machine (D:\\RestoredData).

1. **Generate Hash Manifest on Source:** On the original Linux server, use the sum command to create a manifest file containing the SHA-256 hashes of all source files.  
   Bash  
   zpaqfranz sum /srv/data \-sha256 \-all \> /tmp/source\_hashes.txt

   This creates a text file (source\_hashes.txt) where each line contains a SHA-256 hash and the corresponding file path.  
2. **Perform Backup and Restore:** Use any standard zpaqfranz method to back up /srv/data and restore it to D:\\RestoredData on the Windows machine.  
3. **Verify on Destination:** On the Windows machine, use the versum command to verify the restored files against the original hash manifest.  
   Bash  
   zpaqfranz versum C:\\path\\to\\source\_hashes.txt \-find /srv/data/ \-replace D:\\RestoredData\\

   * versum: The command to verify a hash list against a target.  
   * source\_hashes.txt: The manifest file generated on the source machine.  
   * \-find / \-replace: These switches are used to translate the file paths from the Linux format in the manifest to the Windows format of the restored data. versum will read each line, replace /srv/data/ with D:\\RestoredData\\, and then calculate the SHA-256 hash of the local file and compare it to the hash in the manifest.19

The versum command will report any files that are missing, have been altered, or do not match their original hash, providing a definitive, cross-platform audit of the restoration's integrity.

## **Part 6: Technical Appendix**

This final section contains supplementary materials for quick reference and provides context on the project's development and documentation philosophy.

### **6.1 The zpaqfranz Community and Documentation Model**

Users accustomed to centrally managed, formal documentation may find the information for zpaqfranz to be distributed across various sources, including man pages, GitHub READMEs, release notes, and community forum threads. This is a direct reflection of the project's nature as a hobbyist endeavor driven by a single developer.

The lead developer, Franco Corbelli, has expressed that writing comprehensive, static documentation is often a thankless task for a niche project with a small user base. Detailed guides can become excessively long and complex, while brief notes may be insufficient. As a result, the project has adopted a more dynamic and interactive documentation model.16 The primary venue for detailed technical discussions, bug reports, and feature requests is the "Issues" section of the official GitHub repository. The developer is highly responsive in this forum and often provides in-depth explanations and solutions to specific user queries.

Therefore, users should consider the GitHub Issues section not just as a bug tracker, but as a living, searchable knowledge base that contains the most current information and detailed discussions on the tool's advanced features and edge cases. While this report aims to consolidate that knowledge, the GitHub repository remains the canonical source for direct interaction with the developer and community.

### **6.2 Quick Reference Tables**

The following tables provide a concise summary of zpaqfranz commands and common switches for quick reference during daily use.

#### **Table 6.2.1: Complete Command Summary**

| Command | Description |
| :---- | :---- |
| **Core Archive** |  |
| a, add | Add/append files to an archive, creating a new version. |
| x, extract | Extract files from an archive, preserving paths. |
| l, list | List archive contents. |
| e | Extract files to the current folder, ignoring paths. |
| i | Display detailed archive information. |
| trim | Remove an incomplete transaction from the end of an archive. |
| m | Merge a multipart archive into a single file. |
| password | Change or remove an archive's password. |
| **Backup & System** |  |
| backup | Create a hardened, indexed multipart backup. |
| w | Perform a chunked (RAM-based) extraction/test for large files. |
| k | Kill (delete) files in a destination not present in the archive. |
| ntfs | Regenerate a full disk image from a \-ntfs backup. |
| upgrade | Self-update the zpaqfranz executable (Windows). |
| **Verification** |  |
| t | Test archive integrity. |
| p | Perform a slow, paranoid test of archive integrity. |
| v | Verify archive contents against the live filesystem. |
| testbackup | Verify the integrity of a hardened multipart backup set. |
| sum | Calculate file hashes/checksums and find duplicates on disk. |
| versum | Verify files against a hashdeep-style manifest. |
| **Filesystem Utilities** |  |
| 1on1 | Delete files in folder2 that are identical to files in folder1. |
| c | Compare directories to find differences. |
| cp | Resumable file copy with ETA. |
| d | Deduplicate files on disk using hard links. |
| dir | Display a directory listing. |
| dirsize | Show cumulative folder sizes. |
| f | Fill or wipe free disk space. |
| find | Search for files using wildcards. |
| isopen | Check if a file is locked by another process. |
| n | Decimate (prune) old files in a directory. |
| r | robocopy-like directory synchronization. |
| rd | Remove hard-to-delete directories (e.g., long paths). |
| rsync | Clean up temporary files from rsync. |
| s | Get directory size and free disk space. |
| utf | Fix filenames (convert to Latin, shorten long names). |
| z | Remove empty directories. |
| **ZFS Integration** |  |
| zfsbackup | Back up a ZFS incremental send stream. |
| zfslist | List ZFS snapshots. |
| zfspurge | Delete ZFS snapshots. |
| zfsreceive | Restore a ZFS stream from an archive. |

#### **Table 6.2.2: Common Switches Quick Reference**

| Switch | Associated Commands | Description |
| :---- | :---- | :---- |
| \-key \[pass\] | a, x, l, etc. | Encrypt or decrypt the archive with AES-256. |
| \-method \[0-5\] | a | Set the compression level (0=none, 1=fast, 5=max). |
| \-until \[N|date\] | a, x, l | Roll back the archive to a specific version or time. |
| \-force | a, x | Force overwrite on extract or re-check on add. |
| \-threads \[N\] | (most) | Set the number of CPU threads to use. |
| \-715 | a | Enforce strict compatibility with original zpaq 7.15. |
| \-tar | a, x, l | Preserve/restore Unix permissions and ownership. |
| \-image | a | Perform a raw, sector-level backup of a device. |
| \-ntfs | a | Optimize for NTFS (store used sectors only with \-image, or fast scan). |
| \-vss | a | Use Volume Shadow Copy Service on Windows for live backups. |
| \-tmp | backup, a | Create multipart chunks with a .tmp extension first. |
| \-kill | zfsbackup | Delete the previous ZFS snapshot after a successful backup. |

## **Conclusion**

zpaqfranz stands as a testament to the power of focused, expert-driven open-source development. By building upon the revolutionary foundation of Matt Mahoney's zpaq, it has evolved into a tool of remarkable depth and capability, tailored specifically for individuals and organizations with stringent requirements for data integrity, storage efficiency, and long-term version retention.

The analysis of its features reveals a clear design philosophy centered on robust, verifiable data protection. The expansion beyond simple SHA-1 to a full suite of modern cryptographic hashes, coupled with multi-layered verification commands like testbackup and versum, positions zpaqfranz as a formidable tool against both accidental data corruption and malicious threats. Its journaling, deduplicating architecture remains unparalleled for efficiently backing up large, incrementally changing data sets such as virtual machine disks and databases, offering a significant advantage over traditional backup methods.

Furthermore, the project's expansion into a broader system administration toolkit—with utilities for file synchronization, duplicate finding, and deep ZFS integration—transforms it from a mere archiver into a comprehensive solution for data management. While the distributed nature of its documentation has been a challenge for users, it is also a symptom of a project that prioritizes functional evolution and direct community interaction over static documentation.

For the system administrator, developer, or advanced user tasked with managing critical data, zpaqfranz presents a compelling, if complex, option. It demands a technical understanding from its user but, in return, offers a degree of control, efficiency, and verifiable integrity that is difficult to find in other tools, free or commercial. By consolidating its scattered documentation and elucidating its core principles, this guide aims to make its power more accessible, enabling users to confidently deploy it as a cornerstone of a modern, resilient data protection strategy.

#### **Works cited**

1. fcorbelli/zpaqfranz: Deduplicating archiver with encryption and paranoid-level tests. Swiss army knife for the serious backup and disaster recovery manager. Ransomware neutralizer. Win/Linux/Unix \- GitHub, accessed October 21, 2025, [https://github.com/fcorbelli/zpaqfranz](https://github.com/fcorbelli/zpaqfranz)  
2. zpaqfranz: Swiss army knife for backup and disaster recovery | Man ..., accessed October 21, 2025, [https://www.mankier.com/1/zpaqfranz](https://www.mankier.com/1/zpaqfranz)  
3. ZPAQ \- Wikipedia, accessed October 21, 2025, [https://en.wikipedia.org/wiki/ZPAQ](https://en.wikipedia.org/wiki/ZPAQ)  
4. ZPAQ \- Matt Mahoney, accessed October 21, 2025, [https://www.mattmahoney.net/zpaq/](https://www.mattmahoney.net/zpaq/)  
5. zpaqfranz \- Swiss army knife for backup and disaster recovery \- Ubuntu Manpage, accessed October 21, 2025, [https://manpages.ubuntu.com/manpages/plucky/man1/zpaqfranz.1.html](https://manpages.ubuntu.com/manpages/plucky/man1/zpaqfranz.1.html)  
6. zpaqfranz download | SourceForge.net, accessed October 21, 2025, [https://sourceforge.net/projects/zpaqfranz/](https://sourceforge.net/projects/zpaqfranz/)  
7. FreshPorts \-- archivers/zpaqfranz: Swiss army knife for the serious backup manager, accessed October 21, 2025, [https://www.freshports.org/archivers/zpaqfranz](https://www.freshports.org/archivers/zpaqfranz)  
8. zpaq: Journaling archiver for incremental backups. | Man Page | Commands \- ManKier, accessed October 21, 2025, [https://www.mankier.com/1/zpaq](https://www.mankier.com/1/zpaq)  
9. zpaqfranz \- Pelles C forum, accessed October 21, 2025, [https://forum.pellesc.de/index.php?topic=11094.0](https://forum.pellesc.de/index.php?topic=11094.0)  
10. Releases · fcorbelli/zpaqfranz \- GitHub, accessed October 21, 2025, [https://github.com/fcorbelli/zpaqfranz/releases](https://github.com/fcorbelli/zpaqfranz/releases)  
11. PKGBUILD review: zpaqfranz / Creating & Modifying Packages / Arch Linux Forums, accessed October 21, 2025, [https://bbs.archlinux.org/viewtopic.php?id=288792](https://bbs.archlinux.org/viewtopic.php?id=288792)  
12. New package proposal: zpaqfranz / AUR Issues, Discussion & PKGBUILD Requests / Arch Linux Forums, accessed October 21, 2025, [https://bbs.archlinux.org/viewtopic.php?id=275339](https://bbs.archlinux.org/viewtopic.php?id=275339)  
13. ZFS \- My experience in FreeBSD backup, physical and virtual, accessed October 21, 2025, [https://forums.freebsd.org/threads/my-experience-in-freebsd-backup-physical-and-virtual.79670/](https://forums.freebsd.org/threads/my-experience-in-freebsd-backup-physical-and-virtual.79670/)  
14. ZPAQ's complete code history mirror \- GitHub, accessed October 21, 2025, [https://github.com/zpaq/zpaq](https://github.com/zpaq/zpaq)  
15. zpaqfranz \- Browse /60.9 at SourceForge.net, accessed October 21, 2025, [https://sourceforge.net/projects/zpaqfranz/files/60.9/](https://sourceforge.net/projects/zpaqfranz/files/60.9/)  
16. Issue \#128 · fcorbelli/zpaqfranz \- Documentation clarity \- GitHub, accessed October 21, 2025, [https://github.com/fcorbelli/zpaqfranz/issues/128](https://github.com/fcorbelli/zpaqfranz/issues/128)  
17. zpaqfranz \- Browse /58.2 at SourceForge.net, accessed October 21, 2025, [https://sourceforge.net/projects/zpaqfranz/files/58.2/](https://sourceforge.net/projects/zpaqfranz/files/58.2/)  
18. \[WCX\] ZPAQ \- Page 14 \- Total Commander \- ghisler.ch, accessed October 21, 2025, [https://www.ghisler.ch/board/viewtopic.php?t=42739\&start=195](https://www.ghisler.ch/board/viewtopic.php?t=42739&start=195)  
19. zpaqfranz \- Browse /56.4 at SourceForge.net, accessed October 21, 2025, [https://sourceforge.net/projects/zpaqfranz/files/56.4/](https://sourceforge.net/projects/zpaqfranz/files/56.4/)  
20. Advanced backup mechanisms (for zfs) | TrueNAS Community, accessed October 21, 2025, [https://www.truenas.com/community/threads/advanced-backup-mechanisms-for-zfs.106478/](https://www.truenas.com/community/threads/advanced-backup-mechanisms-for-zfs.106478/)  
21. zpaqfranz \- Browse /57.3 at SourceForge.net, accessed October 21, 2025, [https://sourceforge.net/projects/zpaqfranz/files/57.3/](https://sourceforge.net/projects/zpaqfranz/files/57.3/)  
22. PeaZip / Tickets / \#743 Zpaq was discontinued a long time ago. Now support zpaqfranz, accessed October 21, 2025, [https://sourceforge.net/p/peazip/tickets/743/](https://sourceforge.net/p/peazip/tickets/743/)  
23. Issues · fcorbelli/zpaqfranz \- GitHub, accessed October 21, 2025, [https://github.com/fcorbelli/zpaqfranz/issues](https://github.com/fcorbelli/zpaqfranz/issues)  
24. fcorbelli/zsfx: zpaqfranz/zpaq SFX module for Windows 32 and 64 bit \- GitHub, accessed October 21, 2025, [https://github.com/fcorbelli/zsfx](https://github.com/fcorbelli/zsfx)