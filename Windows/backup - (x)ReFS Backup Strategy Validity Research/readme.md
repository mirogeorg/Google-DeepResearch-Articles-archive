
# **An Expert Analysis of a ReFS-Based Instant Access Backup Strategy**

## **Section 1: A Technical Deep Dive into the Strategy's Core Components**

A comprehensive validation of any data protection strategy requires a granular understanding of its constituent technologies. The proposed strategy leverages a modern filesystem (ReFS), a standard file copy utility (Robocopy), a snapshotting service (VSS), and a delta-transfer mechanism. This section deconstructs each of these components to establish a baseline of their capabilities, limitations, and operational characteristics before evaluating their integration within the proposed workflow.

### **1.1 The Foundation: ReFS and Block Cloning Efficiency**

The Resilient File System (ReFS) is the cornerstone of the proposed strategy, chosen specifically for its block cloning feature. This capability promises significant storage and performance benefits, but its mechanics and operational costs must be fully understood.

#### **Mechanism of Action: From Physical I/O to Metadata Operations**

Traditional file copy operations are resource-intensive, requiring the system to read data blocks from a source location and write them to a destination, consuming significant I/O bandwidth and time.1 ReFS block cloning fundamentally alters this process. Instead of a physical data transfer, a block clone is a metadata-only operation.1 The filesystem manipulates its internal pointers, altering the virtual cluster number (VCN) to logical cluster number (LCN) mappings to make the same physical data blocks on disk appear in a new file or location.1

This approach yields two primary benefits. First, the "copy" is nearly instantaneous because no actual data is read or written. Second, it is exceptionally space-efficient, as multiple files can now share the same physical data clusters, eliminating redundant data storage on the volume.3 This technology is the basis for features marketed by backup software vendors as "Fast Clone," which dramatically accelerates the creation of synthetic full backups.5

#### **The Copy-on-Write (CoW) Process and Data Integrity**

To prevent a change in one file from corrupting another file that shares its data blocks, ReFS employs a robust isolation mechanism known as allocate-on-write, or more commonly, copy-on-write (CoW).1 When an application initiates a write to a shared data block, ReFS intercepts the operation. It first allocates a new, empty block on the volume, copies the original data from the shared block into this new block, and only then applies the modification to the new block. The file's metadata pointers are then updated to reference this newly written block, while the original shared block remains untouched for other files to reference.1 This same CoW mechanism is used for all metadata updates, which is a core element of ReFS's overall resiliency against corruption.7

While this mechanism is highly effective for ensuring data integrity and enabling space savings, it introduces a performance consideration that is often overlooked. Every write operation to a shared block, regardless of its size, necessitates a read-copy-write cycle at the cluster level. Over time, particularly in a backup scenario with daily synthetic fulls and subsequent changes, this can lead to a form of filesystem fragmentation and I/O amplification. The perceived efficiency of the initial block clone is thus paid for with a potential performance penalty on subsequent write and delete operations.

#### **Critical Configuration: The Impact of 4KB vs. 64KB Cluster Sizes**

The efficiency and performance of ReFS, particularly its CoW behavior, are directly influenced by the cluster size chosen when the volume is formatted. This is not a trivial configuration choice, as it creates a trade-off between space efficiency and I/O performance.3

* **4KB Clusters:** This is the default and recommended size for most general-purpose workloads.2 It offers the highest space efficiency for CoW operations. For example, a 1-byte change to a shared region will only trigger the duplication of a 4KB cluster.3 This granularity is ideal for environments with many small, random writes.  
* **64KB Clusters:** This larger cluster size is recommended for large, sequential I/O workloads, such as archival and backup repositories.2 Backup software vendors like Veeam specifically recommend 64KB clusters for their ReFS repositories to improve performance with large backup files.5 The larger size reduces the metadata overhead and the total number of CoW operations required for large, contiguous writes. However, this performance gain comes at the cost of space efficiency; a 1-byte change to a shared block will force the duplication of an entire 64KB cluster.3

#### **Known Issues: Managing ReFS Performance and Stability at Scale**

While ReFS is designed for scalability with its B+ tree structure 7, real-world deployments have revealed operational challenges, particularly under the heavy, specialized load of backup operations.

* **Heavy Memory Usage:** A widely documented issue with ReFS is its potential for high memory consumption. The combination of its allocate-on-write semantics and a block caching logic that is less resource-efficient than traditional file caching can cause the active working set on a server to grow significantly, leading to memory pressure and poor performance.9 This issue was significant enough that Microsoft released updates with tunable registry parameters to help administrators more aggressively trim the ReFS memory cache.  
* **Performance with High File Counts and Deletions:** The process of deleting large numbers of block-cloned files is a metadata-intensive operation. ReFS must traverse its data structures to decrement the reference count for each shared block within the files being deleted. User forums and vendor knowledge bases contain numerous reports of this retention processing causing high system load and even OS stability issues on backup servers.10 This directly challenges the assumption that retention management in the proposed strategy would be simple.

### **1.2 The Keystone of Consistency: Volume Shadow Copy Service (VSS)**

Simply copying files from a running system, especially one hosting active applications like databases or email servers, is a recipe for data corruption. Files may be locked and inaccessible, or they may be captured in a transitional, inconsistent state. The Volume Shadow Copy Service (VSS) is the standard Windows framework designed to solve this problem.11

VSS acts as a coordinator between three key components: the backup application (the *Requester*), the applications whose data is being backed up (the *Writers*, e.g., SQL Server, Exchange), and the storage system (the *Provider*).11 To create a point-in-time, application-consistent snapshot, VSS orchestrates a precise sequence of events:

1. The Requester asks VSS to begin the snapshot process.  
2. VSS notifies all registered Writers to prepare their data. The Writers then flush their in-memory caches to disk, complete open transactions, and bring their data to a consistent state.11  
3. Once the Writers confirm they are ready, VSS instructs them to temporarily freeze all new write I/O operations. This "freeze" window is very short, typically lasting only a few seconds and not permitted to exceed 60 seconds.11  
4. During this freeze, VSS tells the Provider to create the shadow copy (snapshot) of the volume.  
5. Immediately after the snapshot is created, VSS instructs the Writers to "thaw" and resume normal I/O operations.11

The result is a block-level, read-only, point-in-time copy of the volume that is guaranteed to be in a consistent state, from which a reliable backup can be made without extended application downtime.12 The strategy's use of 15 daily snapshots correctly leverages this powerful technology to create reliable historical versions.

### **1.3 The Toolchain: Data Movers and Their Capabilities**

The strategy relies on two distinct types of data movers: Robocopy for server-side duplication and an rsync-like utility for client-side data collection. The specific capabilities and limitations of these tools are critical to the strategy's validity.

#### **Robocopy: Assessing Native Support for ReFS Block Cloning**

A central assumption of the proposed strategy is that using Robocopy to copy the contents of the STORE folder into a new daily folder will trigger ReFS block cloning, thus achieving the desired storage efficiency. However, analysis of official Microsoft documentation reveals this premise to be false for the target server platforms.

The documentation for Robocopy on Windows Server 2016, 2019, and 2022 contains no switch or mention of support for ReFS block cloning.14 Furthermore, Microsoft's primary documentation on block cloning explicitly states that native support for this feature in standard Windows copy operations begins with Windows 11 24H2 and Windows Server 2025\.1 This strongly implies that prior operating systems require applications to use specific APIs to invoke block cloning, which standard command-line tools like Robocopy do not do.16

This single technical point threatens to invalidate the entire strategy. If Robocopy performs a traditional file copy, the daily duplication process will not be a near-instant, space-free metadata operation. Instead, it will be a slow, I/O-intensive full data copy that consumes an amount of disk space equal to the source data each day, completely negating the primary storage efficiency benefit.

#### **The Open File Problem: Robocopy's Inability to Guarantee Consistency**

A second, equally critical flaw in using Robocopy for this purpose is its lack of integration with VSS.17 Robocopy operates at the file level and has no mechanism to coordinate with application writers to ensure data consistency. When it encounters a file that is locked open by an application, it will either fail to copy the file or, if run in backup mode (/B or /ZB), it may copy the file's contents as they exist on disk at that moment, which could be an inconsistent, corrupted state.18 While complex workarounds exist that involve using a separate tool like VShadow.exe to first create a VSS snapshot and then running Robocopy against the mounted snapshot, this is not part of the described strategy and adds significant complexity.19

#### **Client-Side Delta Transfer: Evaluating Rsync-like Utilities for Windows**

The strategy specifies that client systems will use a utility that transfers only the changed portions within files, which accurately describes the rsync algorithm. While rsync is native to Linux, several implementations exist for Windows, each with important distinctions.

| Utility Name | Underlying Technology | Windows ACL Support | Security/Encryption | Key Advantages | Key Disadvantages |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **DeltaCopy** | Recompiled Linux Rsync binaries using Cygwin libraries 20 | **Poor**. Incompatibility between Cygwin's POSIX permissions model and Windows ACLs often leads to permission errors on the server.20 | Data transfer can be encrypted via SSH, but files are not encrypted at rest on the server.20 | Free and open-source; compatible with standard Rsync daemons.22 | Cygwin dependency causes significant permission issues; known problems with Windows junctions.20 |
| **Syncrify** | Custom implementation of the Rsync algorithm; no Cygwin dependency 20 | **Excellent**. Natively handles Windows ACLs without compatibility problems.20 | Data transfer encrypted via HTTPS; supports optional AES encryption for files at rest on the server.20 | High compatibility with Windows environments; offers built-in versioning and bi-directional sync.20 | Commercial product; requires Syncrify server, not compatible with standard Rsync daemons.20 |

The choice of utility is not trivial. Using a Cygwin-based tool like DeltaCopy introduces a high risk of file permission problems, a common frustration in mixed-platform environments. A tool with a native Windows implementation, while potentially incurring a cost, is far more likely to preserve critical security information correctly.

## **Section 2: Functional and Operational Validation of the Proposed Workflow**

With a technical understanding of the components established, the analysis now shifts to the strategy's proposed workflow. This section evaluates the daily processes to determine if they can collectively achieve the stated objectives of storage efficiency, fast data access, and simple retention management.

### **2.1 Analyzing the Daily Backup Process**

The core of the daily operation involves two distinct actions: maintaining VSS snapshots and creating file-level copies with Robocopy. The interaction between these two processes reveals a significant misunderstanding of their respective roles in a backup architecture.

#### **Interaction Between VSS Snapshots and Robocopy Archives: Redundancy or Misunderstanding?**

The strategy calls for keeping 15 days of VSS snapshots *and* creating 15 daily folders via Robocopy on the same disk. This approach is functionally redundant and inefficient.

* VSS snapshots are already block-level, space-efficient, point-in-time copies that are application-consistent.11 They effectively provide the 15-day version history the strategy seeks.  
* The Robocopy folders, as established, are file-level copies that are *not* application-consistent and, critically, are likely *not* space-efficient due to Robocopy's lack of block cloning support in the target OS versions.

This design essentially uses two different mechanisms to achieve the same goal of versioning, storing both sets of versions on the same target. A robust strategy would typically use a single, reliable mechanism (like a VSS-aware backup application) to create one consistent backup set, and then focus on replicating that set to different targets for resilience. The current approach complicates the backup process and consumes additional resources without providing any additional data protection. It confuses the backup *mechanism* with the backup *target*, leading to an inefficient and convoluted workflow.

#### **Confirming the Efficacy of Block Cloning for Daily Duplication**

If the flawed assumption about Robocopy were corrected—for instance, by using a custom PowerShell script that properly invokes the block cloning APIs or a commercial backup tool—the underlying principle of creating space-efficient daily copies is sound. Commercial backup solutions like Veeam heavily leverage this ReFS capability, which they term "Fast Clone," to create synthetic full backups.5 In this model, a new synthetic full backup file is created by referencing the unchanged data blocks from previous backup files on the same ReFS volume, only writing new blocks for changed data.6 This is directly analogous to the user's goal and confirms that, with the right tool, the storage efficiency objective is technologically achievable.

#### **Retention Management: The Practicality of Deleting Daily Folders**

The strategy's claim of "Simple Retention Management" by deleting the oldest daily folder is an oversimplification. In a block-cloned environment, deleting a folder containing potentially millions of shared blocks is a complex metadata operation. The filesystem cannot simply mark the space as free; it must meticulously traverse the metadata for every file, identify each referenced block, and decrement its reference count.1 If a block's reference count drops to zero, it can then be freed.

As observed in production environments, particularly those with large-scale backup repositories, this process of "garbage collection" or retention processing can be extremely resource-intensive, leading to high CPU, RAM, and I/O load, and in some cases, system instability.10 The daily deletion of a large, block-cloned folder is therefore not a trivial task and may introduce significant performance overhead on the backup server, contradicting the goal of simplicity.

### **2.2 Assessing the "Instant Access" Objective**

One of the primary goals of the strategy is to provide fast access to backed-up data without needing a formal restoration process. This objective is largely met, but its scope is limited.

#### **Verification of Direct, Extraction-Free Data Access**

Because the daily backups are stored as regular files and folders in their native format, they are immediately accessible through a standard network share. A user needing to recover a single file can simply browse to the appropriate daily folder (e.g., \\\\BACKUPSERVER\\STORE\\2023-10-27\\) and copy the file back to its original location. This stands in stark contrast to many traditional backup solutions that store data in proprietary, compressed container files, which require a dedicated backup client and a potentially lengthy extraction and restoration process.

#### **Implications for Recovery Time Objectives (RTO)**

This instant access model provides an exceptionally low Recovery Time Objective (RTO) for the most common recovery scenario: single-file or folder restoration. The time to recovery is limited only by the speed of browsing a network share and copying the file.

However, the strategy's effectiveness is confined to this specific scenario. It provides no defined process or capability for a full system recovery. In the event of a server failure, there is no mechanism to restore the operating system, applications, or the entire data volume in a coordinated manner. The strategy is optimized for granular recovery at the expense of any disaster recovery capability.

## **Section 3: Strategic Risk Assessment: Identifying Critical Architectural Flaws**

While the technical and operational analysis reveals inefficiencies and flawed assumptions, a strategic assessment against fundamental data protection principles exposes the architecture's most profound and dangerous weaknesses. This section evaluates the strategy's resilience and its ability to protect data against realistic threats.

### **3.1 The Achilles' Heel: The Single Point of Failure (SPOF)**

The most severe flaw in the proposed strategy is its complete disregard for the concept of a Single Point of Failure (SPOF).

#### **Defining SPOF in the Context of Data Backup**

A SPOF is any component, system, or process whose failure will cause the entire system to cease functioning.24 In a data protection context, a SPOF is any single event that can result in the simultaneous loss of both the primary production data and all backup copies.26 The primary goal of any valid backup strategy is to eliminate such single points of failure through redundancy and isolation.28

#### **How Co-locating Backups on the Source System Violates Core Resiliency Principles**

The proposed strategy places the primary data (STORE share), the VSS snapshots of that data, and the 15 days of daily file-level archives all on the **same server**, and on the **same physical disk volume**. This architecture is the quintessential example of a SPOF. There is no physical, logical, or geographical separation between the production data and its backups.

#### **Scenario Analysis: The Impact of Critical Failure Events**

Evaluating this architecture against common disaster scenarios demonstrates its fragility and the unacceptable level of risk it creates:

* **Hardware Failure:** A failure of the single disk volume, the storage controller, the server motherboard, or the power supply will simultaneously destroy the live data, all 15 VSS snapshots, and all 15 daily archive folders. Recovery will be impossible.27  
* **Ransomware Attack:** Modern, sophisticated ransomware does not just encrypt user files. It actively seeks out and deletes VSS snapshots to prevent easy recovery.32 An attack on this server would encrypt the live STORE share, execute commands to delete all shadow copies, and then proceed to encrypt every file in the 15 daily archive folders. The result would be total, irrecoverable data loss.34  
* **Natural Disaster or Theft:** A fire, flood, or other localized disaster that destroys the server room, or the physical theft of the server itself, will result in the permanent loss of all data and all backup copies in a single incident.31  
* **Catastrophic Human Error:** A single mistaken command by a system administrator, such as accidentally reformatting the ReFS volume or deleting the root STORE folder, would instantly wipe out the production data and all historical versions.31

The strategy's design, which prioritizes the convenience of instant access, has directly created an architecture that is completely defenseless against any significant failure event. It is not a backup strategy; it is a versioned archive that shares the exact same fate as the primary data it is meant to protect.

### **3.2 Benchmarking Against the Industry Standard: The 3-2-1 Backup Rule**

The 3-2-1 rule is a foundational, industry-standard best practice for data protection that has been proven effective for decades.32 It provides a simple, memorable framework for designing a resilient backup architecture. The rule dictates that you should:

* Maintain at least **THREE** total copies of your data (your primary data plus two backups).  
* Store these copies on at least **TWO** different types of storage media or devices.  
* Keep at least **ONE** of these copies in an off-site location.

The purpose of this rule is to ensure data survivability by introducing redundancy, media diversity, and geographic separation, thereby eliminating single points of failure.33

In response to modern threats like ransomware, this rule has evolved into the **3-2-1-1-0 rule**, which adds two crucial layers: one copy must be **immutable** (cannot be altered or deleted) or **air-gapped** (physically disconnected from the network), and the recovery process must be tested to ensure **zero errors**.32

The following scorecard evaluates the proposed strategy against this critical industry benchmark.

| 3-2-1 Rule Requirement | Proposed Strategy's Status | Explanation of Failure |
| :---- | :---- | :---- |
| **3 Copies of Data** | **FAIL** | The strategy creates multiple *versions* (snapshots and daily folders) but they are all part of a single logical *copy* co-located with the primary data. A single failure event destroys all versions simultaneously. Therefore, there is effectively only one copy. |
| **2 Different Media** | **FAIL** | All data—primary, snapshots, and archives—resides on the same disk volume, which represents a single media type and a single device. |
| **1 Off-site Copy** | **FAIL** | No component of the strategy involves storing data at a separate geographical location. All data is vulnerable to a single site-wide disaster. |
| **1 Immutable/Air-Gapped Copy** | **FAIL** | All backup versions are online, writable, and connected to the network, making them fully vulnerable to deletion or encryption by ransomware or malicious actors. |

The strategy fails to meet even a single tenet of the most basic data protection framework. This is the most definitive indicator of its invalidity as a backup solution.

## **Section 4: Final Verdict and Recommendations for a Resilient Architecture**

This final section synthesizes the preceding analysis to deliver a conclusive verdict on the proposed strategy's validity and provides a clear, actionable blueprint for re-architecting the solution to meet modern standards of data protection.

### **4.1 Overall Validity Assessment**

#### **Verdict on Technical Mechanisms: Powerful but Misapplied**

The individual technologies selected for the strategy are, in principle, sound and powerful. ReFS with block cloning is an excellent choice for a storage-efficient backup repository. The goal of providing instant, extraction-free access to file-level data is a valid objective that can significantly lower RTO for common recovery tasks.

However, these technologies are misapplied. The reliance on Robocopy is technically flawed, as it is unlikely to trigger block cloning on the intended server platforms and cannot create application-consistent backups. The parallel use of VSS snapshots and file-level copies on the same volume is functionally redundant and operationally inefficient.

#### **Verdict on Architectural Strategy: Fundamentally Invalid and High-Risk**

From an architectural standpoint, the strategy is critically flawed and must be considered invalid. Its design creates a massive Single Point of Failure, placing primary data and all backup copies on the same physical system. It completely fails to adhere to the industry-standard 3-2-1 rule for data protection.

Consequently, the strategy provides a false sense of security. While it offers versioning capabilities, it offers zero protection against the most common and severe data loss scenarios, including hardware failure, ransomware, natural disasters, and significant human error. **Therefore, the strategy as proposed is not a valid backup strategy and should not be implemented.**

### **4.2 A Blueprint for a Robust Backup Strategy**

To transform this high-risk architecture into a resilient data protection solution, a fundamental redesign is required, focusing on separation, redundancy, and the use of appropriate, integrated tools.

#### **Recommendation 1: Immediate Mitigation \- Physical and Logical Separation**

The most urgent and critical action is to eliminate the primary SPOF. This requires establishing a **separate, dedicated backup server**. The backup repository, which will host the ReFS volume, must reside on this second machine, which is physically and logically distinct from the primary server hosting the live data. All backup operations should be initiated to write data from the primary server to this new, dedicated backup server. This single change immediately mitigates the risk of a single server hardware failure destroying all data.

#### **Recommendation 2: A Phased Approach to Implementing the 3-2-1 Rule**

With a dedicated backup server in place, the organization can systematically implement the 3-2-1 rule.

* **Phase 1 (Achieving 3 Copies, 2 Media):**  
  * **Copy 1:** The primary production data on the main server.  
  * **Copy 2:** The primary backup, stored on the dedicated backup server's ReFS volume.  
  * **Copy 3 / Media 2:** Implement a secondary local backup. This can be achieved by replicating the backups from the dedicated backup server to another storage device, such as a separate Network Attached Storage (NAS) appliance or even a set of rotated external hard drives. This introduces media diversity and protects against the failure of the primary backup server.  
* **Phase 2 (Achieving 3-2-1):**  
  * **Copy 1 (Off-site):** Implement an off-site backup copy. The most scalable and common method is to replicate backups from the dedicated backup server to a cloud storage provider (e.g., Amazon S3, Azure Blob Storage). Alternatively, if a second physical site is available, backups can be replicated there. This protects against a site-wide disaster.

#### **Recommendation 3: Unifying the Toolchain with an Integrated Backup Solution**

The disjointed and flawed toolchain of VSS scripts, Robocopy, and a separate delta-copy utility should be replaced with a modern, integrated, VSS-aware backup application (e.g., Veeam Backup & Replication, Acronis Cyber Protect, etc.). Such solutions are purpose-built to:

* Properly integrate with VSS to create fully application-consistent backups of live systems.  
* Intelligently leverage ReFS block cloning on the backup repository to create space-efficient synthetic full backups, achieving the strategy's original efficiency goal.5  
* Automate the entire backup lifecycle, including scheduling, retention, and the replication of backups to secondary and off-site targets to fulfill the 3-2-1 rule.  
* Provide centralized monitoring, alerting, and robust features for recovery verification and disaster recovery testing.

#### **Recommendation 4: Configuration Best Practices for the ReFS Backup Repository**

When configuring the new dedicated backup server, the following best practices for ReFS should be observed:

* **Use 64KB Cluster Size:** Format the ReFS backup volume with 64KB clusters to optimize performance for large, sequential backup file I/O, as recommended by leading backup software vendors.5  
* **Monitor System Resources:** ReFS can be more resource-intensive than NTFS.8 Ensure the backup server is provisioned with sufficient RAM and CPU resources. Actively monitor memory usage and be prepared to apply the registry tuning parameters if the ReFS cache grows excessively.9  
* **Disable 8.3 Short Name Generation:** On volumes that will contain a very large number of files, disabling the creation of legacy 8.3 DOS-compatible filenames can improve directory enumeration performance.39

## **Section 5: Conclusion: From Efficient Technology to Resilient Protection**

The initial strategy correctly identified a powerful and efficient technology in ReFS block cloning as a means to achieve storage-efficient versioning with instant data access. The core technological premise is sound and is leveraged by leading enterprise backup solutions for exactly these benefits. However, the strategy's ultimate failure lies in its myopic focus on this single technological capability at the expense of fundamental architectural principles of data protection.

A valid backup strategy is defined not by its convenience or storage footprint, but by its resilience in the face of disaster. By co-locating all data and backup versions on a single system, the proposed architecture creates a fragile structure that is guaranteed to fail completely when confronted with any number of common, foreseeable incidents. It is an archiving system, not a backup system.

The path forward requires a paradigm shift: from leveraging a clever technology on a single server to building a robust, distributed architecture. By embracing the time-tested 3-2-1 rule, physically and logically separating backup data from production systems, and deploying an integrated, application-aware backup solution, the organization can transform this high-risk proposal into a genuine, enterprise-grade data protection strategy that provides both the desired efficiency and the essential resilience required to ensure business continuity.

#### **Works cited**

1. Block cloning on ReFS | Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/refs/block-cloning](https://learn.microsoft.com/en-us/windows-server/storage/refs/block-cloning)  
2. Resilient File System (ReFS) overview \- Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/refs/refs-overview](https://learn.microsoft.com/en-us/windows-server/storage/refs/refs-overview)  
3. Block Cloning \- Win32 apps \- Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows/win32/fileio/block-cloning](https://learn.microsoft.com/en-us/windows/win32/fileio/block-cloning)  
4. Analyse ReFS space savings with Block cloning | Veeam Community Resource Hub, accessed October 17, 2025, [https://community.veeam.com/blogs-and-podcasts-57/analyse-refs-space-savings-with-block-cloning-96](https://community.veeam.com/blogs-and-podcasts-57/analyse-refs-space-savings-with-block-cloning-96)  
5. Fast Clone \- User Guide for VMware vSphere \- Veeam Help Center, accessed October 17, 2025, [https://helpcenter.veeam.com/docs/backup/vsphere/backup\_repository\_block\_cloning.html](https://helpcenter.veeam.com/docs/backup/vsphere/backup_repository_block_cloning.html)  
6. Using Veeam and ReFS to Save Time and Space \- ivision, accessed October 17, 2025, [https://ivision.com/blog/using-veeam-and-refs-to-save-time-and-space/](https://ivision.com/blog/using-veeam-and-refs-to-save-time-and-space/)  
7. ReFS Features \- NTFS.com, accessed October 17, 2025, [https://www.ntfs.com/refs-features.htm](https://www.ntfs.com/refs-features.htm)  
8. What is ReFS File System: Benefits and Best Practices \- NAKIVO, accessed October 17, 2025, [https://www.nakivo.com/blog/windows-server-refs-file-system-benefits/](https://www.nakivo.com/blog/windows-server-refs-file-system-benefits/)  
9. Fix heavy memory usage in ReFS \- Windows Server | Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/fix-heavy-memory-usage-refs](https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/fix-heavy-memory-usage-refs)  
10. REFS issues (server lockups, high CPU, high RAM) \- Page 40 \- Veeam R\&D Forums, accessed October 17, 2025, [https://forums.veeam.com/veeam-backup-replication-f2/refs-4k-horror-story-t40629-1170.html](https://forums.veeam.com/veeam-backup-replication-f2/refs-4k-horror-story-t40629-1170.html)  
11. Volume Shadow Copy Service (VSS) | Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service](https://learn.microsoft.com/en-us/windows-server/storage/file-server/volume-shadow-copy-service)  
12. What is VSS Backup and What are its Advantages? \- Sangfor Technologies, accessed October 17, 2025, [https://www.sangfor.com/blog/cloud-and-infrastructure/what-is-vss-backup](https://www.sangfor.com/blog/cloud-and-infrastructure/what-is-vss-backup)  
13. Shadow Copy \- Wikipedia, accessed October 17, 2025, [https://en.wikipedia.org/wiki/Shadow\_Copy](https://en.wikipedia.org/wiki/Shadow_Copy)  
14. Robocopy | Microsoft Learn, accessed October 17, 2025, [https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy)  
15. A Complete Guide to Robocopy | Petri IT Knowledgebase, accessed October 17, 2025, [https://petri.com/robocopy-complete-guide/](https://petri.com/robocopy-complete-guide/)  
16. ReFS Block Cloning \- Use command line to copy a file and use block cloning \- Super User, accessed October 17, 2025, [https://superuser.com/questions/1762696/refs-block-cloning-use-command-line-to-copy-a-file-and-use-block-cloning](https://superuser.com/questions/1762696/refs-block-cloning-use-command-line-to-copy-a-file-and-use-block-cloning)  
17. How to use robocopy for taking regular backups on Windows \- Reddit, accessed October 17, 2025, [https://www.reddit.com/r/windows/comments/n900sy/how\_to\_use\_robocopy\_for\_taking\_regular\_backups\_on/](https://www.reddit.com/r/windows/comments/n900sy/how_to_use_robocopy_for_taking_regular_backups_on/)  
18. Robocopy (win) | Ensuring Integrity | Source/Destination "size" NOT identical (whole drives), accessed October 17, 2025, [https://superuser.com/questions/1147044/robocopy-win-ensuring-integrity-source-destination-size-not-identical-w](https://superuser.com/questions/1147044/robocopy-win-ensuring-integrity-source-destination-size-not-identical-w)  
19. Matthew J. Miller's HOWTOs: Backing Locked Windows Files Using ..., accessed October 17, 2025, [https://www.matthewjmiller.net/howtos/backing-up-locked-windows-files/](https://www.matthewjmiller.net/howtos/backing-up-locked-windows-files/)  
20. Difference between Syncrify and DeltaCopy, accessed October 17, 2025, [https://web.synametrics.com/syncrifyvsdeltacopy.htm](https://web.synametrics.com/syncrifyvsdeltacopy.htm)  
21. Rsync like windows backup tool \[closed\] \- Server Fault, accessed October 17, 2025, [https://serverfault.com/questions/220546/rsync-like-windows-backup-tool](https://serverfault.com/questions/220546/rsync-like-windows-backup-tool)  
22. Best way to backup Windows SMB shares to a remote server : r/homelab \- Reddit, accessed October 17, 2025, [https://www.reddit.com/r/homelab/comments/1m2xqye/best\_way\_to\_backup\_windows\_smb\_shares\_to\_a\_remote/](https://www.reddit.com/r/homelab/comments/1m2xqye/best_way_to_backup_windows_smb_shares_to_a_remote/)  
23. Veeam Fast Clone Space Savings Script \- benharmer.blog, accessed October 17, 2025, [https://benharmer.blog/2022/11/08/veeam-fast-clone-space-savings-script/](https://benharmer.blog/2022/11/08/veeam-fast-clone-space-savings-script/)  
24. Single point of failure \- Wikipedia, accessed October 17, 2025, [https://en.wikipedia.org/wiki/Single\_point\_of\_failure](https://en.wikipedia.org/wiki/Single_point_of_failure)  
25. Single Point of Failure (SPOF): How to Identify and Eliminate It? \- ClouDNS Blog, accessed October 17, 2025, [https://www.cloudns.net/blog/single-point-of-failure-spof-how-to-identify-and-eliminate-it/](https://www.cloudns.net/blog/single-point-of-failure-spof-how-to-identify-and-eliminate-it/)  
26. Single Points of Failure and Backup & Disaster Recovery \- Consolidated Communications, accessed October 17, 2025, [https://www.consolidated.com/Blog/ArtMID/3914/ArticleID/226/Single-Points-of-Failure-and-Backup-Disaster-Recovery](https://www.consolidated.com/Blog/ArtMID/3914/ArticleID/226/Single-Points-of-Failure-and-Backup-Disaster-Recovery)  
27. What is a Single Point of Failure (SPOF)? \- Anomali, accessed October 17, 2025, [https://www.anomali.com/blog/why-single-point-of-failure-is-scary](https://www.anomali.com/blog/why-single-point-of-failure-is-scary)  
28. Understanding Single Point Failures: A Guide to System Resilience \- Bryghtpath, accessed October 17, 2025, [https://bryghtpath.com/single-point-failures/](https://bryghtpath.com/single-point-failures/)  
29. How to Avoid a Single Point of Failure: Key Mitigation Techniques \- TierPoint, accessed October 17, 2025, [https://www.tierpoint.com/blog/single-point-of-failure/](https://www.tierpoint.com/blog/single-point-of-failure/)  
30. How bad is to store my data in a single drive nas?, : r/homelab \- Reddit, accessed October 17, 2025, [https://www.reddit.com/r/homelab/comments/z9bodg/how\_bad\_is\_to\_store\_my\_data\_in\_a\_single\_drive\_nas/](https://www.reddit.com/r/homelab/comments/z9bodg/how_bad_is_to_store_my_data_in_a_single_drive_nas/)  
31. The Dangers of Relying on Physical Servers for Data Backup: Why You Should Switch to Server Cloud Backup \- \- IBT Learning, accessed October 17, 2025, [https://www.ibtlearning.co/the-dangers-of-relying-on-physical-servers-for-data-backup-why-you-should-switch-to-server-cloud-backup/](https://www.ibtlearning.co/the-dangers-of-relying-on-physical-servers-for-data-backup-why-you-should-switch-to-server-cloud-backup/)  
32. The 3-2-1 Backup Rule: A Proven Strategy for Data Protection | Cohesity, accessed October 17, 2025, [https://www.cohesity.com/glossary/321-backup-rule/](https://www.cohesity.com/glossary/321-backup-rule/)  
33. What is the 3-2-1 Backup Strategy? \- 2025 Guide by Acronis, accessed October 17, 2025, [https://www.acronis.com/en/blog/posts/backup-rule/](https://www.acronis.com/en/blog/posts/backup-rule/)  
34. Exploring the Pros and Cons of Common NAS Backup Strategies \- Hackernoon, accessed October 17, 2025, [https://hackernoon.com/exploring-the-pros-and-cons-of-common-nas-backup-strategies](https://hackernoon.com/exploring-the-pros-and-cons-of-common-nas-backup-strategies)  
35. 3-2-1 Backup Rule Explained: Do I Need One? \- Veeam, accessed October 17, 2025, [https://www.veeam.com/blog/321-backup-rule.html](https://www.veeam.com/blog/321-backup-rule.html)  
36. What is the 3-2-1 Backup Rule? | Barracuda Networks, accessed October 17, 2025, [https://www.barracuda.com/support/glossary/3-2-1-backup-rule](https://www.barracuda.com/support/glossary/3-2-1-backup-rule)  
37. Backup Strategies: Why the 3-2-1 Backup Strategy is the Best \- Backblaze, accessed October 17, 2025, [https://www.backblaze.com/blog/the-3-2-1-backup-strategy/](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)  
38. 3-2-1 Backup Rule: A Guide to Efficient Data Protection \- NAKIVO, accessed October 17, 2025, [https://www.nakivo.com/blog/3-2-1-backup-rule-efficient-data-protection-strategy/](https://www.nakivo.com/blog/3-2-1-backup-rule-efficient-data-protection-strategy/)  
39. Can file system performance decrease if there is a very large number of files in a single directory (NTFS)? \- Super User, accessed October 17, 2025, [https://superuser.com/questions/623965/can-file-system-performance-decrease-if-there-is-a-very-large-number-of-files-in](https://superuser.com/questions/623965/can-file-system-performance-decrease-if-there-is-a-very-large-number-of-files-in)  
40. NTFS performance and large volumes of files and directories \- Stack Overflow, accessed October 17, 2025, [https://stackoverflow.com/questions/197162/ntfs-performance-and-large-volumes-of-files-and-directories](https://stackoverflow.com/questions/197162/ntfs-performance-and-large-volumes-of-files-and-directories)